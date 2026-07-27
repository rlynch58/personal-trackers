# Architecture Notes

Non-obvious decisions in this app that would otherwise have to be re-derived
from reading the code. Written for someone who knows the app but hasn't read
the source.

## 1. Local-calendar date keys

Every day in this app is a `YYYY-MM-DD` string built from the browser's
*local* clock, not UTC. A day runs local midnight to local midnight, and
which day something belongs to is decided once, at write time, by that local
date — never recomputed later from a raw timestamp.

This matters for backdated entries. If you log a meal at 12:15 AM, it gets
stamped with the local date key for the day that just started — but because
that's arguably "still last night" to the person logging it, the date key it
lands under is whatever the local calendar says *at that moment*, i.e. the
new day. The important guarantee isn't about which day a 12:15 AM entry
lands on (that's just "today" as the device sees it) — it's that once a row
carries a date key, everything downstream (history grouping, streak
calculation, goal comparison) trusts that stored key completely and never
re-derives "which day is this" from a timestamp using UTC or wall-clock math
at read time. A day is whatever it was bucketed as when it was written,
full stop.

The practical payoff: streak math, history grouping, and goal comparisons
never flip a day's bucket depending on what timezone math happens to run at
read time. "Today" and "yesterday" in the streak walk-back are computed with
the exact same local-date logic used to bucket the underlying rows, so the
two can never disagree — there's no seam where a UTC "today" could
accidentally treat a 12:15 AM entry as belonging to the wrong day.

## 2. Water save serialization

Water is logged by tapping quick-add buttons, and taps can come in fast —
a user might add water three times in two seconds. Each tap needs to persist
before the next one lands, or two overlapping saves could race and the
slower request's write could stomp over the faster one's more up-to-date
total.

Water saves are serialized through a single promise chain: each new save
request attaches itself to the end of whatever chain of saves is already
pending, rather than firing independently. This guarantees saves hit the
database in the order the user made them, one at a time, regardless of how
close together the taps were or how the network reorders responses.

A save that fails is caught and surfaced as an error, but the chain itself
is never allowed to stay broken — a rejected link has to resolve onward so
every later save still gets its turn instead of being silently skipped
forever because an earlier one failed.

## 3. Delete/undo behavior

Deleting a meal commits to the database immediately — it is not a soft
delete waiting out an undo window before actually removing anything. The
row is removed from local state and the delete request fires right away;
the "Undo" toast is just an offer to bring it back, not a pending state that
blocks the delete.

This is deliberate: if deletes waited for the undo window to expire before
committing, a browser tab closing mid-window, or the app being backgrounded
on mobile while the timer is still running, could leave a "removed" meal
never actually deleted server-side — the UI says it's gone, the database
disagrees, forever.

Undo does not reverse the delete — there is nothing to reverse, the delete
already happened. Instead, undo re-inserts the meal as a brand-new write,
using the cached copy of the row that was captured right before it was
removed. This is why undo has to wait for the original delete to actually
finish before it inserts: re-inserting while the delete is still in flight
risks either a duplicate-key error (if the delete lands after the insert) or
silently losing the row again (if the delete completes after re-insertion
and somehow reruns). The undo path explicitly awaits the original delete's
outcome first.

## 4. Last-write-wins tokens

Several independent async operations in the app follow the same pattern:
a module-level counter (e.g. `loadToken`, `weightSeriesToken`,
`streakToken`, `estimateToken`) is incremented every time that kind of
request starts. The value at the moment a request starts is captured, and
when the request resolves, its captured value is compared against the
counter's *current* value. If they no longer match, a newer request of the
same kind has started in the meantime, and the result in hand is stale —
it's discarded instead of being applied to app state.

This exists because network responses don't always arrive in the order
their requests were sent. Without this guard, a slow response to an old
request (e.g. switching to a new day, then quickly switching to another day
before the first day's data has finished loading) could arrive *after* a
newer request's response and overwrite current, correct state with stale
data — silently showing the wrong day's numbers, or an outdated streak,
with no error to indicate anything went wrong.

Each of these tokens is scoped to its own concern (day/date loading, weight
trend range changes, streak refreshes, AI meal-estimate calls) and they're
independent of each other by design — a slow weight-trend fetch can't block
or corrupt a streak refresh, and vice versa. A couple of write paths (meal
insert/move/edit) also read the current `loadToken` without incrementing it,
purely to check whether the user has switched to a different day while the
write was in flight — if so, the result is discarded rather than being
attached to the now-wrong day's in-memory meal list.

## 5. Streak band rules

The three tracked metrics — protein, calories, water — use different rules
for what counts as a "hit" on a given day, and the difference is
intentional, not an oversight to unify later.

**Protein and water** are hit-at-least: a day counts if the day's total is
greater than or equal to that day's goal. More is never a problem for these.

**Calories** are different: a day counts only if the total falls within a
symmetric band around that day's goal — currently ±10% (the tolerance is a
single named constant, `CALORIE_STREAK_TOLERANCE = 0.10`, currently `0.10`).
The goal is a target to land *near*, not a ceiling to stay under — eating
well below it is treated as just as much of a miss as going over it, since
the goal itself is already a deliberately modest target. The lower edge of
the band does double duty: it's also what correctly fails an
under-logged/barely-logged day, since a day with little or nothing recorded
will fall below the floor on its own without needing a separate "did they
even log anything" check.

**Today is handled differently per metric.** Protein and water walk
backward from yesterday, then fold today's value in on top *if today is
already a hit* — today not yet being a hit doesn't break anything, it's just
not counted yet ("today-pending"). Calories go further: today is *always*
pending and is never folded in, hit or not, until the day is actually
closed out. This is because the band makes a partial day genuinely
misleading mid-day — a mostly-empty day can sit inside the ±10% band
by coincidence early on and then drift out of it as more gets logged, so
evaluating today's calories against the band at all (even to count a
temporary "hit") would be scoring an incomplete number.

