# Covalent Bonding: Visual Stage Builder — Design

Date: 2026-09-19
Status: Approved, ready for implementation planning

## Background

Bond Builder's covalent mode currently works, but classroom testing showed it isn't intuitive for a student who doesn't already know what the target molecule looks like: bonding is a bare click-to-arm mechanic against a flat bench, and the only visual structural diagram appears once, at the very end, as a reveal. There's no visual feedback of the molecule taking shape while it's being built.

This design reworks covalent mode so the molecule's structure is visible and grows live on screen as the student bonds atoms together, while keeping the low-overhead click-to-bond mechanic (rejected a free-drag-physics interaction model — see "Alternatives considered").

Ionic mode, the element/molecule content set, and difficulty tiering are unchanged and out of scope here. A follow-up design pass will address adding more difficulty tiers/rounds to both modes.

## Goals

- The partial molecule is visible and updates live as bonds are added, not just at the end.
- Students can freely build in any order — branch, chain, join separate fragments, close rings — including orders that don't match the "textbook" build sequence, per playtesting (e.g. bonding both branches off a central atom before closing the ring between them).
- Adding/removing/upgrading bond order is unambiguous even as the structure gets more complex (e.g. a ring), unlike a flat "bonds formed" chip list which becomes hard to map back to specific bonds once there are several.
- Keep the interaction low-overhead: click-to-arm/click-to-bond, not drag-and-drop physics.

## Non-goals (explicitly out of scope for this pass)

- Ionic mode — untouched.
- New molecules, new elements, or difficulty tiers/rounds for either mode — sequenced as a separate follow-up design.
- Manual drag-to-nudge atom positioning — decided against for v1 (see Alternatives considered).
- A global step-back "undo" — per-atom removal covers this need instead (see Interaction flow).

## Alternatives considered

Three interaction models were compared (with working prototypes) before settling on the design below:

1. **Free canvas drag** — student drags atoms freely on an open canvas; dropping one atom on another bonds them; bonded atoms can be repositioned by dragging. Most direct "build the molecule" feel, but requires real drag physics, overlap handling, and touch/mobile support — a much bigger build for a benefit that turned out to be unnecessary once auto-layout (below) proved itself.
2. **Click-to-bond + live auto-layout preview** — kept today's click-to-arm mechanic unchanged, but promoted the existing end-of-round reveal diagram to render continuously. This is the direction chosen, refined further below.
3. **Hybrid drag** — click-to-bond, but bonded clusters could be dragged as a rigid group for manual tidiness. Considered but dropped once the auto-layout relaxation engine (below) proved it keeps structures readable on its own — manual nudging wasn't needed in testing.

The deciding factor for (2) over (1): a working prototype of (2), extended with a relaxation-based layout engine, correctly and automatically laid out a ring closed in a "wrong" (non-sequential) build order, and correctly separated + reorganized two independently-built fragments once joined — the exact scenarios (2) was initially unproven on. See "Layout engine" below.

## Data model

Extends the existing `cState` shape (`workspace`, `bonds`) rather than replacing it:

- Each workspace atom gains `pos: {x, y} | null`. `null` means the atom is still in the tray (unplaced); any other value means it's on the stage.
- Bonds remain `{id, aId, bId, order}` as today.
- No changes to the ionic-mode state, the element/molecule data tables, or the move-economy fields (`movesLeft`, `score`, etc.).

## Layout engine (new)

A lightweight spring-relaxation simulation replaces the idea of computing a "correct" position once at placement time:

