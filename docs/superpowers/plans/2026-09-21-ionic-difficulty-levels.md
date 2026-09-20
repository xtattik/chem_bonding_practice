# Ionic Mode Difficulty Levels 2-6 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add five new ionic difficulty levels (transition metals with given/undisclosed oxidation state, polyatomic ions with bracket notation, common names) on top of the existing one, restructuring the flat challenge list into explicit per-level data so each level can carry its own hint/reveal rules.

**Architecture:** Single-file vanilla JS, no build step — matches the existing project exactly. All changes live in `index.html`'s ionic-mode section. Two new data tables (`TRANSITION_METALS`, `POLYATOMIC_IONS`) supply extra palette tiles rendered in a new strip below the periodic-table grid; the flat `CHALLENGES` array becomes a `LEVELS` array of per-level objects, each with its own hint text and reveal-mode flags, with `state.level` tracking progress. A workspace atom gains `charge`/`name`/`bg`/`fg` fields captured at spawn time (rather than re-derived from a shared per-symbol table), since transition metals need more than one charge per symbol — something the old `ELEMENTS[symbol].charge` lookup pattern couldn't express.

**Tech Stack:** Vanilla JS (matches existing style), CSS custom properties (existing design tokens). No test framework exists in this project and none is introduced — verification is manual, in-browser (open `index.html` directly; no dev server needed).

**Design reference:** `docs/superpowers/specs/2026-09-21-ionic-difficulty-levels-design.md`

---

## Before you start

- All edits are to the single file `index.html` at the project root. Ionic mode has not been touched by any prior work on this project (confirmed by reading the current file) — everything below targets the code exactly as it originally shipped.
- There is no test framework in this project. Verification is manual/interactive: open `index.html` directly in a browser (double-click it, or drag it into a browser window). No server required.
- Edits are specified as exact old-code → new-code blocks (find-and-replace targets), not line numbers, since line numbers shift as earlier steps land.
- **A note on scope:** this plan does not add a "next level" banner or a level-select menu. Clearing a level's challenges advances directly and seamlessly into the next level (a brief feedback message names the new level), exactly mirroring how the existing single-level game already flows from one compound to the next. If you want a level-select/jump control later, that's a small separate addition, not part of this plan.

---

### Task 1: Add transition-metal and polyatomic-ion data, and their palette strip

**Files:**
- Modify: `index.html`

This is a pure, additive change: two new data tables and a new (empty, hidden) UI section. Nothing reads or renders this data yet, so this task cannot change visible behavior.

- [ ] **Step 1: Add the CSS for the new ion-tile strip**

Find:

```css
  .palette-caption{margin:10px 2px 0;font-size:11.5px;color:var(--muted);max-width:60ch;}
```

Replace with:

```css
  .palette-caption{margin:10px 2px 0;font-size:11.5px;color:var(--muted);max-width:60ch;}

  .ion-row{display:flex;flex-wrap:wrap;gap:4px;}
  .ion-tile{
    position:relative;
    min-width:40px;height:36px;padding:0 8px;
    border:none;border-radius:5px;
    display:flex;flex-direction:column;align-items:center;justify-content:center;
    cursor:pointer;
    font-family:"IBM Plex Sans", sans-serif;
    box-shadow:var(--shadow);
    transition:transform .1s ease, opacity .1s ease;
  }
  .ion-tile:hover:not(:disabled){transform:translateY(-2px);}
  .ion-tile:disabled{opacity:.35;cursor:not-allowed;}
  .ion-tile .sym{font-weight:700;font-size:11px;line-height:1;}
  .ion-tile .charge{font-family:"IBM Plex Mono",monospace;font-size:8px;margin-top:1px;opacity:.9;}
```

- [ ] **Step 2: Add the ion-strip HTML markup**

Find:

```html
      <div class="palette-scroll">
        <div class="palette" id="palette"></div>
      </div>
      <p class="palette-caption" id="palette-caption"></p>
    </div>

  </section>

  <section id="mode-covalent" hidden>
```

Replace with:

```html
      <div class="palette-scroll">
        <div class="palette" id="palette"></div>
      </div>
      <p class="palette-caption" id="palette-caption"></p>
    </div>

    <div class="palette-section" id="ion-section" hidden>
      <div class="palette-head">
        <p class="chamber-label" style="margin:0;">Variable-charge metals &amp; common ions</p>
      </div>
      <div class="ion-row" id="ion-row"></div>
      <p class="palette-caption">Each tile's own charge is shown on the tile — these don't follow the periodic table's simple group-charge pattern.</p>
    </div>

  </section>

  <section id="mode-covalent" hidden>
```

- [ ] **Step 3: Add the `TRANSITION_METALS` and `POLYATOMIC_IONS` data tables**

Find:

```javascript
  const ORDER = ["H","He","Li","Be","B","C","N","O","F","Ne","Na","Mg","Al","Si","P","S","Cl","Ar","K","Ca"];

  const CHALLENGES = [
```

Replace with:

