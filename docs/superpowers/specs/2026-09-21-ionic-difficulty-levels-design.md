# Ionic Mode: Difficulty Levels 2-6 — Design

Date: 2026-09-21
Status: Approved, ready for implementation planning

## Background

Bond Builder's ionic mode currently has one flat difficulty progression (`CHALLENGES`, an ordered array of 6 compounds covering H-Ca main-group ions), with hint-reveal driven by hardcoded index thresholds (`challengeIndex < 3`, etc.). This was the original scope before covalent mode's visual rework took priority (see `2026-09-19-covalent-visual-builder-design.md`).

This design adds five new ionic difficulty levels on top of the existing one, each introducing exactly one new naming/bonding skill: transition metals with a given oxidation state, simple polyatomic ions, more polyatomic ions plus bracket/subscript notation, transition metals with an undisclosed oxidation state, and common (trivial) names in place of systematic names.

Covalent mode's equivalent difficulty ladder, a cross-mode "covalent builds the ion, which then unlocks it in ionic mode" relationship, a full replayable random-draw bank system, and a popup/breakout selector for high-oxidation-state metals (e.g. manganese) are all explicitly out of scope for this design — see "Non-goals."

## Goals

- Six ionic levels (the existing one, plus five new), each isolating one new skill so a student who struggles knows exactly what's new.
- Transition metals and polyatomic ions integrate into the existing bonding/matching engine with minimal new logic — reuse the existing charge-balance and formula-ratio matching wherever possible.
- The data model change (grouping challenges under explicit level objects with per-level hint config) should make it easy to extend individual levels with more content later, and should not block building a random-draw bank per level in the future.

## Non-goals (explicitly out of scope for this pass)

