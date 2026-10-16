# SRE Runbook — IGPS Backfill Migration

Step-by-step execution guide for running this migration independently against **any** customer
environment (not just Ecolab UAT). Read this fully before touching `env_constants.py`. For what the
script does and why, see [`README.md`](README.md); for the generic RDP API plumbing, see
[`project_setup_readme.md`](project_setup_readme.md).

This migration **writes data** in its final phase. Phases 0–3 are read-only and safe to re-run as many
times as you like; Phase 4 is not. Treat the gate between them as a hard stop, not a formality.

---

## 0. Before you start — information you need from the request/ticket

Do not start configuring until you have all of these, confirmed by the requester:

| Item | Where it comes from |
|---|---|
| Target pod / environment name | Ticket or requester (e.g. `rdpprna26`) |
| Target tenant ID | Ticket or requester (e.g. `ecolabuat`, or the customer's prod tenant id) |
| Lookback window (`createdDate` cutoff) | Requester — confirm the exact ISO-8601 cutoff and timezone offset they want |
| Confirmation this is the right fix for the right entity type | Requester — the script is hard-coded to `baseCustomerAccount`; if a different entity type is affected, this script needs code changes, not just config changes |
| Whether you're running from a dev machine or from inside the cluster | Determines `use_proxy` |

If any of these is missing or ambiguous, stop and ask — do not guess a cutoff date or tenant name.

---

## 1. One-time machine setup

```bash
git clone <this-repo-url>
cd ecolab-igps-migration
python3 -m venv .venv
source .venv/bin/activate
pip install requests
```

---

## 2. Obtain a bearer token for the target tenant

All API calls use an Azure AD bearer token passed as `Authorization: Bearer <token>`. Generate it through
your normal RDP/PIM authentication process for the target tenant (the same mechanism you use for other
RDP operational scripts — if you don't already have one, ask the platform team rather than reusing a
token you find lying around).

Keep two things in mind:

- **It expires in ~8 hours.** If a run takes longer than that, you'll get HTTP 401s partway through —
  regenerate the token and resume (phases are resumable, see §5).
- **It is a secret tied to a real person's identity** (the token embeds the issuing user's email/OID).
  Never commit it, paste it into a ticket, or leave it in a file that gets checked in. See §8.

---

## 3. Configure `env_constants.py` for this run

Open `env_constants.py` and set, **for this specific customer/engagement**:

| Variable | Set to |
|---|---|
| `ENV_NAME` | The target pod name from §0 |
| `TENANT_NAME` | The target tenant id from §0 |
| `use_proxy` | `True` if running from a dev machine (routes via `pimapiproxy.syndigo.com`); `False` if running inside the cluster network |
| `BEARER_TOKEN` | The token from §2 |
| `CREATED_DATE_CUTOFF` | The exact ISO-8601 cutoff from §0, e.g. `"2026-02-17T00:00:00.000-0500"` |
| `entity_batch_size` | Leave at `2000` unless told otherwise (platform max) |
| `MAX_WORKERS` | Leave at `10` unless the platform team asks you to throttle (lower this if you see 429/503 errors) |

Double-check `ENTITY_TYPE` further down in the file still reads `"baseCustomerAccount"` — if the
requester told you a different entity type is affected, stop; this script is not written to handle that
and needs a code change, not a config change.

Run `git diff env_constants.py` before going further and confirm you recognize every line that changed.

---

## 4. Run the read-only phases (0–3)

These only read data — nothing is written to the customer's entities yet.

```bash
python main.py
```

This runs phases 0 → 3 in sequence (Phase 4 is commented out in `main.py` by default — leave it that
way for now). Watch the console output and/or tail the per-phase logs:

```
data/phase0/phase0.log
data/phase1/phase1.log
data/phase2/phase2.log
data/phase3/phase3.log
```

Expect this to take anywhere from minutes to a couple of hours depending on tenant size — Phase 0/1/2 are
scroll/batch API calls, Phase 3 does one history lookup per entity (parallelized across `MAX_WORKERS`
threads).

**If a phase fails partway through (e.g. token expired):** regenerate the token, update
`BEARER_TOKEN`, and re-run **only that phase's script** — you don't need to restart from Phase 0:

```bash
python phase0_get_event_ids.py
python phase1_get_entity_ids.py
python phase2_filter_by_created_date.py
python phase3_find_igps.py
```

Each phase reads its input from the previous phase's output file (see table in `README.md`), so
re-running a later phase alone is safe as long as the earlier output files already exist.

---

## 5. Validation gate — mandatory before Phase 4

**Do not proceed to Phase 4 without doing this.** Phase 4 writes to real customer entities with no
dry-run mode and no automated rollback.