- **Attraction**: every bonded pair is pulled toward a fixed rest length (~62px), proportional to how far off that length they currently are.
- **Repulsion**: every pair of atoms on the stage (bonded or not) repels, inverse-square with a cap, which is what naturally separates unrelated fragments and prevents overlapping atoms/labels.
- **Centering**: a small constant pull toward the stage's center keeps the whole structure from drifting off-screen.
- **Seeding**: a newly-placed atom (moving from tray to stage, or the first atom of a new fragment) is seeded at a small random offset from its bonding partner (or near stage center for a fragment's first atom) — not a precomputed "correct" angle. The simulation resolves the final position.
- **Execution**: runs for up to ~150 ticks per change, animated via `requestAnimationFrame` (not snapped instantly) so the settling motion is visible — this doubles as feedback that the tool "understood" the structure just built. A simple convergence check (small enough recent movement) stops the animation early once stable.
- Reruns after every state change that affects the graph: bond added, bond removed, bond order changed, atom removed.

This was prototyped and tested against the specific failure case found in playtesting (a 3-ring closed by bonding both branches off one atom, then joining them last — which produced a degenerate, flattened triangle under a simpler order-dependent placement heuristic). The relaxation approach resolved it into a proper triangle regardless of build order, and was further tested with extra branches and with two independently-built chains joined into one larger structure and then into a ring — all settled into clean, non-overlapping layouts.

## Interaction flow

**Tray.** Unbonded atoms sit in a horizontal strip below the stage, as today. Pulling a new atom from the periodic-table-style palette still costs a move (unchanged move economy).

**Arming and bonding.** Clicking any atom — whether it's still in the tray or already placed on the stage — arms it (visual highlight). Clicking a second atom attempts to bond the two armed atoms, subject to the existing valence/capacity checks:
- Neither atom is placed yet → the first becomes a new fragment's anchor (seeded near stage center), the second seeds near it. Both move from tray to stage.
- One is placed, one isn't → the unplaced atom seeds near the placed one and joins the stage.
- Both are already placed → this is a ring-closure or fragment-join: a bond is added directly between their existing positions; the layout engine handles fitting the structure together on the next relaxation pass.

This single mechanic covers chaining, branching, ring-closing, and joining two independently-built fragments — all validated in playtesting during design.

**Bond order.** Clicking a bond line (anywhere along it, not just its midpoint — a wide invisible hit-target) opens a small floating popover anchored at its midpoint with `−`, the current order, `+`, and `×` (remove entirely). This replaces today's separate "bonds formed" chip list. `+` and `−` respect each atom's remaining bonding capacity the same way today's upgrade logic does; `−` below order 1, or `×`, removes the bond outright. Only one popover is open at a time; clicking elsewhere on the stage closes it.

**Removal.** Each atom (tray or stage) gets a small `×` control, same visual affordance as today's `c-atom-remove`. Removing a stage atom deletes it and every bond attached to it, then triggers a relayout. This is the primary "undo" mechanism — no separate global undo/step-back is built.

**Assemble.** An explicit button, as today (deliberately not auto-detected on match, so students can keep experimenting without an unexpected interrupt). On click, reuse the existing `getComponents()` connected-components scan and matching logic against the current docket target almost unchanged. Because the stage already **is** the live structural diagram, the separate end-of-round SVG-generation step (`moleculeSVG`/`layoutMolecule`) is removed entirely — on a match, the matched cluster flashes/confettis in place on the stage, then those atoms are removed and the next docket request appears. Non-matching feedback messages (wrong ratio, valence not yet filled, right molecule but not today's target, etc.) are unchanged, since that logic is independent of layout.

**No manual dragging.** Positioning is fully automatic (the layout engine). Decided against manual drag-to-nudge for v1 since the relaxation engine already handled every layout scenario tested — including deliberately awkward ones — without it. This can be revisited later if real classroom use turns up a case the auto-layout handles poorly.

## Known risks

- **Layout quality on more tangled structures.** The relaxation engine isn't guaranteed to find the prettiest arrangement for heavily fused/symmetric structures — this wasn't a problem for anything tested (single rings, branches, joined chains), which covers the current Stage 5 molecule set (methane, water, HCl, O₂, CO₂, N₂) comfortably. Risk goes up if the molecule set is ever extended well beyond that (e.g. benzene-scale rings or fused ring systems) — worth a dedicated testing pass if/when that happens. Some students actively enjoy building the weirdest structures they can within the current element set (H, C, N, O, Cl) — this is expected and useful for finding rough edges, not a case to design around up front.
- **Touch/small-screen hit targets.** The bond-order popover buttons are small SVG glyphs; will need testing on whatever hardware the class actually uses (Chromebooks/tablets) and likely need larger tap targets than the desktop-oriented prototype.
- **Performance.** Relaxation reruns on every graph change; not a concern at this game's atom counts (single digits to low teens per stage), so no optimization work is planned here.

## Testing approach

No formal automated test suite exists for this project today (single HTML file, no build tooling), and this design doesn't introduce one. Verification is manual/interactive: exercising the interaction flow (chain, branch, ring-close in both "natural" and "awkward" build orders, join independent fragments, upgrade/downgrade/remove bond orders, remove atoms mid-build, assemble on match and on near-miss) in-browser, consistent with how the rest of the project has been verified so far.
