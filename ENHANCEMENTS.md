# Word Builder — v2 Polish Pass

**Source used:** the live GitHub Pages source, downloaded from
`raw.githubusercontent.com/MrGhostGuy/word-builder/main/index.html` into this
folder (`~/workspace/word-builder-v2/`). Before working, its md5
(`28a94b0f254d46ba506421ab6d5572b3`) was verified **byte-identical** to both
the Muse artifact source (`~/workspace/ts-spaces/word-builder/index.html`,
slug `word-builder`) and the export in `~/workspace/word-builder-src/index.html`
— all three are the same build, so this v2 is a patch on exactly what is live
at https://mrghostguy.github.io/word-builder/. Nothing was pushed anywhere;
these are local files only. Total diff vs the live build: **58 lines**.

## Bugs found and fixed

1. **Stale AI turns leaked into new games (worst bug).** `playYours`,
   `doExchange`, and `doPass` scheduled `setTimeout(aiTurn, 420)`, and
   `aiTurn`'s inner "thinking" callback never re-checked the game state. If
   the player quit to CHANGE FOE, hit NEW GAME, or resigned while the AI was
   "thinking," the pending callback fired into whatever game was current —
   the AI would play tiles into a brand-new game on the player's first turn,
   or move after a resignation and double-count the session record via a
   second `finishGame`.
   *Fix:* every game now carries an id (`G.id = ++gameSeq`); the three call
   sites capture it and `aiTurn(gid)` validates it on entry **and** inside
   the thinking callback (also re-checking `G.over` and hiding the thinking
   overlay when it bails).
   *Proof:* `test-timer.js` (real timers) — play a move, immediately start a
   new game, wait out the timers. Original code: **FAIL** (AI played 3 tiles
   into the fresh game). Patched code: **PASS** (new game untouched).

2. **Dead CANCEL button in the blank-tile picker.** `#blankCancel` existed in
   the modal markup with no event listener anywhere — once the picker opened,
   the only way out was choosing a letter (or reloading the page).
   *Fix:* wired it to close the modal and drop the pending callback. (Every
   other button in the game was audited — all 19 others have listeners.)

3. **Exchange let you redraw the tiles you just gave up.** Both the player's
   `doExchange` and the AI's exchange returned the exchanged tiles to the bag
   and shuffled *before* drawing replacements, so a player could immediately
   draw back the Q they just dumped. Official procedure is the reverse.
   *Fix:* replacements are drawn first; the exchanged tiles go back into the
   bag afterward (both code paths).
   *Proof:* rigged-bag regression test — bag of seven A's, exchange Q+Z,
   both replacements are A's; Q/Z re-enter the bag only after the draw.

4. **Endgame go-out scoring was one-sided.** When a player played their last
   tile, the game added the rival rack's value to the winner but never
   subtracted it from the player left holding the tiles. Regulation scoring
   does both — and the difference can flip the winner in a close finish
   (e.g. 300 vs 308 with 5 points stranded: correct result is a 305–303 win
   for the player who went out; the old code scored it 305–308, a loss).
   *Fix:* `checkEnd` now applies both sides, and the game-over text says so
   ("You gain 12 from the rival rack — PIP-7 also loses 12."). The
   six-scoreless-turns ending already subtracted both racks correctly and was
   left as is.
   *Proof:* rigged-state regression test — 100–100 with Q+Z (20 pts)
   stranded ends 120–80.

5. **VOLT could silently forfeit turns on huge move lists.**
   `aiChooseMove`'s "mid" branch used `Math.max(...moves.map(...))` —
   spreading a very large array (a blank-heavy rack on an open board has
   20,000+ legal moves, and denser boards go higher) can throw a RangeError,
   which `aiTurn`'s try/catch would quietly convert into an exchange/pass.
   *Fix:* plain loop for the maximum. Same result, no stack limit.

6. **Restarting mid-exchange soft-locked the controls.** `startGame` never
   reset the exchange bar / controls / thinking overlay / armed PASS button,
   so quitting to setup while in exchange mode (or while the AI was
   thinking) started the next game with the main controls hidden.
   *Fix:* `startGame` now resets all four pieces of UI state.

7. **Side panel overflowed the R1's 240px screen.** In the stacked
   (≤880px) layout, `.side` kept its desktop `min-width: 250px`, forcing
   ~34px of horizontal overflow at 240px wide. *Fix:* `min-width: 0` inside
   the media query. Board, rack, and fonts were already fluid/`clamp()`-based
   and needed no changes.

## Improvements