1. Open `data/phase3/igps_map.json`. Confirm:
   - The number of entries is a plausible fraction of the entity count seen in Phase 1/2 logs (not
     suspiciously 0, and not suspiciously everything).
   - Spot-check 3–5 `entity_id → value` pairs against the platform UI or a manual `getentityhistory`
     call for those entities — the value shown should match the entity's actual earliest
     `grmprocessstate`.
2. Open `data/phase3/warnings.json`. These are entities that passed all filters but had no
   `grmprocessstate` anywhere in their history — they will **not** be touched by Phase 4. Confirm this
   count is small and sanity-check a couple of them; a large warnings list usually means something
   upstream (Phase 0/1 filters, or the cutoff date) is off, not that the data is simply missing.
3. Get explicit sign-off (from the requester or whoever owns this engagement) on the reviewed
   `igps_map.json` before enabling Phase 4. Record that sign-off in the ticket.

If anything here looks wrong, stop and investigate — don't proceed on the assumption it'll sort itself
out in Phase 4.

---

## 6. Run Phase 4 — the write step

Only after §5 sign-off:

1. In `main.py`, uncomment the Phase 4 line:
   ```python
   ("Phase 4 — Update initialgrmprocessstate", phase4_update_igps.run),
   ```
2. Refresh the bearer token in `env_constants.py` if more than ~8 hours have passed since §2.
3. Run **Phase 4 directly** (not `main.py`), so phases 0–3 don't re-run unnecessarily:
   ```bash
   python phase4_update_igps.py
   ```
4. Watch `data/phase4/phase4.log` for progress (logged every 100 entities).
5. When it finishes, inspect `data/phase4/results.json`:
   ```json
   { "succeeded": [...], "failed": [...] }
   ```
   - `failed` entries include the reason (`non-success response` with the raw API response, or an
     exception message). Triage these individually — a small number of failures can usually be
     re-run by re-invoking `phase4_update_igps.run()` with a trimmed `igps_map.json` containing just
     the failed IDs; do not blanket-rerun against the full map unless you're sure re-applying the
     same value to already-succeeded entities is harmless (it should be, since this sets a fixed
     value, but confirm before assuming).
6. Comment the Phase 4 line back out in `main.py` once the engagement is done, so a future run of
   `main.py` doesn't accidentally re-trigger writes.

---

## 7. After the run

1. **Archive the `data/` directory** for this engagement (copy it out, e.g. to the ticket or a dated
   folder) — the next run of Phase 0 on a different customer will overwrite these same files in place.
2. **Scrub the token.** Replace `BEARER_TOKEN` in `env_constants.py` back to a placeholder
   (e.g. `"eyJ..."`) before you do anything else with git.
3. Run `git status` and `git diff` and confirm **no token, tenant name, or customer-specific value**
   is about to be committed. If `env_constants.py` is tracked, either revert it
   (`git checkout -- env_constants.py`) or commit only placeholder values — never the real token.
4. Record the final succeeded/failed counts and the archived data location in the ticket.

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| HTTP 401 on any request | Bearer token expired (~8h) or wrong tenant | Regenerate token for the correct tenant, update `BEARER_TOKEN`, re-run the failed phase |
| `scrollId` keeps coming back `"invalid"` | Token expired mid-scroll | Get a fresh token, restart that phase (Phase 0 or 1) from scratch |
| Connection errors / wrong host | `use_proxy` doesn't match where you're running from | Set `True` from a dev machine, `False` if inside the cluster |
| Frequent 429/503 despite retries | Too much concurrency against the platform | Lower `MAX_WORKERS` and/or `entity_batch_size` in `env_constants.py` |
| Phase 2 output much smaller than Phase 1 | Expected if `CREATED_DATE_CUTOFF` excludes most entities — confirm this is the intended window | Re-confirm cutoff with requester before proceeding |
| Large `warnings.json` in Phase 3 | Entities with no `grmprocessstate` in history, or upstream filter issue | Spot-check a few; escalate if the count looks wrong, don't just proceed |
| Phase 4 `failed` list non-empty | Per-entity update rejected or errored | Check the `reason`/`response` field per entry, retry just those IDs, escalate persistent ones |

---

## 9. Security notes (read once, follow every time)

- The bearer token is a personal credential, not a service account key — never commit it, log it, or
  share it outside this run.
- `env_constants.py` holds customer-identifying values (tenant name, pod) in addition to the token —
  treat the whole file as sensitive for the duration of the engagement.
- If you ever find a real token already committed in this repo's history, do not reuse it — treat it as
  compromised, have it revoked/rotated, and flag it so the history can be scrubbed.