```javascript
  const ORDER = ["H","He","Li","Be","B","C","N","O","F","Ne","Na","Mg","Al","Si","P","S","Cl","Ar","K","Ca"];

  /* Transition metals with more than one commonly-taught oxidation state.
     Each gets its own tile (own key, own charge) but shares its real chemical
     symbol with its sibling states, so formula matching (which counts by
     symbol) treats "Cu" the same regardless of which charge-tile spawned it —
     exactly like real chemistry. minLevel is a 0-based index into LEVELS
     (declared further below): minLevel:1 means "first appears at LEVELS[1]",
     i.e. the level humans will see labelled "Level 2". */
  const TRANSITION_METALS = {
    Cu1: {symbol:"Cu", name:"Copper(I)",   charge:1, minLevel:1, bg:"#E8956B", fg:"#3A1B0A"},
    Cu2: {symbol:"Cu", name:"Copper(II)",  charge:2, minLevel:1, bg:"#C97442", fg:"#2A1206"},
    Fe2: {symbol:"Fe", name:"Iron(II)",    charge:2, minLevel:1, bg:"#B0B4B8", fg:"#22262A"},
    Fe3: {symbol:"Fe", name:"Iron(III)",   charge:3, minLevel:1, bg:"#888D92", fg:"#1A1D20"},
    Pb2: {symbol:"Pb", name:"Lead(II)",    charge:2, minLevel:1, bg:"#8B92A6", fg:"#1C202A"},
    Pb4: {symbol:"Pb", name:"Lead(IV)",    charge:4, minLevel:1, bg:"#666D80", fg:"#F0F2F5"},
    Sn2: {symbol:"Sn", name:"Tin(II)",     charge:2, minLevel:1, bg:"#D6D9DC", fg:"#282B2E"},
    Sn4: {symbol:"Sn", name:"Tin(IV)",     charge:4, minLevel:1, bg:"#AEB2B6", fg:"#1E2022"},
  };

  /* Polyatomic ions behave as a single charged particle for ionic bonding
     purposes — a new "fake element" with its own symbol/charge and no
     period/group position, rendered in the ion strip rather than on the
     periodic-table grid. */
  const POLYATOMIC_IONS = {
    OH:     {symbol:"OH",     name:"Hydroxide", charge:-1, minLevel:2, bg:"#8FD9C4", fg:"#0A2E24"},
    NO3:    {symbol:"NO3",    name:"Nitrate",   charge:-1, minLevel:2, bg:"#F2C572", fg:"#3A2703"},
    NH4:    {symbol:"NH4",    name:"Ammonium",  charge:1,  minLevel:2, bg:"#9FB8E8", fg:"#0D1E3A"},
    SO4:    {symbol:"SO4",    name:"Sulfate",   charge:-2, minLevel:3, bg:"#E8A0C0", fg:"#3A0F22"},
    CO3:    {symbol:"CO3",    name:"Carbonate", charge:-2, minLevel:3, bg:"#B8A8E0", fg:"#241A3A"},
    PO4:    {symbol:"PO4",    name:"Phosphate", charge:-3, minLevel:3, bg:"#F0967D", fg:"#3A1409"},
    CH3COO: {symbol:"CH3COO", name:"Acetate",   charge:-1, minLevel:3, bg:"#C4D96A", fg:"#242E08"},
  };

  const CHALLENGES = [
```

`minLevel` is 0-based to match `state.level` (added in Task 2) directly, avoiding off-by-one arithmetic at every use site. Level 3's ions (Sulfate/Carbonate/Phosphate/Acetate) get `minLevel:3` (LEVELS[3] = "Level 4") and Level 2's ions (Hydroxide/Nitrate/Ammonium) get `minLevel:2` (LEVELS[2] = "Level 3") — matching the design spec's Level 3/Level 4 content split.

- [ ] **Step 4: Smoke-check — page still loads cleanly**

Open `index.html` in a browser, Ionic tab. Open DevTools console.
Expected: no errors, page looks and behaves exactly as before (nothing calls or renders the new tables yet, and `#ion-section` stays hidden since nothing has removed its `hidden` attribute).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add transition-metal and polyatomic-ion data, and their palette strip (unused yet)"
```

---

### Task 2: Restructure into levels and wire up the new content

**Files:**
- Modify: `index.html`

This is the core change: the flat `CHALLENGES` array and its index-threshold hint logic are replaced by a `LEVELS` array (one object per level, each with its own hint text and all 6 levels' full content), `state` gains a `level` field, every function that reads `CHALLENGES`/`ELEMENTS[symbol].charge` is updated, and the new ion strip becomes live.

- [ ] **Step 1: Replace `CHALLENGES`/`TOTAL_MOVES` with the full `LEVELS` array**

Find:

```javascript
  const CHALLENGES = [
    {name:"Sodium Chloride",    formula:"NaCl",  elements:{Na:1, Cl:1}},
    {name:"Magnesium Chloride", formula:"MgCl₂", elements:{Mg:1, Cl:2}},
    {name:"Calcium Fluoride",   formula:"CaF₂",  elements:{Ca:1, F:2}},
    {name:"Potassium Sulfide",  formula:"K₂S",   elements:{K:2, S:1}},
    {name:"Aluminium Oxide",    formula:"Al₂O₃", elements:{Al:2, O:3}},
    {name:"Magnesium Nitride",  formula:"Mg₃N₂", elements:{Mg:3, N:2}},
  ];
  const TOTAL_MOVES = 31;