**A gap day — nothing logged at all — breaks the streak for any metric,**
regardless of that metric's hit rule. A missing day is never assumed to be
a pass; it's the one case that isn't even given to the hit-test function,
it's checked and rejected first.

Goals are looked up per day through a `goalsFor(dateKey)` function rather
than reading a single global goals value directly. Every streak/history
calculation asks "what were the goals *as of this date*" instead of "what
are the goals right now," so historical days are scored against the goals
that were actually in effect at the time, not today's.

`goalsFor` resolves this against a `goal_periods` table — one row per goal
change ever, each with an `effective_date` and the three dated metrics
(calories, protein, water). For a given date it finds the row with the
**greatest `effective_date` that is still `<= the requested date`**: the
most recent goal change at or before that day. The rows are held in memory
sorted ascending and loaded once per data refresh (a small, bounded set —
never fetched per-day or per-render).

The return shape is a **merge, and deliberately never narrows**:
`goal_periods` only stores calories/protein/water, but the rest of the app
still reads `sodium` and `sugar` off the result of `goalsFor` (the
dashboard's sodium/sugar line does). So `goalsFor` returns the global goals
object with only calories/protein/water overlaid from the matched period —
sodium and sugar pass straight through from the global goals untouched.

Fallback chain, so it can never return `undefined` or a partial object: if
no period's `effective_date` is on or before the requested date (a date
earlier than the oldest goal record — shouldn't happen given the backfill,
but guarded), it falls back to the *earliest* period row; if there are no
period rows at all, it returns the global goals directly.

The `goal_periods` table was backfilled on creation with one row per user,
dated to that user's earliest tracked day and populated from their current
goals — so all pre-existing history resolves correctly rather than falling
into a gap.

**Dual-write, on purpose.** Saving goals in settings writes to *two* places
and this is intentional, not redundancy to clean up. `user_settings`
(JSONB) stays the single source of truth for the *current* goals and is the
only place `sodium` and `sugar` live at all. `goal_periods` is purely the
*historical* record for the three dated metrics: a save upserts a row dated
to today's local calendar date (on conflict with an existing same-day row it
updates it, so several edits in one day collapse to one period rather than
erroring). After a save the in-memory period list is reloaded and streaks
recomputed, so history and streaks reflect the new goals without a page
reload.

