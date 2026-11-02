# SM-2 and its 4-grade (Anki) variant: formulas and defaults

Research for issue #2. Facts only; the design choice belongs to the SM-2 rules ticket.

Sources:

- **[W]** P. Wozniak, *Application of a computer to improve the results obtained in working with the SuperMemo method* (1990 thesis excerpt describing SM-2, in use since 1987): https://super-memory.com/english/ol/sm2.htm
- **[AM]** Anki Manual, Deck Options: https://docs.ankiweb.net/deck-options.html (source: https://github.com/ankitects/anki-manual/blob/main/src/deck-options.md)
- **[AF]** Anki FAQ, "What spaced repetition algorithm does Anki use?": https://docs.ankiweb.net/faqs/what-spaced-repetition-algorithm
- **[AS]** Anki source, `ankitects/anki` at commit `136bd42` (2026-10-29), `rslib/src/scheduler/states/`:
  - `review.rs`: https://github.com/ankitects/anki/blob/136bd42bfbd356fc756b6d3e28cc77b02da11ea7/rslib/src/scheduler/states/review.rs
  - `learning.rs`: https://github.com/ankitects/anki/blob/136bd42bfbd356fc756b6d3e28cc77b02da11ea7/rslib/src/scheduler/states/learning.rs
  - `relearning.rs`: https://github.com/ankitects/anki/blob/136bd42bfbd356fc756b6d3e28cc77b02da11ea7/rslib/src/scheduler/states/relearning.rs
  - `steps.rs`: https://github.com/ankitects/anki/blob/136bd42bfbd356fc756b6d3e28cc77b02da11ea7/rslib/src/scheduler/states/steps.rs
  - Defaults, `rslib/src/deckconfig/mod.rs`: https://github.com/ankitects/anki/blob/136bd42bfbd356fc756b6d3e28cc77b02da11ea7/rslib/src/deckconfig/mod.rs

## 1. Original SM-2 [W]

| Item | Rule |
|---|---|
| Grades | `q` in 0..=5. 5 perfect; 4 correct after hesitation; 3 correct with serious difficulty; 2 incorrect but answer seemed easy once seen; 1 incorrect, answer remembered once seen; 0 blackout. |
| Initial EF | 2.5 |
| Intervals (days) | `I(1) = 1`, `I(2) = 6`, `I(n) = I(n-1) * EF` for n > 2. Fractional intervals are rounded **up** to an integer. |
| EF update | `EF' = EF + (0.1 - (5 - q) * (0.08 + (5 - q) * 0.02))` |
| Minimum EF | if `EF' < 1.3`, set `EF' = 1.3` |
| Failure (`q < 3`) | restart repetitions from `I(1)` (n back to 1), "without changing the E-Factor" (step 6). |
| Same-day drill | after a session, repeat every item that scored < 4 the same day until all score >= 4. |

EF deltas implied by the formula, per `q`: 5: +0.10, 4: 0.00, 3: -0.14, 2: -0.32, 1: -0.54, 0: -0.80.
Ambiguity in [W]: step 5 says to modify EF "after each repetition" (which would apply the 2/1/0 deltas), while step 6 says a failed item restarts "without changing the E-Factor". The text does not settle whether step 6 means "the reset itself leaves EF alone" or "skip step 5 on failure".

SuperMemo versions 1.0-3.0, which used this algorithm, ran from December 13, 1987 to March 9, 1989 [W].

SM-2 has no "learning" state: the first interval is already 1 day. The only sub-day mechanic is the same-day re-drill of items graded < 4.

## 2. Anki's SM-2-derived scheduler (non-FSRS path)

### 2.1 Grades and states [AF][AS]

- 4 buttons: Again / Hard / Good / Easy. Only one failing choice (Again) [AF].
- Card states in source: `New`, `Learning`, `Review`, `Relearning` (`RelearnState` wraps a `LearnState` plus the underlying `ReviewState`) [AS `mod.rs`].

### 2.2 Default constants