```

Replace with:

```javascript
  const TOTAL_MOVES = 31;

  /* Each level is a self-contained set of challenges plus its own hint and
     reveal rules. formulaMode: "first-only" (shown for the level's first
     request only, same as the original game's only behavior), "always"
     (Level 5 — the only way to deduce an undisclosed oxidation state),
     or "never" (Level 6 — no scaffolding at all). revealCharges only ever
     applies to Level 1's own already-known main-group tiles; transition-
     metal and polyatomic-ion tiles always show their own charge regardless,
     since the charge is what distinguishes one such tile from another. */
  const LEVELS = [
    {
      name: "Level 1: Main-Group Ions",
      formulaMode: "first-only",
      revealCharges: true,
      challenges: [
        {name:"Sodium Chloride",    formula:"NaCl",  elements:{Na:1, Cl:1}},
        {name:"Magnesium Chloride", formula:"MgCl₂", elements:{Mg:1, Cl:2}},
        {name:"Calcium Fluoride",   formula:"CaF₂",  elements:{Ca:1, F:2}},
        {name:"Potassium Sulfide",  formula:"K₂S",   elements:{K:2, S:1}},
        {name:"Aluminium Oxide",    formula:"Al₂O₃", elements:{Al:2, O:3}},
        {name:"Magnesium Nitride",  formula:"Mg₃N₂", elements:{Mg:3, N:2}},
      ],
    },
    {
      name: "Level 2: Transition Metals",
      formulaMode: "first-only",
      revealCharges: false,
      hint: "Copper, iron, lead and tin can form more than one ion — the compound's name tells you which charge to use via the Roman numeral.",
      challenges: [
        {name:"Copper(I) Oxide",   formula:"Cu₂O",  elements:{Cu:2, O:1}},
        {name:"Copper(II) Oxide",  formula:"CuO",   elements:{Cu:1, O:1}},
        {name:"Iron(II) Chloride", formula:"FeCl₂", elements:{Fe:1, Cl:2}},
        {name:"Iron(III) Chloride",formula:"FeCl₃", elements:{Fe:1, Cl:3}},
        {name:"Lead(II) Oxide",    formula:"PbO",   elements:{Pb:1, O:1}},
        {name:"Lead(IV) Oxide",    formula:"PbO₂",  elements:{Pb:1, O:2}},
        {name:"Tin(II) Chloride",  formula:"SnCl₂", elements:{Sn:1, Cl:2}},
        {name:"Tin(IV) Chloride",  formula:"SnCl₄", elements:{Sn:1, Cl:4}},
      ],
    },
    {
      name: "Level 3: Simple Polyatomic Ions",
      formulaMode: "first-only",
      revealCharges: false,
      hint: "Hydroxide, nitrate and ammonium act as a single charged particle, just like any other ion — treat the whole group as one unit.",
      challenges: [
        {name:"Sodium Hydroxide",   formula:"NaOH",  elements:{Na:1, OH:1}},
        {name:"Potassium Nitrate",  formula:"KNO₃",  elements:{K:1, NO3:1}},
        {name:"Ammonium Chloride",  formula:"NH₄Cl", elements:{NH4:1, Cl:1}},
        {name:"Lithium Hydroxide",  formula:"LiOH",  elements:{Li:1, OH:1}},
      ],
    },
    {
      name: "Level 4: More Polyatomic Ions",
      formulaMode: "first-only",
      revealCharges: false,
      hint: "Sulfate, carbonate, phosphate and acetate carry charges greater than 1 — when you need more than one of a polyatomic ion, its formula goes in brackets with the count outside, like Ca(OH)₂.",
      challenges: [
        {name:"Calcium Hydroxide",  formula:"Ca(OH)₂",     elements:{Ca:1, OH:2}},
        {name:"Magnesium Nitrate",  formula:"Mg(NO₃)₂",    elements:{Mg:1, NO3:2}},
        {name:"Aluminium Sulfate",  formula:"Al₂(SO₄)₃",   elements:{Al:2, SO4:3}},
        {name:"Ammonium Sulfate",   formula:"(NH₄)₂SO₄",   elements:{NH4:2, SO4:1}},
        {name:"Calcium Phosphate",  formula:"Ca₃(PO₄)₂",   elements:{Ca:3, PO4:2}},
        {name:"Calcium Acetate",    formula:"Ca(CH₃COO)₂", elements:{Ca:1, CH3COO:2}},
      ],
    },
    {
      name: "Level 5: Metals, Charge Unknown",
      formulaMode: "always",
      revealCharges: false,
      hint: "The name doesn't say which charge these metals are using this time — the formula shown does. Work out the metal's charge from the ratio, then match it.",
      challenges: [
        {name:"Copper Oxide", formula:"Cu₂O",  elements:{Cu:2, O:1}},
        {name:"Copper Oxide", formula:"CuO",   elements:{Cu:1, O:1}},
        {name:"Iron Chloride",formula:"FeCl₂", elements:{Fe:1, Cl:2}},
        {name:"Iron Chloride",formula:"FeCl₃", elements:{Fe:1, Cl:3}},
        {name:"Lead Oxide",   formula:"PbO",   elements:{Pb:1, O:1}},
        {name:"Lead Oxide",   formula:"PbO₂",  elements:{Pb:1, O:2}},
        {name:"Tin Chloride", formula:"SnCl₂", elements:{Sn:1, Cl:2}},
        {name:"Tin Chloride", formula:"SnCl₄", elements:{Sn:1, Cl:4}},
      ],
    },
    {
      name: "Level 6: Common Names",
      formulaMode: "never",
      revealCharges: false,
      hint: "These are the old common names chemists still use day-to-day — no Roman numeral, no formula. You'll need to know which charge each name refers to.",
      challenges: [
        {name:"Ferric Oxide",     formula:"Fe₂O₃", elements:{Fe:2, O:3}},
        {name:"Cuprous Chloride", formula:"CuCl",  elements:{Cu:1, Cl:1}},
        {name:"Stannic Oxide",    formula:"SnO₂",  elements:{Sn:1, O:2}},
        {name:"Plumbous Chloride",formula:"PbCl₂", elements:{Pb:1, Cl:2}},
      ],
    },
  ];