**Known edge — the two writes aren't transactional.** If the `user_settings`
write succeeds but the `goal_periods` upsert then fails, the two can diverge
for that day: current goals update but the dated history for today doesn't.
This is self-announcing (the save surfaces an error in the banner rather
than silently claiming success) and self-healing (saving again re-attempts
both writes). It's deliberately *not* wrapped in a database transaction —
the divergence window is one day, fully recoverable, and the app is headed
for a native rewrite that makes hardening this specific web-code path not
worth the complexity. See §12.

## 6. History card calorie coloring (v1.36)

The History list's calorie figure is colored using the exact same band
logic as the streak calculation — same `CALORIE_STREAK_TOLERANCE` constant,
same "goal for that specific day" lookup. A day is green only if it falls
within the band; it's red otherwise, whether it's over the top edge or
under the bottom edge.

This was a deliberate fix (previously the card used an older "green if
under the cap" rule, which disagreed with the streak logic and could show a
badly under-logged day as green while the streak correctly scored it as a
miss). The two now share one constant on purpose. If the tolerance or the
band logic ever changes, both places must move together — the shared
`CALORIE_STREAK_TOLERANCE` constant is the single source of truth for what
"counts" as on-target for calories, anywhere in the app.

## 7. History query scope

The visible History list is capped to the last 45 days — it's a display
list, not an analytical one, and there's no reason to fetch further back
than what's ever shown on screen. Streak calculations, however, run their
own separate, unbounded query that ignores this cap entirely.

The reason for the split: a real "current streak" needs to be able to look
back as far as it takes to find the actual start of an unbroken run, which
could easily be longer than 45 days for someone with a long streak going.
If streaks were computed from the same capped query the History list uses,
a genuine 60-day streak would silently get truncated to whatever the cap
allowed, understating it — or worse, misreporting where the streak actually
broke. Decoupling the two means the visible history list can stay cheap and
bounded while streak math stays correct regardless of how far back it has
to look.

## 8. Water-null handling

A `health_logs` row can exist for a day where only weight was logged and
water was never touched — in that case water is stored as `null`, not `0`.
That distinction matters: `null` means "this metric was never logged this
day," while `0` is a real, deliberate data point (someone logged zero
ounces of water).

Streak calculations treat a `null` water value exactly the same as a
missing row entirely — both are excluded from the day-value map, and both
are therefore treated as a gap day that breaks the streak. A logged `0`
would instead go through water's normal hit-at-least rule (and fail it,
since `0 >= goal` is false for any real goal) — same outcome, but for a
different, more precise reason. The null-exclusion exists so a day where
someone only logged their weight isn't misread as "confirmed zero water
that day."

## 9. Numeric input ceilings (v1.40–v1.41)

Every numeric field a user can type into (calories, protein, carbs, fat,
sodium, sugar, water, weight — across Settings' daily goals, the quick-add
preset editor, the pending-meal form, the meal-edit form, custom water
amounts, and the weight input) is bounded by a `MAX_*` constant and a
`clampNum(n, max)` helper: `Math.max(0, Math.min(max, Number(n)||0))`.

This shipped in two passes. v1.40 added the `MAX_*` constants and put them
on the inputs as HTML `max` attributes — but that alone did nothing to stop
a bad value reaching the database, since these are plain buttons with no
`<form>` submit in play; a `max` attribute only affects the spinner/keyboard-
arrow behavior and native form validation, neither of which fires here. QA
reproduced this by typing protein=24000/carbs=2000/fat=20000 directly and
having it save. v1.41 fixed the actual gap by calling `clampNum()` at every
save path that reads one of these fields: `confirmPendingMeal()` (shared by
both manual entry and the AI-estimate review form), `saveEditMeal()`,
`saveSettingsFromInputs()` (goals and preset), `commitWeightFromInput()`
(clamped to `MAX_WEIGHT_LBS`, 1000), and the custom-water submit handler
(clamped to `MAX_WATER_ENTRY`, 200 per add).

