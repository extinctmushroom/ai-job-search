# Daily H-1B data-analyst job scan — routine configuration

Mechanics for the daily scheduled job-scan routine (cron `0 12 * * *` = 08:00
America/New_York).

> **Privacy note.** This file deliberately contains **no candidate personal data**.
> The identity/profile block is marked `[CANDIDATE PROFILE]` below and must be filled
> in only when pasting the prompt into the routine's settings on claude.ai — same
> convention as the `[YOUR_NAME]` placeholders in `CLAUDE.md`. Do not commit a filled-in
> copy to this repository.

The routine was created via the HTTP API, so an agent session cannot edit its prompt —
it has to be pasted in by the account owner.

---

## Diagnosis of the 2026-08-03 run

The run completed but produced nothing verifiable. Two independent causes.

### 1. Blocking: the environment has no general web access

The routine is bound to an environment whose network policy is
**"trusted network access"**, which allows developer infrastructure only. Measured
from inside the container:

| Host | Result |
|---|---|
| `api.github.com` | 200 |
| `registry.npmjs.org` | 200 |
| `pypi.org` | 200 |
| `github.com` | 400 (reached) |
| `api.anthropic.com` | 404 (reached) |
| `linkedin.com` | **000 — CONNECT refused** |
| `boards.greenhouse.io` | **000 — CONNECT refused** |
| employer careers domains (several tested) | **000 — CONNECT refused** |
| `example.com`, `google.com` | **000 — CONNECT refused** |

Every employer careers site and job board is refused at CONNECT; `WebFetch` returns 403
for all of them. The policy is working as designed — it is simply the wrong policy for a
routine whose entire job is reading employer careers pages.

