# Ionic Mode: Level-Up Announcement & Randomized Order — Design

Date: 2026-09-21
Status: Approved, ready for implementation planning

## Background

Playtesting the 6-level ionic mode (shipped earlier this session) surfaced two rough edges:

1. Leveling up just resets the moves/score counters and swaps a feedback line — it doesn't feel like progress was made, just like the counter reset.
2. Each level's challenges always play in the same fixed order, so a student can learn "Cu(I), Cu(II), Fe(II), Fe(III)..." as a sequence to guess through rather than actually recognizing each compound.

This is a small follow-up to the existing `LEVELS` feature, not a new subsystem — scoped as its own short spec since it's unrelated to the covalent coordinate-bonds work happening in parallel.

## Goals

- Leveling up produces a clear, deliberate acknowledgment moment — not a silent reset.
- Each level's challenge order is randomized per playthrough (Level 2 onward), removing the "memorize the sequence" shortcut.

## Non-goals

- The full replayable random-draw bank system (drawing N from a larger pool) — still deferred, as originally planned. This is pure order-shuffling of the existing fixed challenge lists, not a content-pool change.
- Randomizing Level 1 — stays in its original curated order (intro/tutorial level, deliberately sequenced for a first-time learner), per explicit confirmation.

## Design

**Level-up banner.** Reuses the existing win-banner UI pattern (the same `.banner.win` styling and docket-slot injection point already used for the final "Docket cleared" screen) as an interstitial, not just a terminal state. When a level's last challenge is solved, instead of immediately resetting state and rendering the next level's first docket, the docket area shows a banner naming the level just unlocked, its hint, and how many compounds it contains, with a "Continue" button. Clicking Continue is what actually performs the level-advance (reset moves/score/bench, pick the level's shuffled order, render the first request) — the reset doesn't happen until the student acknowledges it.

**Randomized order.** Each level object gains a played-order concept separate from its authored `challenges` array: when a level is entered (game start for Level 1, or clicking Continue on the level-up banner for Levels 2+), a shuffled *copy* of that level's `challenges` array is made (Fisher-Yates) and iterated via the existing `challengeIndex`, for levels 2 through 6 only. Level 1 always uses its authored order unshuffled. The source `LEVELS[n].challenges` arrays are never mutated, so they stay stable/inspectable and nothing else that reads them (e.g. the final win banner's total-compound count) needs to change.

## Testing approach

Manual/interactive, matching every other feature on this project (no automated test suite). Verify: completing Level 1 shows the new level-up banner (not an immediate silent transition) before Level 2's first request appears; completing a level plays through all of that level's compounds exactly once with no repeats/omissions, in a different order across at least two separate playthroughs; Level 1's order is never shuffled across multiple playthroughs.