```

- [ ] **Step 2: Add `level` to state and reset it on a fresh game**

Find:

```javascript
  function freshState(){
    nextId = 1;
    return {
      movesLeft: TOTAL_MOVES,
      score: 0,
      challengeIndex: 0,
      workspace: [],
      selected: new Set(),
      won: false,
    };
  }

  function formulaVisible(){ return state.challengeIndex === 0; }
  function chargesVisible(){ return state.challengeIndex < 3; }
```

Replace with:

```javascript
  function freshState(){
    nextId = 1;
    return {
      level: 0,
      movesLeft: TOTAL_MOVES,
      score: 0,
      challengeIndex: 0,
      workspace: [],
      selected: new Set(),
      won: false,
    };
  }

  function formulaVisible(){
    const mode = LEVELS[state.level].formulaMode;
    if(mode === "always") return true;
    if(mode === "never") return false;
    return state.challengeIndex === 0;
  }
  function chargesVisible(){
    if(!LEVELS[state.level].revealCharges) return false;
    return state.challengeIndex < 3;
  }
```

- [ ] **Step 3: Give spawned atoms their own charge/name/color instead of deriving it from a shared table**

Find:

```javascript
  function spawnAtom(symbol){
    if(state.won) return;
    if(state.movesLeft<=0){ setFeedback("error","No moves left to pull new elements — bond what's already on the bench."); return; }
    state.movesLeft--;
    state.workspace.push({id: nextId++, symbol});
    render();
  }
```

Replace with:

```javascript
  function spawnAtom(entry){
    if(state.won) return;
    if(state.movesLeft<=0){ setFeedback("error","No moves left to pull new elements — bond what's already on the bench."); return; }
    state.movesLeft--;
    state.workspace.push({id: nextId++, symbol: entry.symbol, name: entry.name, charge: entry.charge, bg: entry.bg, fg: entry.fg});
    render();
  }
```

`entry` is a ready-made descriptor (`{symbol, name, charge, bg, fg}`) regardless of whether it came from `ELEMENTS`, `TRANSITION_METALS`, or `POLYATOMIC_IONS` — `spawnAtom` no longer needs to know which table a tile came from, and the resulting atom carries its own charge/color rather than requiring a later lookup keyed by symbol (which would break for transition metals, since two different charges can share one symbol).

- [ ] **Step 4: Update `attemptBond` to read charge/name from the atom itself, and reject mixed-charge selections of the same symbol**

Find:

```javascript
  function attemptBond(){
    if(state.selected.size===0){ setFeedback("neutral","Select some atoms from the bench first."); return; }
    const atoms = state.workspace.filter(a=>state.selected.has(a.id));
    const nonIonic = atoms.find(a=>ELEMENTS[a.symbol].charge===null);
    if(nonIonic){
      setFeedback("error", ELEMENTS[nonIonic.symbol].name + " doesn't form a simple ion — it can't take part in an ionic bond.");
      return;
    }
    const counts = {};
    atoms.forEach(a=> counts[a.symbol] = (counts[a.symbol]||0) + 1);
    const symbols = Object.keys(counts);
    if(symbols.length < 2){
      setFeedback("error","An ionic compound needs a cation and an anion — you've only selected one element.");
      return;
    }
    if(symbols.length > 2){
      setFeedback("error","Select atoms of exactly two elements — one cation, one anion.");
      return;
    }
    const netCharge = symbols.reduce((sum,s)=> sum + counts[s]*ELEMENTS[s].charge, 0);
    if(netCharge !== 0){
      setFeedback("error","Net charge is " + fmtCharge(netCharge) + " — adjust how many of each ion you've selected until it cancels to zero.");
      return;
    }
    const g = gcd(counts[symbols[0]], counts[symbols[1]]);
    const reduced = {};
    reduced[symbols[0]] = counts[symbols[0]]/g;
    reduced[symbols[1]] = counts[symbols[1]]/g;

    const target = CHALLENGES[state.challengeIndex].elements;
    const targetSymbols = Object.keys(target);
    const sameElements = targetSymbols.every(s=>reduced[s]!==undefined) && Object.keys(reduced).every(s=>target[s]!==undefined);
    const sameRatio = sameElements && targetSymbols.every(s=>reduced[s]===target[s]);

    if(sameElements && sameRatio){
      const ids = atoms.map(a=>a.id);
      ids.forEach(id=>{
        const tile = document.querySelector('[data-atom-id="'+id+'"]');
        if(tile) tile.classList.add("leaving");
      });
      const compoundName = CHALLENGES[state.challengeIndex].name;
      setTimeout(()=>{
        state.workspace = state.workspace.filter(a=>!ids.includes(a.id));
        state.selected.clear();
        state.score++;
        render();
        celebrate(compoundName);
      }, 160);
      return;
    }
    if(sameElements && !sameRatio){
      setFeedback("error","Charges balance, but that's the wrong ratio for " + CHALLENGES[state.challengeIndex].name + " — check the formula and try again.");
      return;
    }
    setFeedback("neutral","That combination is charge-balanced, just not what's on today's docket (" + CHALLENGES[state.challengeIndex].name + "). Try the right elements.");
  }
