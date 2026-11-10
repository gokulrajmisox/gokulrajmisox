# Interview Prep — Questions From Your Own Code

How to use this: every question below comes from code you actually wrote and
shipped (or have on a branch). An interviewer who opens your repos will ask
about exactly these decisions, trade-offs, and gaps. Don't memorise the
answers — be able to open the cited file and walk through it.

Status tags: `[main]` = the code is on the repo's default branch.
`[branch: name]` = the work sits on an unmerged branch — say so honestly if
asked. `[known gap]` = a verified defect or limitation, not yet fixed; the
strongest answer names it before the interviewer finds it.

---

## PrivacyFirewall (Kotlin, on-device BERT NER keyboard)

**Q1. Your keyboard loads a 106 MB ONNX model. What happened on a cold start
before your fix, and what did you change?** `[branch: day3-async-model-init]`
`AiDetector`'s constructor did `assets.open("model_quantized.onnx").readBytes()`
plus `ortEnv.createSession()` inline, and `PrivacyKeyboardService.onCreate()`
constructs the engine on the main thread — so every IME cold start blocked
the UI thread on 106 MB of asset I/O and native session init (ANR risk).
The fix is async/lazy init: a `suspend fun initialize()`, `Mutex`-guarded so
it loads exactly once, running the I/O on `Dispatchers.IO`, with a `ModelState`
(`UNINITIALIZED` / `LOADING` / `READY` / `FAILED`) exposed as a `StateFlow`
the keyboard UI observes ("Loading AI privacy model…" → cleared on READY).
Until READY, `detect()` returns empty and only the regex detector runs — a
deliberate safe degradation, not a crash.

**Q2. What happens when a user pastes text longer than BERT's 512-token
limit?** `[known gap]`
`BertTokenizer.tokenize()` never truncates, `AiDetector.detect()` feeds all
tokens to `ortSession.run()`, the call throws past 512 tokens, and the
blanket `catch (e: Exception) { printStackTrace() }` swallows it and returns
an empty list. The user gets no AI warning and no indication anything failed.
The fix is to truncate to 510 tokens (reserving CLS/SEP) before inference —
not yet implemented.

**Q3. Why is your custom WordPiece tokenizer the riskiest code in the repo?**
It is hand-rolled Kotlin doing WordPiece splitting plus exact character-offset
bookkeeping, and `aggregateTokens` (BIO state machine) plus `redactText`
(index arithmetic) build on those offsets — a one-character offset error
redacts the wrong text. The repo's only test is `RegexDetectorTest.kt`; the
tokenizer, BIO aggregation, and redaction have zero tests.

**Q4. Why `StateFlow` + `Mutex` instead of a boolean `isModelLoaded` flag?**
The old flag was a non-volatile `var` written during init and read from
other threads, so visibility wasn't guaranteed. The `Mutex` makes the load
one-shot, and the `StateFlow` gives the UI a real loading/failed state to
render instead of a silent boolean.

---

## GATEKEEP.AI (JavaScript browser extension, prompt firewall)