Goal ceilings and entry ceilings are separate constants for the same metric
(e.g. `MAX_CALORIES_GOAL = 10000` vs `MAX_CALORIES_ENTRY = 5000`) since a
daily goal and a single meal's value are different orders of magnitude.
Sodium is the one exception — `MAX_SODIUM` is shared across both goal and
entry contexts, since a single very salty meal and a full day's sodium
budget are already the same order of magnitude. These are generous ceilings
meant only to catch an accidental extra digit, not tight clinical limits.

## 10. Recent Meals: window, cap, and dedupe

The "Recent meals" quick-add list is fed by `loadRecentMeals()`, which
queries `meals` for the current user, `log_date` within the last **8
days** (not 7 — a same-day-of-week weekly special should still show up a
week later instead of dropping out right at a 7-day boundary), ordered
most-recent-first, with a **150-row safety cap** on the query itself so an
unusually heavy 8-day stretch can't blow up the payload.

That raw result is then deduplicated down to at most **15** entries by
`dedupeRecentMeals()`. Meal names are normalized (`normalizeMealNameTokens`:
lowercased, punctuation stripped, connector words like "and"/"with"/"w"
removed) into token sets, and two meals are merged into the same cluster if
their token sets score above `MEAL_NAME_SIMILARITY_THRESHOLD = 0.4` on
Jaccard similarity (intersection over union) — single-link clustering, so a
name only has to clear the threshold against one existing cluster member,
not all of them. This threshold was tuned against real data: near-duplicate
names that prompted the feature scored ~0.44–1.0 once normalized, while
genuinely distinct short names (e.g. "Chicken salad" vs. "Chicken
sandwich") scored ~0.33 — 0.4 sits between the two with margin on both
sides, erring toward under-merging (a missed merge just costs a slot; a
wrong merge would hide a real meal).

Within a cluster, meals are grouped by their exact trimmed name (the
literal "variants"), and the most-frequently-typed variant wins, tied by
which variant's latest entry is more recent — never a blended/averaged
row, since quick-add has to insert one real logged meal. The winning row
per cluster is what appears in the list, capped to the first 15 after
clustering.

## 11. Known gaps and tech debt

- **`user_settings` stores goals/preset/theme as a single JSONB blob**
  rather than typed columns. Why it matters: there's no schema-level
  validation or querying on individual settings fields — malformed or
  partial JSON silently falls back to defaults rather than failing loudly.
- **The weight trend's up/down coloring** hardcodes "down is good" (green)
  and "up is bad" (red) with no setting to flip that assumption for
  someone whose goal is to gain weight or muscle.

The following items from this list as of v1.38 have since been fixed and
are recorded here only so the fix is traceable: `cssVar()` not resolving
nested `var(...)` references (fixed v1.40 — it now recurses through the
chain); `signOut()` not clearing local state (fixed v1.40 via
`resetLocalState()`, which also bumps every in-flight-request token so a
stale response from the old session can't land after sign-out); the
weight-fetch error banner sharing `state.loadError` with unrelated failure
paths (fixed v1.40 — it now has its own `state.weightFetchError` slot); the
calorie ring's divide-by-zero on an unset goal (fixed v1.40 — a
`calGoalSet` guard now renders an empty ring and "No goal set" instead);
and numeric inputs having no real upper bound (fixed v1.40–v1.41, see §9).

## 12. Strategic direction

This app is committed to a native SwiftUI rewrite. The current web/PWA
version is being kept **data-model-correct** — bugs and inconsistencies in
how data is computed, stored, and scored are worth fixing — but it is
**not** getting further structural or architectural investment. Code here
is being retired, not built on top of indefinitely.

The practical implication: data-model integrity work (streak logic, date
handling, goal scoring, the kinds of decisions documented above) remains
high priority regardless of the rewrite, because that logic and reasoning
carries over conceptually to the native app. Web-only architectural
investment — refactoring this specific implementation's structure,
component patterns, or framework choices — does not carry over, and is not
worth spending further effort on here.

The reasons for going native rather than continuing to invest in the
web/PWA version are HealthKit integration and real (non-web-push)
notifications — both meaningfully better, or only fully possible, in a
native app.