8. **Move-generation wall-clock budget (anti-freeze).** The AI's only brake
   was a node cap; measured on a fast desktop CPU, a two-blank rack on an
   open board generates **22,861 legal moves in ~1.0–1.6s** at OMEGA-9's
   settings — on the R1's far weaker CPU that is a long frozen screen with
   the board unresponsive. `genAllMoves` now also sets a deadline
   (1.5s–6s, scaled by difficulty cap, shared across both board
   orientations); when it trips, the generator returns the legal moves found
   so far instead of hanging. On fast hardware nothing changes (the search
   finishes well inside the budget); on slow hardware the AI plays a slightly
   less exhaustive — still fully legal — move instead of freezing.
   *Proof:* `test-deadline.js` runs the clock at 100× — generation aborts
   early and returns a partial set verified 100% legal by the scorer.

9. **OMEGA-9 evaluation is ~3× cheaper per move.** The "best" picker called
   `remainingAfter(m)` up to three times per candidate (22k+ candidates);
   it now computes the post-move rack once and reuses it. Same choices,
   less heat.

## Verified correct — left alone

- **Lexicon:** unpacks to exactly **178,691 words**; **101** two-letter
  words — both correct for TWL.
- **Tile bag:** counts sum to exactly 100 with regulation distribution
  (E×12, blanks×2, Q/Z/J/X/K×1, …).
- **Premium squares:** TW×8, DW×17 (incl. center star), TL×12, DL×24, no
  overlaps, mirror/180° symmetric, matching the official board square for
  square.
- **Scoring engine:** premiums apply per word formed (main + cross) only for
  newly placed tiles, blanks score 0, bingo +50 for a 7-tile play, first
  word must cover the center and be ≥2 letters, connectivity and gap checks
  all enforced. In every simulation, every generated move (human-side and
  AI) was re-scored through `analyze` and matched exactly.
- **Six-scoreless-turns ending** and its rack penalties — already official.
- **No challenge phase:** invalid words are rejected live in the preview
  instead. That's the game's established design (a challenge system would be
  a rules redesign, not a polish fix), so it was not added.
- **AI personalities, caps, hint system (3/game, no charge when no play
  exists), session win/loss records, drag-and-drop + tap + keyboard play,
  blank picker, unseen-tile tracker** — all exercised and working; untouched
  except where the fixes above required.

## Simulation results

Headless harness (`test-harness.js`, DOM stubbed, real game code) plays full
games to game-over — a greedy generator-driven player against each AI, with
**tile conservation checked after every turn** (board + racks + bag +
in-play = the exact regulation multiset of 100) and every move re-validated
and re-scored through the game's own scorer.

- Baseline (original code): 4/4 games finished, one per difficulty.
- Patched code: **16/16 full games finished** (4 runs × 4 difficulties),
  no crashes, no illegal moves, no tile leaks, scores reconciled.
  Sample finals: YOU 366–99 PIP-7 · YOU 258–312 VOLT · YOU 252–267 LEXA ·
  YOU 263–410 OMEGA-9 (the stand-in player is a decent but not optimal
  generator driver, so VOLT+ winning is expected).
- `test-regressions.js`: go-out scoring, exchange draw order, and
  `aiChooseMove` legality on a blank-heavy open board at all four
  difficulties — all PASS.
- `test-timer.js`: stale-timer regression — PASS (and FAIL on the original
  code, confirming the test detects the bug).
- `test-perf.js` / `test-deadline.js`: worst-case generation board and the
  deadline-abort behavior described above.

## Files

- `index.html` — the enhanced game (58-line diff vs live).
- `ENHANCEMENTS.md` — this file.
- `test-harness.js`, `test-regressions.js`, `test-timer.js`, `test-perf.js`,
  `test-deadline.js` — the verification suite (Node; `test-timer.js` needs
  `WB_REAL_TIMERS=1`).
- `README.md`, `icon.png` — carried over from the repo unchanged.

## Not done / still open

- **Not pushed to GitHub and the Muse artifact was not edited** — per
  instructions, this is a local deliverable. Publishing = replacing
  `index.html` in repo `MrGhostGuy/word-builder` (and/or an artifact edit)
  with this file; the creation code/URL stay the same.
- **Physical-R1 feel remains unverified**, as before: the anti-freeze budget
  is tuned from desktop measurements (worst observed case ~1.6s vs a 6s
  OMEGA-9 budget), so on-device OMEGA-9 turns should be confirmed once after
  publishing. If R1 turns still feel slow, the budget constants in
  `genAllMoves` (`Math.min(6000, Math.max(1500, cap / 120))`) are the single
  knob to lower.