**Q5. Your tool exists to protect privacy — but it logged users' raw prompts
to the console. How did you find and fix that?** `[main]`
The `Logger` in `modules/utils.js` was completely ungated: `info`/`warn`/
`error` called `console.log` unconditionally. `scanner.js` `scanWithAI`
logged the full raw prompt text on every AI scan, and
`lib/transformer-detector.js` logged the actual detected entity values (real
names, orgs, locations). The fix (PR #1, merged): a `debug` setting defaulting
to `false`, `Logger` no-ops unless debugging is on, logs carry lengths/counts/
types only, and `Logger.error` still logs but redacts finding `value` fields
and truncates strings over 300 chars. Verified by the repo's own suite
(44/44) plus a Logger smoke test.

**Q6. Why does detection run in an offscreen document instead of the service
worker?**
The BERT-NER detector (`@huggingface/transformers`) needs a document context
to run; a service worker has none. So the flow is content script →
`Scanner.scanWithAI` → background service worker → offscreen document →
transformer, with results messaged back. The architecture follows from a
platform constraint, not preference.

**Q7. Your credit-card regex has no Luhn check. What does that cost you?**
`[known gap]` False positives on any 13–16 digit run — order numbers, IDs,
phone fragments. The regex finds digit runs; only a Luhn check proves a
card number. It's a documented precision gap, not a detection gap.

**Q8. You dedupe findings by rule ID / entity type. What's the flaw?**
One pasted prompt containing two different person names collapses to a
single finding row, because dedup keys on the type, not the value. The user
sees "PERSON detected" once and can't tell two names were found.

**Q9. Your sanitizer redacts finding values — but only at the top level.
Explain the remaining hole.** `[known gap]`
`stringify()` redacts `value` fields only when the logged argument is itself
a findings array, and truncates long strings only when the argument itself is
a string. A plain object like `{findings: [...], textLength: N}` is
`JSON.stringify`'d verbatim, so values could reach the console via
`Logger.error` with debug off. Current call sites only pass summaries, so
there is no live leak — but the sanitizer's contract is weaker than
documented. The fix is recursive sanitization.

---
## ews / SilentWindow (Python, XGBoost ICU early-warning prototype)

**Q10. Your training target was the patient's final outcome stamped on every
hourly row. Why is that wrong, and what did you change?** `[main]`
`ml/labeling.py` `assign_hourly_labels` did `df["target"] =
outcome["in_hospital_death"]` — one scalar broadcast to all hourly rows. A
row at hour 4 for a patient who dies at hour 46 was labeled 1, so the model
learned eventual-outcome discrimination, not near-term deterioration (the
reported validation ROC-AUC 0.6990 / PR-AUC 0.2803 are cohort-level
eventual-outcome numbers). The fix, now on main: train on the future-window
proxy label (`assign_future_window_proxy_labels`, 12-hour horizon — positive
only when 0 < endpoint − eval_time ≤ 12h), config-switchable
(`labeling.target`, legacy `eventual_outcome` one line away for A/B
comparison), with 15 regression tests pinning the boundary behaviour
(delta == 12h positive, delta == 0 excluded, survivors all-zero).

**Q11. How do you prove your pipeline has no target leakage?**
Four independent checks, all in code: patient-level stratified seeded splits
with disjointness asserts (no patient in two splits); past-only features
(`extract_patient_features_at_time` filters `time <= T` with a raising
assert; baselines capped at `min(6h, current_time)`); replay reveals only
`t <= T` per step with an assert; and outcome-adjacent metadata (`sofa`,
`saps`, `length_of_stay`, `survival`) is excluded from `feature_cols`.
Calibration (Platt sigmoid) is fit on the validation split only, and tuning
is validation-only.

**Q12. One global lock serializes your whole streaming pipeline. Why is
that a reliability problem, and how would you fix it?** `[known gap]`
`StreamingInferenceService.update()` in `backend/streaming.py` holds a single
process-wide `RLock` across the entire batch — artifact lazy-load, feature
extraction over the patient's full retained history, XGBoost predict,
calibration, accumulator update. The endpoint runs on threadpool threads, so
concurrent batches for *different* patients queue behind one lock: a first-call
model load or a long-history patient stalls scoring for everyone, and even
status reads take the same lock. For an early-warning monitor, delayed scoring
is the failure mode. Fix direction: load artifacts once at startup (fail
fast), narrow the critical section to per-patient state mutation
(per-patient locks), keep compute outside the lock — plus bound per-patient
history, which is currently unbounded.

**Q13. How does your evidence accumulator decide when to alert?**
Layer 3 is a CUSUM-style sequential accumulator over calibrated risk:
STABLE below 0.40, WATCH 0.40–0.75, ALERT at 0.75 and above, with 0.88
per-step decay, a 6-hour refractory period, and a 3-alerts-per-patient
budget. The point is alarm-fatigue control: a single high score doesn't page
anyone; sustained evidence does.

**Q14. Your "without trust layer" ablation row is identical to the full
model. Why?**
Because it isn't a real ablation. Variant B in `ml/ablation.py` only disables
the accumulator's credibility weight; the trust processing is already baked
into the features (credibility-filtered baselines, `*_last_credibility`,
implausible readings excluded from trust histories). A genuine no-trust
ablation needs `replay_cohort` to bypass `process_patient_stream`, which its
flags don't support. The README flags the identical row honestly — the
mechanism is now known.

**Q15. What does your reported 13.6% replay sensitivity actually measure?**
Sensitivity on a replay whose "lead time" is distance to end-of-telemetry
(median proxy lead time 37.0h), not to a real deterioration event — the repo
discloses this as a proxy in its evaluation disclaimers. It is a lower bound
on usefulness, not a clinical sensitivity claim, and I wouldn't quote it
without the caveat.

---

## smartcampus / CampusAI (Python, OSMnx campus navigation)

**Q16. Your A* (Manhattan) returns longer routes than UCS. Why?** `[known gap]`
The Manhattan heuristic is inadmissible: (|dy| + |dx|) × 111000 m/degree
exceeds the true road distance by up to ~41% on diagonal routes, and A*
optimality requires a heuristic that never overestimates. The Combined
heuristic (0.7 × Euclidean + 0.3 × Manhattan) inherits the overestimate.
Even Euclidean is slightly inadmissible here — 1° of longitude at 13.2°N is
≈108,400 m, not 111,000 m, so east–west distances are overestimated by
~2.4% — and closed nodes are never reopened, so a suboptimal first pop is
returned as-is. Measured on real campus routes: Manhattan is suboptimal on
48 of 132 routes (worst +3.40%), Combined on 18, Euclidean on 10 (worst
+1.47%). Only UCS is exactly shortest; A* Euclidean is the best trade-off
(half UCS's node expansions, mean gap 0.016%).

**Q17. How did you prove the suboptimality, and what is the ground truth?**
`[branch: day24-benchmark]` A reproducible benchmark script runs all six
shipped algorithms over every ordered pair of the 12 campus POIs (132
routes), median of 5 repetitions, on the repo's own campus graph (1,059
nodes, 1,920 edges), with UCS as ground truth — UCS is exact for weighted
shortest path, so any A* distance above UCS is a proven suboptimality, not
an opinion. The report states its limits honestly (BFS/DFS use a different
distance metric; accessibility mode not benchmarked).

**Q18. A ramp exists next to a flight of steps, but accessibility mode blocks
the route. Why?** `[known gap]`
`is_edge_accessible` does `edge_data.get(0, {})` on a MultiDiGraph — it
consults only parallel-edge key 0. If edge 0 is `steps` and edge 1 is an
accessible ramp, the ramp is ignored and the route is wrongly blocked. The
fix is to return True if *any* parallel edge is accessible. Pinned by
strict-`xfail` tripwire tests, so the defect is on record until fixed.

**Q19. What happens if someone starts the demo from the wrong directory?**
`CampusPathfinder.__init__` loads `campus_data/demo_locations.json` from a
hardcoded CWD-relative path and, on *any* error, silently falls back to just
Main Gate + Library — so routes to every other place fail or report 0.0 m,
discovered mid-demo, not at startup. `[branch: day25-setup-check]` A
stdlib-only pre-flight script (`scripts/check_setup.py`) now catches it
before the demo: POI file present, parses, every POI named with numeric
coordinates, OSM graph parses, deps installed, port free — exit 1 with a fix
hint on any failure.

**Q20. Why do BFS/DFS report different distances than UCS for the same
route?**
A metric inconsistency, not a search difference: a missing edge `length`
defaults to 1.0 in UCS/A* cost accumulation but 0.0 in
`calculate_path_distance`, which is what BFS/DFS report. Part of the BFS/DFS
gap in the benchmark is this inconsistency, and the report says so.

---
## Tgl (TypeScript, Next.js 14 + Supabase Auth edtech platform)

**Q21. How could a user become an admin in your app — and what stops them?**
`[known gap, fix on branch: day18-insert-role-student]` Through the app,
they can't: the signup trigger hardcodes `role = 'student'`,
`ensureProfile()` hardcodes `'student'`, no form or action accepts a role
field, and the `prevent_role_self_change` trigger blocks role *updates*
unless the caller is `service_role`. The hole is the profiles INSERT policy
on main: `"Users can insert own profile"` checks only `auth.uid() = id`, not
the role — so a client with no profile row could insert `{role: 'admin'}`
directly via the anon key, and `ensureProfile()` would then preserve it. The
fix constrains the insert policy to `role = 'student'`; it still needs to be
re-applied on the live Supabase project, because editing an already-applied
migration changes nothing live.

**Q22. Walk me through a request: what does your middleware actually do?**
`middleware.ts` refreshes the Supabase session per request, then gates by
path: `/admin` requires admin, `/mentor` requires any non-student role. That
is only the first layer — each role home re-verifies server-side with
`getSession()` and redirects unauthenticated users to `/login`. `getRole()`
defaults unknown, missing, or errored roles to `student` (fail-closed: a
privileged user whose profile read fails lands on access-denied, not on a
privileged page). Redirects go through `safeNext()`, which accepts only
same-origin absolute paths, so there is no open-redirect hole via `?next=`.

**Q23. Why three Supabase clients?**
Because each context has different cookie handling: a browser singleton
(used only for a `getSession()` check in the reset-password form), a server
client via `next/headers`, and a middleware client that refreshes cookies per
request. All three use the anon key only — no service-role key exists
anywhere in the repo, which is itself a deliberate boundary.

**Q24. How did you make the mobile drawer keyboard-accessible?**
`[branch: day19-drawer-a11y]` The drawer in `DashboardShell.tsx` had no
Escape handling, no focus move, and no accessible name — unlike the repo's
own `Modal`, which already did all of it. The fix moves focus to the drawer's
Close button on open and returns it to the menu button on close (never stolen
on first mount), closes on Escape, cycles Tab within the drawer, and labels
it `aria-label="Dashboard menu"`. Mouse behaviour is unchanged, and it
typechecks clean (`tsc --noEmit` exit 0).

**Q25. What does your CI check, and what doesn't it?**
`[branch: day20-tgl-ci]` `.github/workflows/ci.yml` runs `npm ci`, `npm run
typecheck`, and `npm run build` on pushes and PRs to main, with placeholder
public Supabase values for the build step only (nothing contacts Supabase at
build time). There is deliberately no lint step — the repo has no ESLint
config, so `next lint` would fail in CI. The workflow hasn't run on GitHub
yet: its triggers are push-to-main and PR-to-main, and no PR exists yet.

---

## foodbridge (JavaScript, React + Supabase food-donation platform)

**Q26. How could a user rate themselves five stars, and how did you close
it?** `[branch: day13-rls-tightening]` The loop needed zero server checks:
listing insert required only `auth.uid() = donor_id`, claim insert only
`auth.uid() = receiver_id`, and ratings insert only `auth.uid() = rater_id`
— with `claim_id`, `listing_id`, and `ratee_id` all client-supplied. So: post
a listing, claim your own listing, rate yourself 5 stars; only
`unique(rater_id, listing_id)` limited it to one fake rating per listing.
The fix tightens the documented RLS in the README setup SQL: a rating now
requires a *completed* claim by the rater for that listing, with `ratee_id`
equal to that listing's donor; claims forbid self-claims, duplicates, and
claims on non-`available` listings. Verified on real PostgreSQL: 12 of 13
behaviour tests pass.

**Q27. Your own testing found a bug in your fix. What was it?** `[known gap]`
The ratings policy as written says `c.listing_id = listing_id` inside its
`EXISTS` subquery — but the bare `listing_id` resolves to the *inner* scope
(claims), so the condition is a tautology and a completed claim on listing
L1 lets the same user rate the donor's *other* listings. The one-line fix
(`c.listing_id = ratings.listing_id`) was validated on the same scratch
database — valid ratings still pass, cross-listing ratings are now rejected —
but it is not yet committed. The honest answer is that the fix is tested,
not shipped.

**Q28. Your README says one thing and your code another. Give an example.**
Two: the documented profile-read policy allows reading only your own row,
but the Claims page fetches donor profiles to render a WhatsApp link and the
Dashboard leaderboard joins donor names — under the documented policy both
return null. And the `/transparency` route is advertised as public without
logging in, but the listings select policy requires an authenticated user.
Either the live database's policies are looser than the README, or those
features are broken on a fresh setup. The fix isn't to silently widen reads
— donor phone numbers would become readable to any authenticated user — but
a scoped rule (donor contact visible only to claimants).

---

## Cross-cutting

**Q29. You have a dozen unmerged branches across six repos. How do you
decide what ships first, and what is your merge discipline?**
Branches are isolated by design: one task per branch, no force-push, no
history rewrites, and merges only on explicit approval. The Tgl cluster
(Days 18–20) is independently mergeable — the three branches touch disjoint
files from the same main tip, so order doesn't create conflicts. When a
branch history is messy (an intermediate commit that briefly broke a file),
the merge should be a squash so the broken state never enters main. And some
branches are deliberately *not* merged yet because the fix isn't live until
something else happens — the Tgl policy fix needs a live-database re-apply,
the profile-README branch went stale when master was rewritten and must be
reworked, not merged as-is.

**Q30. What's the most honest limitation in your portfolio right now?**
Pick the true one for the repo being discussed, and say it first:
PrivacyFirewall's async-init change has never been compiled (no Android SDK
in the environment it was written in); ews's streaming service still holds
one global lock and its `/chat` endpoint ships patient CSV text to Gemini;
smartcampus's A* variants still return non-shortest routes; Tgl's role-insert
fix is branch-only and the live database still runs the old policy; the
foodbridge ratings fix has a known tautology bug with a tested-but-uncommitted
correction. Knowing exactly which of your claims are proven, which are
branch-only, and which are known gaps is itself the strongest interview
signal in this document.

---

## Quick map — question → repo → file

| Q | Repo | File(s) |
|---|------|---------|
| 1–4 | PrivacyFirewall | `domain/AiDetector.kt`, `domain/BertTokenizer.kt`, `service/PrivacyKeyboardService.kt` |
| 5–9 | GATEKEEP.AI | `src/extension/modules/utils.js`, `src/extension/modules/scanner.js`, `src/extension/lib/transformer-detector.js` |
| 10–15 | ews | `ml/labeling.py`, `ml/train.py`, `ml/replay.py`, `ml/ablation.py`, `ml/evidence_accumulator.py`, `backend/streaming.py` |
| 16–20 | smartcampus | `src/pathfinding.py`, `benchmarks/pathfinding_benchmark.py`, `scripts/check_setup.py` |
| 21–25 | Tgl | `supabase/migrations/20261006_profiles.sql`, `middleware.ts`, `src/lib/auth/*`, `src/components/dashboard/DashboardShell.tsx`, `.github/workflows/ci.yml` |
| 26–28 | foodbridge | README setup SQL (RLS policies), `src/pages/*` (scrambled tree — cite components, not filenames) |