```

Replace with:

```javascript
  function attemptBond(){
    if(state.selected.size===0){ setFeedback("neutral","Select some atoms from the bench first."); return; }
    const atoms = state.workspace.filter(a=>state.selected.has(a.id));
    const nonIonic = atoms.find(a=>a.charge===null);
    if(nonIonic){
      setFeedback("error", nonIonic.name + " doesn't form a simple ion — it can't take part in an ionic bond.");
      return;
    }
    const chargesBySymbol = {};
    atoms.forEach(a=>{ (chargesBySymbol[a.symbol] = chargesBySymbol[a.symbol] || new Set()).add(a.charge); });
    const mixedSymbol = Object.keys(chargesBySymbol).find(sym => chargesBySymbol[sym].size > 1);
    if(mixedSymbol){
      setFeedback("error", "You've selected " + mixedSymbol + " atoms with different charges — pick a single charge state and use only that one.");
      return;
    }
    const counts = {};
    atoms.forEach(a=> counts[a.symbol] = (counts[a.symbol]||0) + 1);
    const symbols = Object.keys(counts);
    if(symbols.length < 2){
      setFeedback("error","An ionic compound needs a cation and an anion — you've only selected one element.");
      return;
    }
    if(symbols.length > 2){
      setFeedback("error","Select atoms of exactly two elements — one cation, one anion.");
      return;
    }
    const netCharge = atoms.reduce((sum,a)=> sum + a.charge, 0);
    if(netCharge !== 0){
      setFeedback("error","Net charge is " + fmtCharge(netCharge) + " — adjust how many of each ion you've selected until it cancels to zero.");
      return;
    }
    const g = gcd(counts[symbols[0]], counts[symbols[1]]);
    const reduced = {};
    reduced[symbols[0]] = counts[symbols[0]]/g;
    reduced[symbols[1]] = counts[symbols[1]]/g;

    const level = LEVELS[state.level];
    const challenge = level.challenges[state.challengeIndex];
    const target = challenge.elements;
    const targetSymbols = Object.keys(target);
    const sameElements = targetSymbols.every(s=>reduced[s]!==undefined) && Object.keys(reduced).every(s=>target[s]!==undefined);
    const sameRatio = sameElements && targetSymbols.every(s=>reduced[s]===target[s]);

    if(sameElements && sameRatio){
      const ids = atoms.map(a=>a.id);
      ids.forEach(id=>{
        const tile = document.querySelector('[data-atom-id="'+id+'"]');
        if(tile) tile.classList.add("leaving");
      });
      const compoundName = challenge.name;
      setTimeout(()=>{
        state.workspace = state.workspace.filter(a=>!ids.includes(a.id));
        state.selected.clear();
        state.score++;
        render();
        celebrate(compoundName);
      }, 160);
      return;
    }
    if(sameElements && !sameRatio){
      setFeedback("error","Charges balance, but that's the wrong ratio for " + challenge.name + " — check the formula and try again.");
      return;
    }
    setFeedback("neutral","That combination is charge-balanced, just not what's on today's docket (" + challenge.name + "). Try the right elements.");
  }
```

The mixed-charge guard is new and necessary: with more than one charge sharing a symbol (e.g. Cu⁺ and Cu²⁺ both have `symbol:"Cu"`), the existing ratio-matching logic only ever counted atoms *by symbol*, never checking that all atoms of that symbol actually share the same charge. Without this guard, a student could in principle mix charge-states of the same metal and have the count-based ratio check accept it as if it were a single, chemically consistent selection.

- [ ] **Step 5: Update `celebrate` to advance to the next level once a level's challenges are exhausted**

Find:

```javascript
  function celebrate(compoundName){
    const shell = document.getElementById("i-shell");
    shell.classList.add("flash-success");
    setTimeout(()=> shell.classList.remove("flash-success"), 700);
    setFeedback("celebrate", "✓ " + compoundName + " bonded — next request coming up.");
    spawnConfetti("i-shell");
    setTimeout(()=>{
      state.challengeIndex++;
      if(state.challengeIndex >= CHALLENGES.length){
        state.won = true;
      }
      render();
      if(!state.won) setFeedback("neutral","Pick a cation and an anion from the periodic table below — each pull costs one move.");
    }, 1000);
  }
```

Replace with:

```javascript
  function celebrate(compoundName){
    const shell = document.getElementById("i-shell");
    shell.classList.add("flash-success");
    setTimeout(()=> shell.classList.remove("flash-success"), 700);
    setFeedback("celebrate", "✓ " + compoundName + " bonded — next request coming up.");
    spawnConfetti("i-shell");
    setTimeout(()=>{
      state.challengeIndex++;
      let leveledUp = false;
      if(state.challengeIndex >= LEVELS[state.level].challenges.length){
        if(state.level >= LEVELS.length-1){
          state.won = true;
        } else {
          state.level++;
          state.challengeIndex = 0;
          state.score = 0;
          state.movesLeft = TOTAL_MOVES;
          state.workspace = [];
          state.selected.clear();
          leveledUp = true;
        }
      }
      render();
      if(!state.won){
        setFeedback("neutral", leveledUp
          ? LEVELS[state.level].name + " — " + LEVELS[state.level].hint
          : "Pick a cation and an anion from the periodic table below — each pull costs one move.");
      }
    }, 1000);
  }