| Constant | Default | Source |
|---|---|---|
| Learning steps | `1m 10m` (stored as minutes `[1.0, 10.0]`) | [AS deckconfig `learn_steps`], [AM] |
| Relearning steps | `10m` (`[10.0]`) | [AS deckconfig `relearn_steps`] |
| Graduating interval (Good on last step) | 1 day | [AS `graduating_interval_good: 1`] |
| Easy interval (Easy from any learning step) | 4 days | [AS `graduating_interval_easy: 4`] |
| Starting ease | 2.5 (250%) | [AS `INITIAL_EASE_FACTOR`, `initial_ease: 2.5`], [AM] |
| Minimum ease | 1.3 (130%) | [AS `MINIMUM_EASE_FACTOR`], [AF] |
| Ease delta Again | -0.20 | [AS `EASE_FACTOR_AGAIN_DELTA`] |
| Ease delta Hard | -0.15 | [AS `EASE_FACTOR_HARD_DELTA`] |
| Ease delta Good | 0 | [AS], [AF] |
| Ease delta Easy | +0.15 (no upper cap in code) | [AS `EASE_FACTOR_EASY_DELTA`] |
| Hard interval multiplier | 1.2 | [AS `hard_multiplier`], [AM] |
| Easy bonus | 1.3 | [AS `easy_multiplier`], [AM] |
| Interval modifier | 1.0 | [AS `interval_multiplier`], [AM] |
| New interval (lapse multiplier) | 0.0 | [AS `lapse_multiplier`], [AM] |
| Minimum interval after lapse | 1 day | [AS `minimum_lapse_interval`], [AM] |
| Maximum interval | 36,500 days (100 years) | [AS `maximum_review_interval`], [AM] |
| Leech threshold | 8 lapses | [AS `leech_threshold`] |
| New cards/day | 20 | [AS `new_per_day`] |
| Reviews/day | 200 | [AS `reviews_per_day`] |

Valid ranges enforced in code [AS deckconfig]: starting ease 1.31..=5.0; easy bonus 1.0..=5.0; hard multiplier 0.5..=1.3; lapse multiplier 0.0..=1.0; interval modifier 0.5..=2.0.

### 2.3 Learning (new card) [AM][AS `learning.rs`, `steps.rs`]

With steps `s[0..n]`:

- **Again**: go to step 0 (delay `s[0]`).
- **Hard**: on step 0, delay = `(s[0] + s[1]) / 2` (default: (1m+10m)/2 = 5.5m, which the manual shows as "6m"); if only one step, `min(1.5 * s[0], s[0] + 1 day)`. On any later step, repeat the current step's delay. Does not advance.
- **Good**: advance to the next step; after the last step, graduate to Review with interval = graduating interval (1 d) and ease = starting ease.
- **Easy**: graduate immediately from any step, interval = easy interval (4 d), ease = starting ease.
- With empty learning steps, Again/Hard/Good all graduate with the graduating interval.
- Ease is not touched during learning [AF: failures during learning do not lower ease].

### 2.4 Review [AS `review.rs::passing_nonearly_review_intervals`]

Let `I` = current scheduled interval (days, at least 1), `late = max(0, elapsed - I)`, `EF` = ease, `HM` = hard multiplier, `EB` = easy bonus, `IM` = interval modifier. Each result is multiplied by `IM`, fuzzed, rounded, then clamped to `[minimum, max_interval]`:

| Button | Interval | Minimum | New EF |
|---|---|---|---|
| Hard | `I * HM` | `I + 1` (if `HM > 1`) | `max(EF - 0.15, 1.3)` |
| Good | `(I + late/2) * EF` | `hard + 1` (or `I + 1` if `HM <= 1`) | `EF` |
| Easy | `(I + late) * EF * EB` | `good + 1` | `EF + 0.15` |
| Again | lapse, see 2.5 | | `max(EF - 0.20, 1.3)` |