Consequence: **H-1B sponsorship polarity cannot be verified.** `WebSearch` still works
(it runs server-side, not through the container's egress), but it returns titles, URLs
and snippets — never the sentence that says whether an employer sponsors. Since
excluding postings that refuse sponsorship is the routine's hard constraint, the run
cannot deliver its core product while this policy is in place.

**Fix (owner action, cannot be automated):** in claude.ai → Code → Environments, give the
environment full/unrestricted network access, or create one that has it and repoint the
routine there. There is no API surface for changing a network policy.

### 2. Independent defects in the stored prompt

These would bite even with perfect network access:

- **The output had no delivery path.** The prompt ended at "make the report your final
  message" plus "save to REPORT.md". Nobody reads a scheduled session's final message,
  and the container is wiped after each run, so `REPORT.md` evaporated. Neither a
  notification nor a commit was required.
- **No cross-run deduplication**, despite running daily. The repo already ships the
  machinery — `job_scraper/seen_jobs.json`, used by the `/scrape` skill — and the prompt
  neither read nor wrote it (`job_scraper/` still contains only `.gitkeep`). Every
  morning would re-report the same listings.
- **No preflight check.** The run spent roughly ten minutes and ~160k tokens across three
  subagents before discovering it could not fetch anything; one subagent died on a
  session limit. A three-line `curl` check detects this in seconds.
- **No degraded-mode contract.** With no instruction covering "fetch is unavailable", the
  run produced a list that reads like verified matches but is not.

The prompt below fixes all four.

---

## Corrected prompt — fill in `[CANDIDATE PROFILE]`, then paste into the routine

````text
Daily US data-analyst job scan. Each firing is a FRESH session with the repo cloned clean — this prompt is the complete instruction.

## STEP 0 — PREFLIGHT (always first, before any searching)

Run:
for u in https://www.linkedin.com https://boards.greenhouse.io https://www.indeed.com; do echo "$u -> $(curl -sS -o /dev/null -w '%{http_code}' --max-time 12 "$u" 2>&1 | tail -1)"; done

A `000` means the environment's egress policy refused CONNECT — employer careers sites are unreachable from this container.

- All three `000` -> DEGRADED MODE. Do NOT retry WebFetch on job sites this run (every call will 403 and waste the budget). Jump to Step 4, then Steps 5-6.
- Any real HTTP status -> FULL MODE. Continue to Step 1.

Quote the preflight output verbatim in the report's Sources block. Never report sponsorship as "verified" in degraded mode.

## COST DISCIPLINE (every run)
Spawn at most 3 subagents, each with a narrow, bounded brief — a previous run died mid-flight on a session limit after three open-ended "search everything" subagents burned ~160k tokens and returned unverifiable results. Prefer running searches yourself. Cap WebFetch at ~15 calls per run. If you find yourself on the 20th search, stop and report what you have.

## STEP 1 — Load dedup state (this is what makes a DAILY routine useful)
Read `job_scraper/seen_jobs.json` (if missing, start `{"seen": {}}`). Any posting whose URL, or whose company+title pair, already appears there was reported on an earlier day — exclude it from the "new" table. Without this the same listing is re-reported every morning. Also read `job_search_tracker.csv` if present and skip company+role combinations already applied to.

## [CANDIDATE PROFILE] — fill this in when pasting; do not commit it to the repo
Replace this whole block with: name, years of experience, home city and commute constraints (e.g. metro-commutable or remote-US only; never non-US), core tooling (SQL/Python/BI stack, warehouses, ETL), analytics specialisms (funnels, cohorts, retention, segmentation, A/B testing), employment history, and degree.
HARD CONSTRAINT: employer must plausibly sponsor H-1B — any posting that refuses sponsorship is excluded, always.
Target titles: Data Analyst, Senior Data Analyst, Product Data Analyst, Product Analyst, senior-leaning BI Analyst. Fit bands — high: SQL+Python+BI product/growth/payments analytics at mid-senior/senior level; medium: adjacent (analytics engineer, growth/marketing/ops analyst, BI) or seniority off by one; low/drop: heavy ML or data-engineering builds, clearance/citizenship-required, staffing-agency spam, unrelated domains.

## Recency window
Default: postings from the LAST 3 DAYS. WIDENING RULE — before reporting zero, re-run the sweeps once at 7 days and title the report "No new postings in the last 3 days — showing past-week matches". Only report zero if the 7-day pass is also empty, and still include the Sources block. Newest first. If a date cannot be determined, include when otherwise strong and flag "date unknown". Never pad with listings older than 7 days.

## STEP 2 — Search sweeps (FULL MODE; in DEGRADED MODE run these as WebSearch-only, see Step 4)
- "data analyst" postings explicitly offering visa/H-1B sponsorship ("visa sponsorship available", "will sponsor" H1B), filtered recent
- jobright.ai "H1B Sponsor Likely" data-analyst listings
- Local H-1B CAP-EXEMPT employers (can file year-round, no lottery — priority): the major teaching hospitals, universities and nonprofit research institutes in the candidate's metro
- Verified H-1B sponsors with local or remote presence: maintain the target-company list in the routine settings alongside the profile block

LinkedIn CLI (optional, secondary — LinkedIn blocks datacenter IPs; expect failure and move on fast). Only attempt in FULL MODE: run `bun install` in `.agents/skills/linkedin-search/cli`, then from the repo root:
- bun run .agents/skills/linkedin-search/cli/src/cli.ts search -q "data analyst" -l "<metro>" --jobage 3 --limit 20 --format json
- the same with -l "United States" --remote remote, and with -q "product analyst"
One failure (403/999) is enough — record it in diagnostics and stop retrying.

## STEP 3 — Sponsorship verification (FULL MODE only; the core of the task)
For each candidate, pull the full description (LinkedIn CLI `detail <id> --format plain`, else WebFetch) and classify polarity from the ACTUAL posting text: stated-yes / stated-no / cap-exempt-employer / unknown. Quote the exact sentence as evidence. CRITICAL: most postings that mention H-1B do so to REFUSE it ("must be authorized to work without sponsorship now or in the future") — read polarity, never keyword-match. Exclude stated-no. Silence on visas is "unknown", not refusal. "H1B Sponsor Likely" badges are filing-history predictions, not commitments — label them as such.
Then for each surviving high/medium job find the DIRECT application page on the employer's own site (Greenhouse/Lever/Workday/Ashby/company careers domain) and confirm via WebFetch that the requisition is live and matches. Report that as the primary apply link, board/LinkedIn URL secondary. If no direct page can be confirmed, say so — never guess or construct a URL.

## STEP 4 — DEGRADED MODE (preflight blocked)
Sponsorship polarity and requisition liveness CANNOT be verified — WebSearch returns titles, URLs and snippets only. Do not pretend otherwise. Run a BOUNDED WebSearch-only sweep: at most 8 searches, no subagents, no WebFetch on job sites. Produce an explicitly-labelled lead list, titled:
"UNVERIFIED LEADS — sponsorship not checked (network blocked)"
Every row's H-1B column must read "unverified — network blocked", except cap-exempt employers which may read "cap-exempt employer (structural inference, not posting text)". Do not include an "excluded / refused sponsorship" section — you could not read any posting to know. State plainly in the report that the routine is running degraded, and that the fix is to give the environment full network access in claude.ai -> Code -> Environments.

## STEP 5 — The report
Markdown, forwardable as-is:
1. One-line summary: "X new verified matches (Y high fit) — <date>", or the widened-window title, or the degraded-mode title.
2. Table sorted by fit: # | Fit | Title | Company | Location | H-1B signal | Direct apply link | Board link
3. For each high-fit job: 2-3 bullets — why it matches the profile, key requirements, and the exact sponsorship-evidence quote (FULL MODE only).
4. Short "excluded" line naming companies whose postings refused sponsorship (FULL MODE only).
5. MANDATORY "Sources" diagnostics block, one line per source stating what it yielded or exactly how it failed — including the verbatim preflight output. Required even when results are zero, so failures stay visible.
Never fabricate postings, sponsorship claims, dates, or URLs. Report only what search/fetch output actually showed.

## STEP 6 — DELIVER (the run has FAILED if you skip this)
Nobody reads this session and the container is wiped afterwards. The report arrives only if you do all three:
1. PushNotification with the summary wrapped in <routine_summary> tags — first sentence is the phone banner (match count + best match, or the blocker), the rest is the email body. Send it on every run that has results, and on every degraded/blocked/failed run. Skip it ONLY when the run was healthy AND genuinely found nothing new.
2. Write the report to `reports/<YYYY-MM-DD>.md` and overwrite `REPORT.md` with the same content.
3. Update `job_scraper/seen_jobs.json` with every posting evaluated this run (reported or not), keyed by URL, with company, title and date. Then git add the report files plus seen_jobs.json, commit, and `git push -u origin master`. Do NOT open a pull request — this is routine state persistence, and a daily PR is noise. If the push is rejected (branch protection), push branch `job-scan/<YYYY-MM-DD>` instead and note that in the notification.
PRIVACY: the reports and seen_jobs.json committed here must contain job postings and links only. Do not write the candidate's name, address, phone, email, visa status or employment history into any file committed to this repository.

## Known environment constraint
If the preflight keeps failing, the routine is still bound to an environment whose egress policy allows only developer infrastructure (GitHub, npm, PyPI, Anthropic). Job boards and employer careers sites are refused at CONNECT, which forces DEGRADED MODE and makes real sponsorship verification impossible. The fix is the owner's: in claude.ai -> Code -> Environments, give the environment full/unrestricted network access, or create one that has it and repoint this routine to it. Until then, state in every notification that results are unverified leads.
````

---

## Applying the fix

1. **Give the environment network access** (this is the blocking one). claude.ai → Code →
   Environments → the environment used by this routine → set network access to
   full/unrestricted. Without this the routine can only ever emit unverified leads.
2. **Replace the routine's prompt** with the block above, filling in `[CANDIDATE PROFILE]`
   and the target-company list at paste time. Keep that filled-in copy out of git.

After step 1, confirm the fix by checking that the Step 0 preflight returns real HTTP
statuses rather than `000`.
