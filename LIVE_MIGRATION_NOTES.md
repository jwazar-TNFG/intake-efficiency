# Intake Dashboard — Live Migration Notes (2026-08-13, Vera/Jon)

## Goal
Make the intake-efficiency dashboard read 100% live from Supabase so the
launchd refresh job (`com.clawd.intake-dashboard.plist` → `scripts/refresh_intake_dashboard.py`)
can be retired with ZERO data loss.

## Key finding: RLS blocks the browser
Browser anon/publishable key (`sb_publish...`) access via REST:
- readable: intake_specialists (22), default_shifts (169), intake_shifts (2260)
- BLOCKED (RLS → 0 rows): leads, rep_logs
=> The refresh job is the numeric engine (LD counts, heatmap, pace, MTD signed),
   NOT vestigial. Killing it as-is freezes all numbers permanently.

## Solution: SECURITY DEFINER RPCs (RLS stays intact, only aggregates exposed)
Created in Postgres (public schema), granted EXECUTE to anon, authenticated:

1. get_intake_mtd_signed() -> (intakespecialist text, cnt bigint)
   - rep_logs, signed is TEXT (YYYY-MM-DD HH:MM:SS); text compare on YYYY-MM-DD
   - excludes kbonilla@thenofaultgroup.com
   - Verified via anon REST: 22 rows, total 1475 (matches script)

2. get_intake_ld_today() -> (intakespecialist text, status text, hour int, cnt bigint)
   - leads, dateLockedDown is TIMESTAMP; today window
   - excludes submitterName 'Katiria Bonilla'
   - Verified via anon REST: total 61 (matches script)

## Still TODO to fully go live (front-end)
The Python script also computes department-level PACE stats that need RPCs too:
- yesterday_ld, yesterday_same_time, last_week_same_time, last_week_total
  (all COUNT(*) on leads by dateLockedDown windows, exclude kbonilla)
- (optional) a get_intake_pace() RPC would cover these.

Then rewrite index.html fetchData():
- replace `fetch('./data.json')` numeric fields with RPC calls
- rebuild reps[] (ld/still_ld/came_in/dropped/hourly), department totals, mtd_signed
- keep existing live schedule merge (already works)
- Validate side-by-side vs data.json BEFORE disabling launchd job.

## Data-quality caveat
rep_logs has phantom duplicate-id rows (White-cases bug, 2026-08-12). MTD signed
counts `signed`-dated rows; re-validate the RPC output against official numbers
before relying on it as the sole source.

## Done today
- Fixed deploy key (new: ~/.ssh/intake-efficiency-deploy-new; ssh config alias github-intake repointed; old config backed up)
- Pushed backlog: main + gh-pages now in sync (dashboard unfrozen after ~7wk June-22 freeze)
- Removed redundant hourly GitHub Action (.github/workflows/update-data.yml) — pinged Pages rebuild only, computed nothing
- Created + granted the two RPCs above; verified anon-key REST access

## NOT done (deliberately)
- launchd job NOT disabled — remains the numeric engine until live front-end is built + verified.

## VERIFICATION COMPLETE (2026-08-13 ~13:51)
Built index-live.html (copy of index.html) with buildLiveData() calling 3 RPCs.
Headless Node harness compared live-built data object vs current data.json:

DEPARTMENT: total_ld 61=61, still_ld 36=36, came_in 25=25, dropped 0=0,
  specialists_with_ld 12=12, hourly_totals/came_in/dropped EXACT MATCH.
MTD: total 1475=1475; all 22 intaker counts EXACT MATCH.
PER-REP: all 12 reps ld/came_in/dropped + hourly heatmap EXACT MATCH.
PACE: live 69/70 vs snapshot 68/67 — expected drift (live real-time 13:51 vs frozen 13:39). Live is fresher.
MTD phantom-dup check: COUNT(*)=COUNT(DISTINCT id)=1475, gap 0 — NOT contaminated.

RPCs (public schema, SECURITY DEFINER, GRANT EXECUTE anon+authenticated), all TZ=America/New_York:
- get_intake_mtd_signed()
- get_intake_ld_today()
- get_intake_pace()

## REMAINING BEFORE KILLING JOB
1. Human review of index-live.html rendered in a real browser (Jon) — schedule merge + visual.
   NOTE: index-live.html's live schedule merge overwrites rep name/abbrev/hours/rate; verify names render (not raw emails).
2. Promote index-live.html -> index.html on a branch; push; confirm live GitHub Pages renders correctly.
3. THEN disable launchd: launchctl bootout / unload com.clawd.intake-dashboard; move plist aside.
4. Keep data.json in repo as last-known fallback (harmless).