So Hard < Good < Easy is guaranteed by the minimums. A separate formula applies when a review card is answered **early** (`passing_early_review_intervals`); not detailed here.

Fuzz: Anki randomizes review intervals within a range (`with_review_fuzz`, in `fuzz.rs`). Not covered here; deterministic tests in Anki run with fuzz disabled.

### 2.5 Lapse and relearning [AM][AS `review.rs::answer_again`, `relearning.rs`]

- **Again on a Review card**: `lapses += 1`; `EF = max(EF - 0.20, 1.3)`; the post-relearning interval is computed now as `max(I, 1) * lapse_multiplier` clamped to at least `minimum_lapse_interval` (defaults: `I * 0.0` -> clamped to 1 day).
- If relearning steps exist, card enters Relearning at step 0 (default 10 m). Otherwise it goes straight back to Review with that interval [AM: "will be assigned a new interval of 1 day by default"].
- **In Relearning**: Again -> step 0; Hard -> as in learning (first-step average / repeat current); Good -> next step, and after the last step returns to Review with the interval computed at lapse time; **Easy** -> back to Review with `that interval + 1` day. Ease is not changed during relearning steps.
- Leech: flagged when `lapses` reaches the threshold (8), then every `threshold/2` lapses after (8, 12, 16, ...) per `leech_threshold_met` tests.

## 3. Units: minutes vs whole days

- Anki stores learning and relearning steps **in minutes** (`steps.rs`: "The steps in minutes"; converted to seconds internally). Defaults `1m 10m` and `10m` are sub-day [AS].
- Steps may also be written in hours or days (e.g. `1m 10m 1d`) [AM].
- When a step crosses the day boundary Anki converts the delay to whole days; `maybe_round_in_days` rounds any delay > 1 day to whole days [AM "Day Boundaries"][AS `steps.rs`].
- The review-state intervals (graduating, easy, all Review intervals, minimum lapse interval) are already **whole days** [AS: `scheduled_days: u32`].

Implications for a whole-day scheduler (`today: u32` days since epoch), stated as facts, not a choice:

- A minute-based step cannot be represented as a due date in days; any sub-day step collapses to "due today" (0 days) or must be rounded to >= 1 day.
- Anki's defaults keep a card in Learning/Relearning only within one sitting (1m, 10m); everything leaving those states is in days. So the day-granular part of Anki (Review intervals, lapse interval, graduating/easy intervals, ease math) maps onto a `u32`-day scheduler unchanged.
- SM-2 [W] itself has no sub-day steps; its only same-day mechanic is re-drilling items graded < 4 within the session, which is a session-loop concern rather than a stored due date.
- Anki's manual itself recommends (for FSRS) keeping (re)learning steps under 1 day so they finish the same day [AM, FSRS "A Short Guide"].

## 4. Mapping SM-2 grades to Anki buttons

Neither source defines an official mapping. Observable correspondences:

| Anki | Closest SM-2 behaviour |
|---|---|
| Again | `q < 3`: reset repetitions. SM-2: EF unchanged per step 6 (see ambiguity above); Anki subtracts 0.20. |
| Hard | `q = 3` (EF -0.14 in SM-2 vs -0.15 in Anki). Interval: SM-2 uses `I * EF`; Anki uses `I * 1.2`. |
| Good | `q = 4` (EF unchanged in both). Interval `I * EF` in both (plus Anki's late bonus). |
| Easy | `q = 5` (EF +0.10 in SM-2 vs +0.15 in Anki). Anki also multiplies interval by easy bonus 1.3. |

First intervals differ: SM-2 is 1 d then 6 d then `I * EF`; Anki is learning steps, then graduating 1 d (or easy 4 d), then `I * EF` from the second review onward (no fixed 6-day second interval).

## Could not verify

- The FAQ page [AF] did not state the easy-bonus or new-interval defaults; those come from [AM] and [AS].
- The exact original 1987 SM-2 publication is not online as a separate document; [W] is Wozniak's own 1990 description hosted on SuperMemo's site.