```

Score, moves, and the bench all reset at each level boundary — each level is its own self-contained set, matching how the design spec frames levels as independent containers.

- [ ] **Step 6: Update `renderDocket` for per-level naming, hints, and the final win banner**

Find:

```javascript
  function renderDocket(){
    const slot = document.getElementById("docket-slot");
    if(state.won){
      slot.innerHTML =
        '<div class="banner win">' +
          '<h2>Docket cleared</h2>' +
          '<p>All ' + CHALLENGES.length + ' compounds bonded, with ' + state.movesLeft + ' of ' + TOTAL_MOVES + ' moves left over.</p>' +
          '<button class="btn primary" id="btn-play-again">Run it again</button>' +
        '</div>';
      document.getElementById("btn-play-again").addEventListener("click", resetGame);
      return;
    }
    const c = CHALLENGES[state.challengeIndex];
    const hint = formulaVisible()
      ? "Formula's given for this first one — after this you'll work from the name alone."
      : (chargesVisible()
        ? "Build it from ions balanced to zero net charge, in this compound's real ratio."
        : "Ion charges are hidden now too — recall them from each element's group on the table below.");
    slot.innerHTML =
      '<div class="docket">' +
        '<div class="docket-top">' +
          '<span class="docket-label">Request No. ' + (state.challengeIndex+1) + ' / ' + CHALLENGES.length + '</span>' +
        '</div>' +
        '<p class="docket-name">' + c.name + '</p>' +
        (formulaVisible() ? '<span class="docket-formula">' + c.formula + '</span>' : '') +
        '<p class="docket-hint">' + hint + '</p>' +
      '</div>';
  }
```

Replace with:

```javascript
  function renderDocket(){
    const slot = document.getElementById("docket-slot");
    if(state.won){
      const totalCompounds = LEVELS.reduce((s,l)=>s+l.challenges.length, 0);
      slot.innerHTML =
        '<div class="banner win">' +
          '<h2>Docket cleared</h2>' +
          '<p>All ' + totalCompounds + ' compounds across all ' + LEVELS.length + ' levels bonded, with ' + state.movesLeft + ' of ' + TOTAL_MOVES + ' moves left in the final level.</p>' +
          '<button class="btn primary" id="btn-play-again">Run it again</button>' +
        '</div>';
      document.getElementById("btn-play-again").addEventListener("click", resetGame);
      return;
    }
    const level = LEVELS[state.level];
    const c = level.challenges[state.challengeIndex];
    const hint = state.level===0
      ? (formulaVisible()
        ? "Formula's given for this first one — after this you'll work from the name alone."
        : (chargesVisible()
          ? "Build it from ions balanced to zero net charge, in this compound's real ratio."
          : "Ion charges are hidden now too — recall them from each element's group on the table below."))
      : level.hint;
    slot.innerHTML =
      '<div class="docket">' +
        '<div class="docket-top">' +
          '<span class="docket-label">' + level.name + ' &middot; Request No. ' + (state.challengeIndex+1) + ' / ' + level.challenges.length + '</span>' +
        '</div>' +
        '<p class="docket-name">' + c.name + '</p>' +
        (formulaVisible() ? '<span class="docket-formula">' + c.formula + '</span>' : '') +
        '<p class="docket-hint">' + hint + '</p>' +
      '</div>';
  }
```

- [ ] **Step 7: Update `renderReadouts` to read charge from the atom and score against the current level's challenge count**

Find:

```javascript
  function renderReadouts(){
    document.getElementById("read-moves").textContent = state.movesLeft + " / " + TOTAL_MOVES;
    document.getElementById("read-score").textContent = state.score + " / " + CHALLENGES.length;
    const atoms = state.workspace.filter(a=>state.selected.has(a.id));
    if(atoms.length===0){
      document.getElementById("read-charge").textContent = "—";
    } else {
      const sum = atoms.reduce((s,a)=> s + (ELEMENTS[a.symbol].charge||0), 0);
      document.getElementById("read-charge").textContent = fmtCharge(sum);
    }
  }
```

Replace with:

```javascript
  function renderReadouts(){
    document.getElementById("read-moves").textContent = state.movesLeft + " / " + TOTAL_MOVES;
    document.getElementById("read-score").textContent = state.score + " / " + LEVELS[state.level].challenges.length;
    const atoms = state.workspace.filter(a=>state.selected.has(a.id));
    if(atoms.length===0){
      document.getElementById("read-charge").textContent = "—";
    } else {
      const sum = atoms.reduce((s,a)=> s + (a.charge||0), 0);
      document.getElementById("read-charge").textContent = fmtCharge(sum);
    }
  }
```

- [ ] **Step 8: Update `renderChamber` to read color/name from the atom itself**

Find:

```javascript
  function renderChamber(){
    const chamber = document.getElementById("chamber");
    chamber.innerHTML = "";
    if(state.workspace.length===0){
      const p = document.createElement("p");
      p.className = "chamber-empty";
      p.textContent = "Bench is empty — pull elements from the supply below.";
      chamber.appendChild(p);
      return;
    }
    state.workspace.forEach(a=>{
      const el = ELEMENTS[a.symbol];
      const btn = document.createElement("button");
      btn.className = "atom" + (state.selected.has(a.id) ? " selected" : "");
      btn.style.background = el.bg;
      btn.style.color = el.fg;
      btn.dataset.atomId = a.id;
      btn.title = el.name;
      btn.textContent = a.symbol;
      btn.addEventListener("click", ()=>toggleSelect(a.id));
      chamber.appendChild(btn);
    });
  }
```

Replace with:

```javascript
  function renderChamber(){
    const chamber = document.getElementById("chamber");
    chamber.innerHTML = "";
    if(state.workspace.length===0){
      const p = document.createElement("p");
      p.className = "chamber-empty";
      p.textContent = "Bench is empty — pull elements from the supply below.";
      chamber.appendChild(p);
      return;
    }
    state.workspace.forEach(a=>{
      const btn = document.createElement("button");
      btn.className = "atom" + (state.selected.has(a.id) ? " selected" : "");
      btn.style.background = a.bg;
      btn.style.color = a.fg;
      btn.dataset.atomId = a.id;
      btn.title = a.name;
      btn.textContent = a.symbol;
      btn.addEventListener("click", ()=>toggleSelect(a.id));
      chamber.appendChild(btn);
    });
  }
