# Covalent Mode: Coordinate Bonds & Polyatomic Ion Builder — Design

Date: 2026-09-21
Status: Approved, ready for implementation planning

## Background

Covalent mode currently supports only neutral, fully-valence-filled molecules — every atom's bonds must exactly equal its normal capacity, and the game has no concept of charge. This is the first of several planned covalent difficulty tiers beyond the existing intro level (simple molecules + single/double/triple bonds); see the broader ladder discussed alongside this design (alkanes/alkenes, a structural/positional-matching engine, haloalkanes/alcohols/carboxylic acids/rings) — those are separate sub-projects, not part of this one.

This design adds coordinate (dative) covalent bonding and, building directly on it, a set of polyatomic-ion "build" challenges (ammonium, hydronium, hydroxide, nitrate, sulfate, carbonate, phosphate, acetate) — the same ions ionic mode's Level 3/4 already use as pre-formed tiles. The intent (per earlier discussion, not part of this design) is that completing this eventually unlocks those tiles in ionic mode; for now, both remain independently available, with no cross-mode gating built.

## Goals

- Teach coordinate bonding (an atom donating both electrons of a shared pair, rather than one each) as a direct, minimal extension of the existing bond-forming mechanic.
- Teach that a covalent structure can carry a net charge, and that the charge is a direct, visible consequence of an atom having more or fewer bonds than its normal capacity — not a separate concept to memorize per ion.
- Produce the same 7 polyatomic ions ionic mode already uses (plus hydronium, a natural second worked example), each correctly matched on both composition and bond-order pattern, with per-ion guidance on which atom(s) carry the charge and why.

## Non-goals (explicitly out of scope for this pass)

- Cross-mode unlocking/gating between this level and ionic mode's polyatomic ion tiles — deferred, as previously agreed. Both stay independently available.
- The structural/positional-matching engine needed for haloalkane/alcohol position isomers and ring isomers — a separate sub-project; this design's target-matching approach (sorted per-atom bond-order lists per symbol) is deliberately simpler and doesn't require full graph comparison, since none of this level's ions have positional ambiguity.
- Alkanes/alkenes content — a separate, simpler sub-project with no new mechanic.
- Coordinate double/triple bonds — none of this level's targets need one; coordinate bonds are scoped to order 1 only.
- A popup/breakout selector for high-oxidation-state donors — not needed here; this level's donors (N, O) each only ever donate once.

## The mechanic: charge as capacity deviation

Every covalent atom already has a `capacity` (its normal maximum total bond order) and a computed `usedValence` (its actual total bond order in the current structure) — this is how the existing engine already blocks over-bonding. The entire new concept is:

**An atom's formal charge = usedValence − capacity.** A molecule's net charge is the sum of this value across all its atoms.