- Covalent mode's difficulty ladder — a separate design, planned next.
- Any cross-mode "build it in covalent to unlock it in ionic" gating — explicitly deferred by the user; for now all new ionic content is simply available once its level is reached.
- Replayable random-draw content banks — the level data structure is being shaped to make this easy later (see "Data model"), but this pass still uses fixed ordered challenge lists per level, matching the user's own assessment that early/mid levels don't have enough content variety yet to benefit from randomization.
- High-oxidation-state metals requiring more than 2-3 commonly-taught states (e.g. manganese, which has 5+ commonly-cited oxidation states and cannot fit as static tiles in the periodic-table grid's single-cell-per-position layout). Captured for a future "challenge tier" that combines complex ionic and covalent content and introduces a popup/breakout state-selector — not this pass.
- Additional polyatomic ions beyond this pass's list (nitrite, sulfite, permanganate, chromate, dichromate) — captured as a future extension list, not built now.

## Level ladder

| Level | New skill | Formula reveal | Charge/state reveal |
|---|---|---|---|
| 1 (existing, unchanged) | Simple main-group ions (H-Ca) | First request only, then hidden | Shown, then hidden |
| 2 | Transition metals, oxidation state **given** via Roman numeral in the name | First request only, then hidden (same pattern as Level 1 — once you know the metal's charge from the name, the charge-balance method is unchanged) | N/A (state is in the name) |
| 3 | Simple polyatomic ions, all ±1 charge — no bracket notation needed yet | First request only, then hidden | N/A |
| 4 | More polyatomic ions (charges up to ±3), forcing bracket/subscript notation (e.g. `Ca(OH)₂`) for the first time | First request only, then hidden (models the bracket syntax once before hiding it) | N/A |
| 5 | Transition metals, oxidation state **not given** in the name | **Always shown** — this is the only way to deduce which state is meant without a Roman numeral, mirroring how real chemistry handles this ambiguity | Name omits the Roman numeral entirely; student deduces the metal's charge from the shown formula's ratio against the known anion charge |
| 6 | Common (trivial) names in place of systematic names, e.g. "Ferric Oxide" not "Iron(III) Oxide" | **Never shown** — hardest level, pure recall (common-name → charge) plus deduction, no formula fallback | Name uses the common name only |

Naming style per level is baked directly into each level's challenge data (each level's `name` strings are pre-written in the appropriate style) — there is no separate "naming mode" flag to compute from, keeping the challenge data self-describing.

## Content lists

**Level 2 — transition metals (capped at 2 commonly-taught states each):** Copper (Cu⁺/Cu²⁺), Iron (Fe²⁺/Fe³⁺), Lead (Pb²⁺/Pb⁴⁺), Tin (Sn²⁺/Sn⁴⁺). Paired only with anions already known from Level 1 (Cl⁻, O²⁻, S²⁻) so the only new skill under test is picking the correct charge-tile. Example dockets: Copper(I) Oxide (Cu₂O), Copper(II) Oxide (CuO), Iron(II) Chloride (FeCl₂), Iron(III) Chloride (FeCl₃), Lead(II) Oxide (PbO), Lead(IV) Oxide (PbO₂), Tin(II) Chloride (SnCl₂), Tin(IV) Chloride (SnCl₄).

**Level 3 — simple polyatomic ions (all ±1, every compound a 1:1 ratio):** Hydroxide (OH⁻), Nitrate (NO₃⁻), Ammonium (NH₄⁺). Paired with familiar ±1 partners (Na⁺, K⁺, Li⁺, Cl⁻). Example dockets: Sodium Hydroxide (NaOH), Potassium Nitrate (KNO₃), Ammonium Chloride (NH₄Cl), Lithium Hydroxide (LiOH).

**Level 4 — more polyatomic ions, bracket notation required:** Sulfate (SO₄²⁻), Carbonate (CO₃²⁻), Phosphate (PO₄³⁻), Acetate (CH₃COO⁻). Example dockets: Calcium Hydroxide (Ca(OH)₂), Magnesium Nitrate (Mg(NO₃)₂), Aluminium Sulfate (Al₂(SO₄)₃), Ammonium Sulfate ((NH₄)₂SO₄ — polyatomic on both sides), Calcium Phosphate (Ca₃(PO₄)₂), Calcium Acetate (Ca(CH₃COO)₂).

**Level 5 — same 4 metals as Level 2, oxidation state undisclosed:** Every compound reuses a Level 2 pairing with the naming/reveal flipped. Example dockets: "Copper Oxide" (formula `CuO` shown → tells them it's Cu²⁺, not Cu⁺), "Iron Chloride" (formula `FeCl₃` shown → Fe³⁺), etc.

**Level 6 — common names, formula hidden:** Ferrous/Ferric (Fe²⁺/Fe³⁺), Cuprous/Cupric (Cu⁺/Cu²⁺), Stannous/Stannic (Sn²⁺/Sn⁴⁺), Plumbous/Plumbic (Pb²⁺/Pb⁴⁺). Example dockets: Ferric Oxide, Cuprous Chloride, Stannic Oxide, Plumbous Chloride.

## Data model & engine changes

**Transition metals break a current assumption.** A workspace atom is currently `{id, symbol}`, with charge/color looked up from a shared `ELEMENTS[symbol]` table — one charge per symbol. Copper needing two different charges depending on which palette tile was clicked means charge can no longer be purely a function of symbol. Fix: a workspace atom becomes `{id, symbol, charge, bg, fg}` — charge and color are captured from the specific palette tile at spawn time and travel with the atom instance, rather than being re-derived from a shared table on every lookup. The palette gains two tiles for copper (labelled "Cu⁺" and "Cu²⁺", or equivalent), both spawning an atom with `symbol:"Cu"` but different `charge`/`bg`/`fg`. Formula counting/matching is unaffected, since it already counts purely by `symbol` and both copper tiles share that symbol — exactly like real chemistry.

**Polyatomic ions are new fake "elements."** Each has its own symbol (`"OH"`, `"NO3"`, `"NH4"`, `"SO4"`, `"CO3"`, `"PO4"`, `"CH3COO"`), a single fixed charge, no group/period position, and a `minLevel`. They render as pill-shaped tiles in a new "common ions" strip below the periodic-table grid (not on the grid itself, since they have no natural period/group position and forcing one onto an existing empty cell would misrepresent what the grid means). At bond-time they behave identically to any monatomic ion — the existing gcd-ratio-reduction matching logic already treats symbols opaquely, so a compound like `Ca(OH)₂` requires zero new matching logic, only data.

**Levels become an explicit structure, replacing magic-number index thresholds.** The flat `CHALLENGES` array and its hardcoded `challengeIndex < 3` style checks are replaced with an array of level objects, each carrying its own hint configuration and challenge list:

```js
const LEVELS = [
  { name: "Level 1", formulaMode: "first-only", revealCharges: true,  challenges: [ /* existing 6 */ ] },
  { name: "Level 2", formulaMode: "first-only", revealCharges: false, challenges: [ /* 8 transition-metal compounds */ ] },
  { name: "Level 3", formulaMode: "first-only", revealCharges: false, challenges: [ /* 4 simple polyatomic compounds */ ] },
  { name: "Level 4", formulaMode: "first-only", revealCharges: false, challenges: [ /* 6 harder polyatomic compounds */ ] },
  { name: "Level 5", formulaMode: "always",     revealCharges: false, challenges: [ /* same 8 pairings as Level 2, state undisclosed */ ] },
  { name: "Level 6", formulaMode: "never",      revealCharges: false, challenges: [ /* 4 common-name compounds */ ] },
];
```

`formulaMode` is three-valued (`"first-only"` — matches today's existing behavior; `"always"` — needed for Level 5, since the formula is the only way to deduce an undisclosed oxidation state; `"never"` — Level 6's maximum-difficulty capstone). `revealCharges` is only ever `true` within Level 1 itself (mirroring the existing behavior where it starts shown and ends hidden partway through); every level from 2 onward keeps it `false` since main-group charges are assumed mastered by then — it does **not** reset back to `true` at Level 2. This flag only ever controls the charge badge on already-known **main-group** tiles. Transition-metal and polyatomic-ion tiles are unaffected by it and always display their own charge as part of the tile itself (e.g. a tile labelled "Cu²⁺"), since the charge *is* what distinguishes one such tile from another — there would be no way to tell a Cu⁺ tile from a Cu²⁺ tile otherwise. There is no separate flag for "is the oxidation state given in the name" — that's already fully determined by which level's (pre-written) challenge names are in play.

**Palette tiles are level-gated.** Every new tile (each transition-metal state, each polyatomic ion) carries a `minLevel`, and the palette only renders tiles with `minLevel <= current level` — so a Level 1 student is never shown copper or hydroxide tiles they have no use for yet.

## Testing approach

No automated test suite exists in this project and this design doesn't introduce one, consistent with the covalent-mode rework. Verification is manual/interactive: for each new level, confirm the correct tiles are available (and no higher-level tiles leak in early), confirm the hint text/reveal behavior matches the table above, and confirm every example compound in each level's content list can actually be built and correctly validates as a match.