```

- [ ] **Step 9: Update `renderPalette`'s spawn call to build a full entry, and add `renderIonRow`**

Find:

```javascript
  function renderPalette(){
    const palette = document.getElementById("palette");
    palette.innerHTML = "";
    ORDER.forEach(sym=>{
      const el = ELEMENTS[sym];
      const btn = document.createElement("button");
      btn.className = "cell";
      btn.style.gridColumn = el.group;
      btn.style.gridRow = el.period;
      btn.style.background = el.bg;
      btn.style.color = el.fg;
      btn.title = el.name + " — pulling costs 1 move";
      btn.disabled = state.movesLeft<=0 || state.won;
      btn.innerHTML =
        '<span class="z">' + el.z + '</span>' +
        '<span class="sym">' + sym + '</span>' +
        (chargesVisible() && el.charge!==null ? '<span class="charge">' + fmtCharge(el.charge) + '</span>' : '');
      btn.addEventListener("click", ()=>spawnAtom(sym));
      palette.appendChild(btn);
    });
    document.getElementById("palette-caption").textContent = chargesVisible()
      ? "Carbon, silicon, boron and the noble gases sit on the shelf as distractors — they don't form simple ions at this tier, same as an unused reagent bottle."
      : "Charges are hidden from here on — read them off each element's group instead of the tile.";
  }
```

Replace with:

```javascript
  function renderPalette(){
    const palette = document.getElementById("palette");
    palette.innerHTML = "";
    ORDER.forEach(sym=>{
      const el = ELEMENTS[sym];
      const btn = document.createElement("button");
      btn.className = "cell";
      btn.style.gridColumn = el.group;
      btn.style.gridRow = el.period;
      btn.style.background = el.bg;
      btn.style.color = el.fg;
      btn.title = el.name + " — pulling costs 1 move";
      btn.disabled = state.movesLeft<=0 || state.won;
      btn.innerHTML =
        '<span class="z">' + el.z + '</span>' +
        '<span class="sym">' + sym + '</span>' +
        (chargesVisible() && el.charge!==null ? '<span class="charge">' + fmtCharge(el.charge) + '</span>' : '');
      btn.addEventListener("click", ()=>spawnAtom({symbol:sym, name:el.name, charge:el.charge, bg:el.bg, fg:el.fg}));
      palette.appendChild(btn);
    });
    document.getElementById("palette-caption").textContent = chargesVisible()
      ? "Carbon, silicon, boron and the noble gases sit on the shelf as distractors — they don't form simple ions at this tier, same as an unused reagent bottle."
      : "Charges are hidden from here on — read them off each element's group instead of the tile.";
  }

  function renderIonRow(){
    const section = document.getElementById("ion-section");
    const row = document.getElementById("ion-row");
    const items = [];
    Object.keys(TRANSITION_METALS).forEach(key=>{
      const e = TRANSITION_METALS[key];
      if(e.minLevel <= state.level) items.push(e);
    });
    Object.keys(POLYATOMIC_IONS).forEach(key=>{
      const e = POLYATOMIC_IONS[key];
      if(e.minLevel <= state.level) items.push(e);
    });
    section.hidden = items.length===0;
    row.innerHTML = "";
    items.forEach(e=>{
      const btn = document.createElement("button");
      btn.className = "ion-tile";
      btn.style.background = e.bg;
      btn.style.color = e.fg;
      btn.title = e.name + " — pulling costs 1 move";
      btn.disabled = state.movesLeft<=0 || state.won;
      btn.innerHTML =
        '<span class="sym">' + e.symbol + '</span>' +
        '<span class="charge">' + fmtCharge(e.charge) + '</span>';
      btn.addEventListener("click", ()=>spawnAtom(e));
      row.appendChild(btn);
    });
  }
```

`renderIonRow` mirrors `renderPalette`'s existing pattern exactly (build tiles, disable when out of moves or won, spawn on click) — the only real difference is filtering by `minLevel` and hiding the whole section when nothing qualifies yet.

- [ ] **Step 10: Wire `renderIonRow` into the main render loop**

Find:

```javascript
  function render(){
    renderDocket();
    renderReadouts();
    renderChamber();
    renderPalette();
    document.getElementById("btn-bond").disabled = state.selected.size===0 || state.won;
    document.getElementById("btn-select-all").disabled = state.workspace.length===0;
    document.getElementById("btn-discard").disabled = state.selected.size===0;
  }
```

Replace with:

```javascript
  function render(){
    renderDocket();
    renderReadouts();
    renderChamber();
    renderPalette();
    renderIonRow();
    document.getElementById("btn-bond").disabled = state.selected.size===0 || state.won;
    document.getElementById("btn-select-all").disabled = state.workspace.length===0;
    document.getElementById("btn-discard").disabled = state.selected.size===0;
  }
```

- [ ] **Step 11: Verify — Level 1 behaves exactly as before**

Open `index.html`, Ionic tab (default). Confirm the docket now reads "Level 1: Main-Group Ions · Request No. 1 / 6" (new level prefix, same numbers as before). Build Sodium Chloride (Na + Cl) exactly as the existing game always allowed. Confirm no console errors, and confirm `#ion-section` stays hidden throughout Level 1 (no transition-metal/ion tiles appear yet, since their `minLevel` is 1 or higher and `state.level` is 0).