This is mathematically equivalent to the standard formal-charge rule for octet-following atoms (verified against nitrate's real Lewis structure: nitrogen with a double bond and two single bonds has usedValence 4 against a normal capacity of 3, giving +1; the two singly-bonded oxygens each have usedValence 1 against capacity 2, giving −1 each; the doubly-bonded oxygen is neutral; net −1, matching the real ion). No separate "this bond is coordinate" flag is needed for charge computation — it falls directly out of the same numbers the engine already tracks.

Mechanically, this requires relaxing two existing hard rules, both currently absolute:
1. **Bonding never allows an atom past its capacity.** This must now be allowed by exactly +1, and only for donor-capable atoms (nitrogen and oxygen, the two donors this ion set needs — not hydrogen or carbon, which have no lone pair to donate).
2. **A molecule is only "complete" when every atom's `usedValence` exactly equals its capacity.** This must now accept a fragment where specific atoms are deliberately under capacity, per what the current target actually requires.

## Coordinate bonds: forming one, and how they look

To form a coordinate bond: arm an atom already at capacity (e.g. nitrogen in a built ammonia, 3/3), then click a **fresh, unbonded** atom. Today this combination is a hard-blocked error ("already has every shared pair it can take"); it becomes allowed specifically when the donor is nitrogen or oxygen and the result is exactly +1 over that atom's capacity — the acceptor atom must be freshly spawned with zero existing bonds (it's contributing no electrons of its own, so it can't already be bonded elsewhere).

Coordinate bonds are visually distinguished from ordinary bonds with an arrowhead pointing from donor to acceptor, matching the standard dative-bond textbook notation (rather than the plain line used for a normal shared-pair bond) — so a student can see at a glance which bond in a structure is the "special" one, and the visual matches what they'll see in a textbook or exam. They're limited to bond order 1 — no target in this level needs a coordinate double bond, so upgrading one isn't supported.

## Target matching: bond-order pattern, not just atom count

Symmetric ions like nitrate (three oxygens, but only one carries a double bond) can't be verified by atom-counting alone — two different, equally-valid builds could have the same atom counts but a wrong bond pattern. The fix: for each symbol in a target, store the **sorted list of expected per-atom `usedValence` values**, not just a count. Nitrate's oxygen entry becomes `[2,1,1]` rather than a plain count of 3. A built fragment matches a target if, for every symbol, the sorted list of its atoms' actual `usedValence` values equals the target's list — order-independent, so it doesn't matter *which* of the three oxygens ends up double-bonded, only that the pattern (one 2, two 1s) is right. This is a direct generalization of the existing atom-counting approach, not a new class of algorithm, and doesn't require comparing full connectivity graphs (unlike the deferred structural-matching problem for chain positional isomers). Net charge is checked the same way: the sum of a built fragment's per-atom deviations must equal the target's specified charge.

## Level content and flow

**Request 1 (tutorial, scripted start):** Stage opens with ammonia (NH₃) already built — three N–H bonds, nitrogen at capacity. The docket explains directly that nitrogen still has an unused lone pair despite "looking full," and walks the student through bonding a fourth H to it, naming the result (coordinate/dative bond) and showing the live charge readout move to +1 as they do it. Success message reinforces: "Ammonium, NH₄⁺ — the '+' means nitrogen ended up with one more bond than usual."

**Request 2 (Hydronium, H₃O⁺):** Same pattern with a different donor (oxygen, from a pre-built water), reinforcing that the rule is general rather than nitrogen-specific.

**Requests 3+ (the anions — hydroxide, nitrate, sulfate, carbonate, phosphate, acetate):** A level-transition hint flags the direction flip explicitly ("now the opposite: an atom can also end up with *fewer* bonds than normal — same rule, opposite sign"). Ordered roughly by complexity: hydroxide first (a single oxygen, one bond, simplest possible case), then nitrate/carbonate/sulfate/phosphate (each needs the one-double-bond-among-several-single-bonds pattern), then acetate last (combines a full neutral methyl/carbon skeleton with the charged carboxylate end — the most compound structure in the set). Each ion gets its own accurate hint naming which atom(s) carry the charge and why — not a generic reminder, and not oversimplified where the real structure needs more than one deviating atom (e.g. nitrate's hint correctly describes two oxygens involved, not one).

**Charge display:** the existing "Bonds formed" readout area gains a live "Charge" number while building (mirroring ionic mode's existing selection-charge readout), and an assembled ion's name carries its charge going forward (e.g. "Ammonium NH₄⁺") in the success feedback.

## Testing approach

No automated test suite exists in this project and this design doesn't introduce one, consistent with every prior feature on this project. Verification is manual/interactive: build each of the 8 targets (ammonium, hydronium, hydroxide, nitrate, sulfate, carbonate, phosphate, acetate), confirming correct bond-order pattern matching, correct live and final charge display, correct visual distinction for coordinate bonds, and correct blocking (donors that shouldn't be able to exceed capacity — e.g. carbon or hydrogen — still can't).