- [ ] **Step 12: Verify — leveling up into Level 2 reveals the transition-metal tiles**

Continue building all 6 Level 1 compounds. On the 6th success, confirm: feedback names "Level 2: Transition Metals" plus its hint; moves/score reset to 31/0; the docket shows "Level 2: Transition Metals · Request No. 1 / 8"; the `#ion-section` becomes visible with 8 tiles (Cu⁺, Cu²⁺, Fe²⁺, Fe³⁺, Pb²⁺, Pb⁴⁺, Sn²⁺, Sn⁴⁺), each showing its own charge.

- [ ] **Step 13: Verify — the mixed-charge guard**

Still in Level 2: spawn one Cu⁺ tile and one Cu²⁺ tile, select both, click "Bond selected". Expected: error feedback "You've selected Cu atoms with different charges — pick a single charge state and use only that one." — not a false-positive match, not a crash.

- [ ] **Step 14: Verify — build one Level 2 compound correctly**

Discard the mixed selection (free). Spawn 2× Cu⁺ and 1× O (from the main periodic table), select all three, click "Bond selected". Expected: matches "Copper(I) Oxide", flashes/clears, feedback congratulates, docket advances to the next Level 2 request.

- [ ] **Step 15: Commit**

```bash
git add index.html
git commit -m "Restructure ionic mode into 6 explicit difficulty levels"
```

---

### Task 3: Full playthrough verification and polish

**Files:**
- Modify: `index.html` (only if a genuine bug is found)

- [ ] **Step 1: Full sequential playthrough, Levels 1 through 6**

Open `index.html` fresh (Reset if needed). Play through every level in order, building every compound listed in each level's `challenges` array (refer to Task 2 Step 1's `LEVELS` data for the exact list and formulas). For each level, confirm:
- The correct tiles are available (no higher-level tiles leak in early — e.g. confirm sulfate/carbonate/phosphate/acetate tiles are absent until Level 4, per their `minLevel:3`).
- The hint text and formula-reveal behavior matches the design spec's table (Level 1: formula shown first request only, charges shown then hidden after 3; Level 2: formula shown first request only, transition-metal charges always on-tile; Level 3-4: formula shown first request only; Level 5: formula **always** shown; Level 6: formula **never** shown).
- Every compound in the level's list can actually be built and correctly matches (this exercises every polyatomic ion, every transition-metal charge pairing, and the bracket-notation compounds like `Al₂(SO₄)₃` and `(NH₄)₂SO₄`).
- Level 5's docket names omit the Roman numeral (e.g. "Copper Oxide", not "Copper(II) Oxide") while still showing the formula.
- Level 6's docket names use the common name (Ferric/Cuprous/Stannic/Plumbous) with no formula shown at all.

On completing Level 6's last compound, confirm the final "Docket cleared" banner shows the correct total compound count (36 — sum of 6+8+4+6+8+4) and correct level count (6), and "Run it again" resets fully back to Level 1 with `#ion-section` hidden again.

If any compound fails to match when built correctly, or any tile appears at the wrong level, treat it as a real bug: investigate against the `LEVELS` data written in Task 2 and fix it there (a data typo is the most likely cause — double-check the `elements` counts against the formula shown alongside it).

- [ ] **Step 2: Dark mode and reduced motion spot check**

Toggle dark mode (OS-level or browser devtools `prefers-color-scheme: dark` emulation) and reload. Confirm the new `.ion-tile` elements and their charge badges remain legible — they use inline `background`/`color` styles set directly from each entry's `bg`/`fg` hex values (not CSS custom properties), matching exactly how the existing periodic-table `.cell` tiles already work, so no new dark-mode-specific styling is needed — just confirm this holds by looking at it.

Toggle `prefers-reduced-motion: reduce` and reload. Confirm ionic mode's existing confetti-suppression (`spawnConfetti`'s pre-existing guard) still applies on a Level 2+ celebration exactly as it did before this feature (unaffected by this plan's changes, but worth confirming nothing regressed).

- [ ] **Step 3: Final commit (only if Step 1 or 2 required a fix)**

```bash
git add index.html
git commit -m "<describe the fix>"
```

If no bugs were found, there's nothing to commit — the feature is complete as of Task 2's commit.

---

## Self-review notes

- **Spec coverage:** all 6 levels' content and reveal rules (design spec's "Level ladder" and "Content lists" tables) are fully represented in Task 2 Step 1's `LEVELS` data. The data-model changes described in the spec (workspace atom carrying its own charge, polyatomic ions as fake elements, level-gated tiles via `minLevel`) are implemented in Task 1 (data/UI) and Task 2 (wiring). The mixed-charge guard was not explicitly called out in the spec but is a direct, necessary consequence of the spec's own transition-metal design (multiple charges sharing one symbol) — addressed in Task 2 Step 4 rather than left as a latent bug.
- **Type/naming consistency:** `LEVELS`, `TRANSITION_METALS`, `POLYATOMIC_IONS`, `state.level`, `renderIonRow`, `spawnAtom(entry)` are each defined once and referenced identically everywhere they're used across all three tasks — no renamed duplicates.
- **No placeholders:** every step contains complete, exact code and real content data (all 36 compounds' names/formulas/element-counts are fully written out) — no TBD/TODO markers, no "similar to Task N" references.
- **Out of scope, by design (per the approved spec):** covalent mode's difficulty ladder, cross-mode unlock/lock gating, replayable random-draw banks, and the high-oxidation-state popup selector are untouched by this plan.
