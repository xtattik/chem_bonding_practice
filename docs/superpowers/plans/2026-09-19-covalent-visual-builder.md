# Covalent Bonding Visual Stage Builder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rework Bond Builder's covalent mode so the molecule's structure is visible and grows live on a "stage" as the student bonds atoms — click-to-arm/click-to-bond (not drag), with a spring-relaxation layout engine that self-organizes the diagram regardless of build order, plus a per-bond popover for setting bond order and per-atom removal for undo.

**Architecture:** Single-file vanilla JS, no build step, no dependencies — matches the existing project exactly. All changes live in `index.html`'s covalent-mode IIFE section. A new pure layout-engine block (no DOM) computes atom positions by relaxing a spring simulation each time the bond graph changes; the existing bonding/matching logic (`getComponents`, `symbolCounts`, `isComplete`, `countsEqual`) is reused unchanged. The old flat "bench + bonds-formed chip list + end-of-round reveal" rendering is replaced by an SVG "stage" (live diagram) plus a small "tray" (unplaced atoms).

**Tech Stack:** Vanilla JS (ES5-ish, matches existing style), inline SVG, CSS custom properties (existing design tokens), `requestAnimationFrame`. No test framework exists in this project and none is introduced — verification is manual, in-browser (open `index.html` directly; no dev server needed).

**Design reference:** `docs/superpowers/specs/2026-09-19-covalent-visual-builder-design.md`

---

## Before you start

- All edits are to the single file `index.html` at the project root.
- The entire script is one IIFE (`(function(){ ... })();`). There is no way to inspect `cState` or covalent-mode functions from the DevTools console — they're closure-private. Verification therefore relies on **DOM state** (attribute values, rendered text) or **visual/interactive checks**, never on typing variable names into the console.
- Because of this, Tasks 1–3 make changes that aren't yet visible in the UI (the old rendering ignores the new data until Task 4 replaces it). Their "Verify" steps are deliberately lighter (regression smoke-checks); the deep interactive verification happens in Task 4 onward, once the new rendering exists to observe.
- Edits are specified as exact old-code → new-code blocks (not line numbers) since line numbers shift as earlier tasks land. Use these blocks as literal find-and-replace targets.
- To view the page at any point: open `C:\Code Projects\chem_bonding_practice\index.html` directly in a browser (double-click it, or drag it into a browser window). No server required.

---

### Task 1: Layout engine (pure spring relaxation)

**Files:**
- Modify: `index.html` (covalent-mode script section)
- Test: none (no test framework in this project) — manual smoke-check only, see Step 3

This adds a standalone, DOM-free block of functions that computes atom positions from the bond graph. Nothing calls it yet, so this task cannot change visible behavior — it's pure addition.

- [ ] **Step 1: Locate the insertion point**

Find this exact block (it's near the top of the covalent-mode section, right after `TOTAL_C_MOVES` is declared):

```javascript
  const TOTAL_C_MOVES = 30;

  let cState, cNextId, cNextBondId;
```

- [ ] **Step 2: Insert the layout engine between those two lines**

Replace the block above with:

```javascript
  const TOTAL_C_MOVES = 30;

  /* ---------- layout engine (pure — no DOM) ----------
     Positions atoms by relaxing a simple spring simulation every time the
     bond graph changes, so the final shape depends only on which atoms are
     bonded to which, never on the order they were clicked in. */
  const STAGE_W = 480, STAGE_H = 260;
  const BOND_REST_LEN = 62, REPEL_K = 9000, SPRING_K = 0.10, CENTER_K = 0.004, MAX_STEP = 5;
  const RELAX_MIN_TICKS = 20, RELAX_MAX_TICKS = 200, RELAX_STOP_THRESHOLD = 0.05;

  function seedPosition(newAtomId, anchorId){
    const anchor = cState.workspace.find(a=>a.id===anchorId);
    const atom = cState.workspace.find(a=>a.id===newAtomId);
    if(!anchor || !anchor.pos){
      atom.pos = {x: STAGE_W/2 + (Math.random()-0.5)*20, y: STAGE_H/2 + (Math.random()-0.5)*20};
      return;
    }
    const angle = Math.random()*Math.PI*2;
    const jitter = BOND_REST_LEN*0.7;
    atom.pos = {x: anchor.pos.x + jitter*Math.cos(angle), y: anchor.pos.y + jitter*Math.sin(angle)};
  }

  function relaxTick(){
    const placed = cState.workspace.filter(a=>a.pos);
    const disp = {};
    placed.forEach(a=> disp[a.id] = {x:0,y:0});
    for(let i=0;i<placed.length;i++){
      for(let j=i+1;j<placed.length;j++){
        const A=placed[i].pos, B=placed[j].pos;
        const dx=B.x-A.x, dy=B.y-A.y;
        const dist = Math.hypot(dx,dy)||0.01;
        const force = Math.min(REPEL_K/(dist*dist), 40);
        const ux=dx/dist, uy=dy/dist;
        disp[placed[i].id].x -= ux*force; disp[placed[i].id].y -= uy*force;
        disp[placed[j].id].x += ux*force; disp[placed[j].id].y += uy*force;
      }
    }
    cState.bonds.forEach(b=>{
      const A = cState.workspace.find(a=>a.id===b.aId), B = cState.workspace.find(a=>a.id===b.bId);
      if(!A || !B || !A.pos || !B.pos) return;
      const dx=B.pos.x-A.pos.x, dy=B.pos.y-A.pos.y;
      const dist = Math.hypot(dx,dy)||0.01;
      const force = SPRING_K*(dist-BOND_REST_LEN);
      const ux=dx/dist, uy=dy/dist;
      disp[A.id].x += ux*force; disp[A.id].y += uy*force;
      disp[B.id].x -= ux*force; disp[B.id].y -= uy*force;
    });
    placed.forEach(a=>{
      disp[a.id].x += (STAGE_W/2 - a.pos.x)*CENTER_K;
      disp[a.id].y += (STAGE_H/2 - a.pos.y)*CENTER_K;
    });
    let maxMove = 0;
    placed.forEach(a=>{
      let dx=disp[a.id].x, dy=disp[a.id].y;
      const mag = Math.hypot(dx,dy);
      if(mag>MAX_STEP){ dx = dx/mag*MAX_STEP; dy = dy/mag*MAX_STEP; }
      a.pos.x = Math.max(20, Math.min(STAGE_W-20, a.pos.x+dx));
      a.pos.y = Math.max(20, Math.min(STAGE_H-20, a.pos.y+dy));
      maxMove = Math.max(maxMove, Math.hypot(dx,dy));
    });
    return maxMove;
  }

  let relaxRafId = null;
  function runRelax(){
    if(relaxRafId){ cancelAnimationFrame(relaxRafId); relaxRafId = null; }
    const reduceMotion = window.matchMedia && window.matchMedia("(prefers-reduced-motion: reduce)").matches;
    if(reduceMotion){
      for(let i=0;i<RELAX_MAX_TICKS;i++){ relaxTick(); }
      renderCovalent();
      return;
    }
    let settledTicks = 0, totalTicks = 0;
    function step(){
      const m = relaxTick();
      totalTicks++;
      renderCovalent();
      settledTicks = (m < RELAX_STOP_THRESHOLD) ? settledTicks+1 : 0;
      if(totalTicks < RELAX_MAX_TICKS && (settledTicks < 5 || totalTicks < RELAX_MIN_TICKS)){
        relaxRafId = requestAnimationFrame(step);
      } else {
        relaxRafId = null;
      }
    }
    step();
  }

  let cState, cNextId, cNextBondId;
```

Note: `runRelax` calls `renderCovalent()`, which is defined later in the same file. This is safe — `function renderCovalent(){...}` is a hoisted function declaration, so it's callable from code that runs earlier in the file, same as the existing codebase already relies on elsewhere (e.g. `attemptBond` in ionic mode calls `render()`, defined later, the same way).

- [ ] **Step 3: Smoke-check — page still loads cleanly**

Open `index.html` in a browser. Open DevTools console.
Expected: no errors printed, page looks and behaves exactly as before (nothing calls the new functions yet, so there is nothing to observe beyond "it still works").

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add spring-relaxation layout engine for covalent stage (unused yet)"
```

---

### Task 2: Extend covalent state with atom positions

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add `pos` to freshly spawned atoms**

Find:

```javascript
  function cFreshState(){
    cNextId = 1;
    cNextBondId = 1;
    return {
      movesLeft: TOTAL_C_MOVES,
      score: 0,
      challengeIndex: 0,
      workspace: [],
      bonds: [],
      armedId: null,
      won: false,
    };
  }
```

Replace with:

```javascript
  function cFreshState(){
    cNextId = 1;
    cNextBondId = 1;
    return {
      movesLeft: TOTAL_C_MOVES,
      score: 0,
      challengeIndex: 0,
      workspace: [],
      bonds: [],
      armedId: null,
      openBondId: null,
      won: false,
    };
  }
```

- [ ] **Step 2: Give spawned atoms a `pos` field**

Find:

```javascript
  function cSpawnAtom(symbol){
    if(cState.won) return;
    if(cState.movesLeft<=0){ setCFeedback("error","No moves left to pull new elements — try assembling with what's on the bench."); return; }
    cState.movesLeft--;
    cState.workspace.push({id: cNextId++, symbol});
    renderCovalent();
  }
```

Replace with:

```javascript
  function cSpawnAtom(symbol){
    if(cState.won) return;
    if(cState.movesLeft<=0){ setCFeedback("error","No moves left to pull new elements — try assembling with what's on the bench."); return; }
    cState.movesLeft--;
    cState.workspace.push({id: cNextId++, symbol, pos: null});
    renderCovalent();
  }
```

`pos: null` means "still in the tray, not yet placed on the stage" — this is the flag the rendering rewrite (Task 4) will use to decide tray vs. stage.

- [ ] **Step 3: Smoke-check — no regression**

Open `index.html` in a browser. Switch to the Covalent tab. Pull a couple of atoms from the palette, bond them using the existing (old) UI.
Expected: behaves exactly as before — `pos` is set but nothing reads it yet, so there's no visible change.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add pos field to covalent workspace atoms"
```

---

### Task 3: Wire bonding into the layout engine

**Files:**
- Modify: `index.html`

This makes `armToggle`/`createBond` place newly-bonded atoms and trigger relaxation. The old UI still renders every atom as a flat button regardless of `pos` (Task 4 changes that), so this task is still not visually different — but it's the task that actually produces meaningful position data.

- [ ] **Step 1: Replace `createBond`**

Find:

```javascript
  function createBond(aId,bId){
    const exists = cState.bonds.find(b=> (b.aId===aId&&b.bId===bId) || (b.aId===bId&&b.bId===aId));
    if(exists){ setCFeedback("neutral","Already bonded — use the + on that bond below to add another shared pair."); return; }
    const aAtom = cState.workspace.find(a=>a.id===aId);
    const bAtom = cState.workspace.find(a=>a.id===bId);
    if(!aAtom || !bAtom) return;
    if(usedValence(aId) >= CELEMENTS[aAtom.symbol].capacity){
      setCFeedback("error", CELEMENTS[aAtom.symbol].name + " already has every shared pair it can take — remove a bond first if you want to change it.");
      return;
    }
    if(usedValence(bId) >= CELEMENTS[bAtom.symbol].capacity){
      setCFeedback("error", CELEMENTS[bAtom.symbol].name + " already has every shared pair it can take — remove a bond first if you want to change it.");
      return;
    }
    cState.bonds.push({id: cNextBondId++, aId, bId, order:1});
    setCFeedback("neutral","Single bond formed — free. Use + below if the atoms still need more shared pairs.");
  }
```

Replace with:

```javascript
  function createBond(aId,bId){
    if(aId===bId) return;
    const exists = cState.bonds.find(b=> (b.aId===aId&&b.bId===bId) || (b.aId===bId&&b.bId===aId));
    if(exists){ setCFeedback("neutral","Already bonded — click the bond line to change its order."); return; }
    const aAtom = cState.workspace.find(a=>a.id===aId);
    const bAtom = cState.workspace.find(a=>a.id===bId);
    if(!aAtom || !bAtom) return;
    if(usedValence(aId) >= CELEMENTS[aAtom.symbol].capacity){
      setCFeedback("error", CELEMENTS[aAtom.symbol].name + " already has every shared pair it can take — remove a bond first if you want to change it.");
      return;
    }
    if(usedValence(bId) >= CELEMENTS[bAtom.symbol].capacity){
      setCFeedback("error", CELEMENTS[bAtom.symbol].name + " already has every shared pair it can take — remove a bond first if you want to change it.");
      return;
    }
    if(!aAtom.pos && !bAtom.pos){
      aAtom.pos = {x: STAGE_W/2 + (Math.random()-0.5)*20, y: STAGE_H/2 + (Math.random()-0.5)*20};
      seedPosition(bId, aId);
    } else if(aAtom.pos && !bAtom.pos){
      seedPosition(bId, aId);
    } else if(!aAtom.pos && bAtom.pos){
      seedPosition(aId, bId);
    }
    cState.bonds.push({id: cNextBondId++, aId, bId, order:1});
    setCFeedback("neutral","Single bond formed — free. Click the bond line if it still needs more shared pairs.");
    runRelax();
  }
```

- [ ] **Step 2: Clear the bond-order popover when arming**

Find:

```javascript
  function armToggle(atomId){
    if(cState.armedId===atomId){ cState.armedId=null; renderCovalent(); return; }
    if(cState.armedId===null){ cState.armedId=atomId; renderCovalent(); return; }
    createBond(cState.armedId, atomId);
    cState.armedId=null;
    renderCovalent();
  }
```

Replace with:

```javascript
  function armToggle(atomId){
    cState.openBondId = null;
    if(cState.armedId===atomId){ cState.armedId=null; renderCovalent(); return; }
    if(cState.armedId===null){ cState.armedId=atomId; renderCovalent(); return; }
    createBond(cState.armedId, atomId);
    cState.armedId=null;
    renderCovalent();
  }
```

- [ ] **Step 3: Trigger relax after atom removal too**

Find:

```javascript
  function removeAtomC(atomId){
    cState.workspace = cState.workspace.filter(a=>a.id!==atomId);
    cState.bonds = cState.bonds.filter(b=> b.aId!==atomId && b.bId!==atomId);
    if(cState.armedId===atomId) cState.armedId=null;
    renderCovalent();
  }
```

Replace with:

```javascript
  function removeAtomC(atomId){
    cState.workspace = cState.workspace.filter(a=>a.id!==atomId);
    cState.bonds = cState.bonds.filter(b=> b.aId!==atomId && b.bId!==atomId);
    if(cState.armedId===atomId) cState.armedId=null;
    cState.openBondId = null;
    runRelax();
  }
```

- [ ] **Step 4: Smoke-check — no regression**

Open `index.html`, Covalent tab. Spawn H, O, H. Bond H-O, then O-H (forming water) using the existing flat-button UI.
Expected: bonding still works exactly as before (each click still arms/bonds as it did previously — the extra position/relax work happens silently since nothing renders it yet). No console errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Wire bonding into the layout engine (not yet rendered)"
```

---

### Task 4: Replace bond-order controls with a single function

**Files:**
- Modify: `index.html`

Consolidates `upgradeBond` (which only ever incremented) into one `setBondOrder(bondId, action)` that also handles decrementing and removing, and rewires the *existing* bond-chip-list buttons to call it — so this task stays independently verifiable through the current UI, before that UI is replaced in Task 5.

- [ ] **Step 1: Replace `upgradeBond` with `setBondOrder`**

Find:

```javascript
  function upgradeBond(bondId){
    const bond = cState.bonds.find(b=>b.id===bondId);
    if(!bond) return;
    if(cState.movesLeft<=0){ setCFeedback("error","No moves left to upgrade a bond."); return; }
    const aAtom = cState.workspace.find(a=>a.id===bond.aId);
    const bAtom = cState.workspace.find(a=>a.id===bond.bId);
    const newUsedA = usedValence(bond.aId)+1, newUsedB = usedValence(bond.bId)+1;
    if(bond.order>=3 || newUsedA>CELEMENTS[aAtom.symbol].capacity || newUsedB>CELEMENTS[bAtom.symbol].capacity){
      setCFeedback("error","That bond can't take another shared pair — one of the atoms is already at capacity.");
      return;
    }
    cState.movesLeft--;
    bond.order++;
    renderCovalent();
  }
```

Replace with:

```javascript
  function setBondOrder(bondId, action){
    const bond = cState.bonds.find(b=>b.id===bondId);
    if(!bond) return;
    if(action==="remove"){
      cState.bonds = cState.bonds.filter(b=>b.id!==bondId);
      cState.openBondId = null;
      runRelax();
      return;
    }
    const delta = action==="up" ? 1 : -1;
    const newOrder = bond.order + delta;
    if(newOrder < 1){
      cState.bonds = cState.bonds.filter(b=>b.id!==bondId);
      cState.openBondId = null;
      runRelax();
      return;
    }
    if(newOrder > 3) return;
    if(action==="up" && cState.movesLeft<=0){ setCFeedback("error","No moves left to upgrade a bond."); return; }
    const aAtom = cState.workspace.find(a=>a.id===bond.aId);
    const bAtom = cState.workspace.find(a=>a.id===bond.bId);
    const otherUsed = (id)=> cState.bonds.reduce((s,bb)=> s + (bb.id!==bondId && (bb.aId===id||bb.bId===id) ? bb.order : 0), 0);
    if(otherUsed(bond.aId)+newOrder > CELEMENTS[aAtom.symbol].capacity){
      setCFeedback("error", CELEMENTS[aAtom.symbol].name + " already has every shared pair it can take.");
      return;
    }
    if(otherUsed(bond.bId)+newOrder > CELEMENTS[bAtom.symbol].capacity){
      setCFeedback("error", CELEMENTS[bAtom.symbol].name + " already has every shared pair it can take.");
      return;
    }
    if(action==="up") cState.movesLeft--;
    bond.order = newOrder;
    renderCovalent();
  }
```

Only upgrading (`"up"`) costs a move, matching the existing "first pair free, upgrades cost a move" economy. Downgrading and removing are free — cancelling a bond order back down while experimenting shouldn't be penalized.

- [ ] **Step 2: Remove the now-orphaned `removeBond` function**

Find:

```javascript
  function removeBond(bondId){
    cState.bonds = cState.bonds.filter(b=>b.id!==bondId);
    renderCovalent();
  }
```

Delete this block entirely (its job is now covered by `setBondOrder(bondId, "remove")`).

- [ ] **Step 3: Rewire the existing bond-chip buttons to the new function**

Find (inside `cRenderBondList`):

```javascript
      const plus = document.createElement("button");
      plus.className = "plus";
      plus.textContent = "+1";
      plus.title = "Add a shared pair (costs a move)";
      plus.disabled = !canUpgrade || cState.movesLeft<=0 || cState.won;
      plus.addEventListener("click", ()=>upgradeBond(b.id));

      const rm = document.createElement("button");
      rm.className = "remove";
      rm.textContent = "×";
      rm.title = "Remove this bond";
      rm.addEventListener("click", ()=>removeBond(b.id));
```

Replace with:

```javascript
      const plus = document.createElement("button");
      plus.className = "plus";
      plus.textContent = "+1";
      plus.title = "Add a shared pair (costs a move)";
      plus.disabled = !canUpgrade || cState.movesLeft<=0 || cState.won;
      plus.addEventListener("click", ()=>setBondOrder(b.id, "up"));

      const rm = document.createElement("button");
      rm.className = "remove";
      rm.textContent = "×";
      rm.title = "Remove this bond";
      rm.addEventListener("click", ()=>setBondOrder(b.id, "remove"));
```

- [ ] **Step 4: Verify through the existing (still-present) UI**

Open `index.html`, Covalent tab. Spawn O, H, H. Bond O-H, then O-H again (water, complete). In the "Bonds formed" chip list, click "+1" on one bond.
Expected: that bond's glyph changes from "–" to "=" (per `glyphs = {1:"–",2:"=",3:"≡"}` in `cRenderBondList`), moves-left counter decreases by 1. Click "×" on a bond.
Expected: that bond disappears from the list, moves-left unchanged (removal is free).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Consolidate bond-order controls into setBondOrder"
```

---

### Task 5: Replace bench/chip-list rendering with the live stage

**Files:**
- Modify: `index.html`

This is the core visual swap: HTML, CSS, and rendering functions change together since they're tightly coupled. After this task, the covalent mode looks and behaves like the validated prototype — atoms bond into a live, self-organizing SVG diagram instead of a flat button list with a separate chip list.

- [ ] **Step 1: Replace the covalent HTML markup**

Find:

```html
    <div class="chamber-shell" id="c-shell">
      <p class="chamber-label">Reaction bench</p>
      <div class="chamber" id="c-chamber"></div>
      <p class="chamber-label" style="margin-top:16px;">Bonds formed</p>
      <div class="bondlist" id="c-bondlist"></div>
      <div class="chamber-actions">
```

Replace with:

```html
    <div class="chamber-shell" id="c-shell">
      <p class="chamber-label">Reaction stage</p>
      <svg id="c-stage" viewBox="0 0 480 260" style="width:100%;height:260px;display:block;"></svg>
      <p class="chamber-label" style="margin-top:16px;">Tray</p>
      <div class="chamber" id="c-tray" style="min-height:64px;"></div>
      <div class="chamber-actions">
```

- [ ] **Step 2: Remove now-unused CSS (`.c-atom-badge`, `.bondlist`/`.bond-chip`)**

Find:

```css
  .c-atom-wrap{position:relative;width:46px;height:46px;}
  .c-atom-badge{
    position:absolute;bottom:-6px;right:-6px;
    background:var(--surface);color:var(--ink);
    font-family:"IBM Plex Mono",monospace;font-size:9px;font-weight:600;
    border-radius:8px;padding:1px 4px;box-shadow:var(--shadow);line-height:1.3;
    pointer-events:none;
  }
  .c-atom-remove{
```

Replace with:

```css
  .c-atom-wrap{position:relative;width:46px;height:46px;}
  .c-atom-remove{
```

Find:

```css
  .bondlist{display:flex;flex-wrap:wrap;gap:8px;min-height:22px;align-items:center;}
  .bondlist-empty{color:var(--muted);font-size:12.5px;}
  .bond-chip{
    display:flex;align-items:center;gap:5px;
    background:var(--surface);border:1px solid var(--line);border-radius:999px;
    padding:4px 6px 4px 4px;box-shadow:var(--shadow);
  }
  .bond-chip .sym-badge{
    width:20px;height:20px;border-radius:50%;flex:none;
    display:flex;align-items:center;justify-content:center;
    font-family:"IBM Plex Sans",sans-serif;font-weight:700;font-size:9.5px;
  }
  .bond-chip .order-glyph{
    font-family:"IBM Plex Mono",monospace;font-weight:700;font-size:15px;
    color:var(--muted);min-width:12px;text-align:center;
  }
  .bond-chip button{
    border:none;background:transparent;cursor:pointer;
    font-family:"IBM Plex Mono",monospace;font-size:11.5px;font-weight:600;
    border-radius:5px;padding:2px 6px;
  }
  .bond-chip button.plus{color:var(--accent);}
  .bond-chip button.plus:disabled{opacity:.3;cursor:not-allowed;}
  .bond-chip button.remove{color:var(--muted);}
  .bond-chip button.remove:hover{color:var(--danger);}
```

Delete this block entirely.

- [ ] **Step 3: Add stage-specific CSS**

Find:

```css
  @media (prefers-reduced-motion: reduce){
    .atom, .cell, button.btn, .chamber-shell.flash-success{animation:none !important;transition:none !important;}
  }
```

Replace with:

```css
  #c-stage circle, #c-stage text { transform-box: fill-box; transform-origin: center; }

  @media (prefers-reduced-motion: reduce){
    .atom, .cell, button.btn, .chamber-shell.flash-success{animation:none !important;transition:none !important;}
  }
```

`transform-box: fill-box` makes the existing `.leaving` shrink animation (used later, in Task 6) scale each SVG shape from its own center instead of the SVG viewport's corner.

- [ ] **Step 4: Replace `cRenderChamber` with `cRenderStage` + `cRenderTray`**

Find:

```javascript
  function cRenderChamber(){
    const chamber = document.getElementById("c-chamber");
    chamber.innerHTML = "";
    if(cState.workspace.length===0){
      const p = document.createElement("p");
      p.className = "chamber-empty";
      p.textContent = "Bench is empty — pull elements from the supply below.";
      chamber.appendChild(p);
      return;
    }
    cState.workspace.forEach(a=>{
      const el = CELEMENTS[a.symbol];
      const used = usedValence(a.id);
      const wrap = document.createElement("div");
      wrap.className = "c-atom-wrap";

      const btn = document.createElement("button");
      btn.className = "atom" + (cState.armedId===a.id ? " selected" : "");
      btn.style.background = el.bg;
      btn.style.color = el.fg;
      btn.dataset.catomId = a.id;
      btn.title = el.name;
      btn.textContent = a.symbol;
      btn.addEventListener("click", ()=>armToggle(a.id));

      const badge = document.createElement("span");
      badge.className = "c-atom-badge";
      badge.textContent = used + "/" + el.capacity;

      const rm = document.createElement("button");
      rm.className = "c-atom-remove";
      rm.textContent = "×";
      rm.title = "Remove atom";
      rm.addEventListener("click", (e)=>{ e.stopPropagation(); removeAtomC(a.id); });

      wrap.appendChild(btn);
      wrap.appendChild(badge);
      wrap.appendChild(rm);
      chamber.appendChild(wrap);
    });
  }
```

Replace with:

```javascript
  function cRenderStage(){
    const svgEl = document.getElementById("c-stage");
    const placed = cState.workspace.filter(a=>a.pos);
    if(placed.length===0){
      svgEl.innerHTML = '<text x="'+(STAGE_W/2)+'" y="'+(STAGE_H/2)+'" text-anchor="middle" font-size="13" fill="var(--muted)" font-family="IBM Plex Sans, sans-serif">Bench is empty — pull elements from the supply below.</text>';
      return;
    }
    let svg = "";
    cState.bonds.forEach(b=>{
      const aAtom = cState.workspace.find(a=>a.id===b.aId), bAtom = cState.workspace.find(a=>a.id===b.bId);
      if(!aAtom || !bAtom || !aAtom.pos || !bAtom.pos) return;
      const p1=aAtom.pos, p2=bAtom.pos;
      const dx=p2.x-p1.x, dy=p2.y-p1.y, len=Math.hypot(dx,dy)||1;
      const nx=-dy/len, ny=dx/len;
      const offsets = b.order===1?[0]:b.order===2?[-3.5,3.5]:[-5.5,0,5.5];
      offsets.forEach(off=>{
        svg += '<line x1="'+(p1.x+nx*off)+'" y1="'+(p1.y+ny*off)+'" x2="'+(p2.x+nx*off)+'" y2="'+(p2.y+ny*off)+'" style="stroke:var(--accent);stroke-width:2.5px;stroke-linecap:round;"></line>';
      });
      svg += '<line data-bondhit="'+b.id+'" x1="'+p1.x+'" y1="'+p1.y+'" x2="'+p2.x+'" y2="'+p2.y+'" stroke="transparent" stroke-width="18" style="cursor:pointer;pointer-events:auto;"></line>';
    });
    placed.forEach(a=>{
      const el = CELEMENTS[a.symbol];
      const used = usedValence(a.id);
      const ring = cState.armedId===a.id ? '<circle cx="'+a.pos.x+'" cy="'+a.pos.y+'" r="27" fill="none" stroke="var(--accent)" stroke-width="2.5"></circle>' : '';
      svg += ring;
      svg += '<g data-atomid="'+a.id+'">';
      svg += '<circle data-atomhit="'+a.id+'" cx="'+a.pos.x+'" cy="'+a.pos.y+'" r="23" fill="'+el.bg+'" stroke="var(--surface)" stroke-width="2" style="cursor:pointer;pointer-events:auto;"></circle>';
      svg += '<text x="'+a.pos.x+'" y="'+(a.pos.y+5)+'" text-anchor="middle" font-size="14" font-weight="700" fill="'+el.fg+'" font-family="IBM Plex Sans, sans-serif" style="pointer-events:none;">'+a.symbol+'</text>';
      svg += '<text x="'+a.pos.x+'" y="'+(a.pos.y+34)+'" text-anchor="middle" font-size="9" fill="var(--muted)" font-family="IBM Plex Mono, monospace">'+used+'/'+el.capacity+'</text>';
      svg += '</g>';
      svg += '<circle data-removehit="'+a.id+'" cx="'+(a.pos.x+18)+'" cy="'+(a.pos.y-18)+'" r="9" fill="var(--surface)" stroke="var(--line)" style="cursor:pointer;pointer-events:auto;"></circle>';
      svg += '<text x="'+(a.pos.x+18)+'" y="'+(a.pos.y-14)+'" text-anchor="middle" font-size="10" fill="var(--muted)" font-family="IBM Plex Mono, monospace" style="pointer-events:none;">&#10005;</text>';
    });
    if(cState.openBondId!==null){
      const b = cState.bonds.find(bb=>bb.id===cState.openBondId);
      if(b){
        const aAtom = cState.workspace.find(a=>a.id===b.aId), bAtom = cState.workspace.find(a=>a.id===b.bId);
        const p1=aAtom.pos, p2=bAtom.pos;
        const mx=(p1.x+p2.x)/2, my=(p1.y+p2.y)/2 - 26;
        svg += '<rect x="'+(mx-48)+'" y="'+(my-16)+'" width="96" height="32" rx="8" fill="var(--surface)" stroke="var(--line)"></rect>';
        svg += '<text data-pop="down" x="'+(mx-32)+'" y="'+(my+5)+'" text-anchor="middle" font-size="16" font-weight="700" fill="var(--ink)" font-family="IBM Plex Mono, monospace" style="cursor:pointer;pointer-events:auto;">&#8722;</text>';
        svg += '<text x="'+mx+'" y="'+(my+5)+'" text-anchor="middle" font-size="12" fill="var(--muted)" font-family="IBM Plex Mono, monospace">'+b.order+'</text>';
        svg += '<text data-pop="up" x="'+(mx+20)+'" y="'+(my+5)+'" text-anchor="middle" font-size="16" font-weight="700" fill="var(--ink)" font-family="IBM Plex Mono, monospace" style="cursor:pointer;pointer-events:auto;">+</text>';
        svg += '<text data-pop="remove" x="'+(mx+42)+'" y="'+(my+5)+'" text-anchor="middle" font-size="13" font-weight="700" fill="var(--danger)" font-family="IBM Plex Mono, monospace" style="cursor:pointer;pointer-events:auto;">&#10005;</text>';
      }
    }
    svgEl.innerHTML = svg;
    svgEl.querySelectorAll('[data-atomhit]').forEach(elx=>{
      elx.addEventListener("click", (e)=>{ e.stopPropagation(); armToggle(Number(elx.dataset.atomhit)); });
    });
    svgEl.querySelectorAll('[data-removehit]').forEach(elx=>{
      elx.addEventListener("click", (e)=>{ e.stopPropagation(); removeAtomC(Number(elx.dataset.removehit)); });
    });
    svgEl.querySelectorAll('[data-bondhit]').forEach(elx=>{
      elx.addEventListener("click", (e)=>{ e.stopPropagation(); const id=Number(elx.dataset.bondhit); cState.openBondId = cState.openBondId===id? null : id; renderCovalent(); });
    });
    svgEl.querySelectorAll('[data-pop]').forEach(elx=>{
      elx.addEventListener("click", (e)=>{ e.stopPropagation(); setBondOrder(cState.openBondId, elx.dataset.pop); });
    });
  }

  function cRenderTray(){
    const tray = document.getElementById("c-tray");
    tray.innerHTML = "";
    const unplaced = cState.workspace.filter(a=>!a.pos);
    if(unplaced.length===0){
      const p = document.createElement("p");
      p.className = "chamber-empty";
      p.textContent = cState.workspace.length===0 ? "Pull elements from the supply below to begin." : "All atoms placed on the stage.";
      tray.appendChild(p);
      return;
    }
    unplaced.forEach(a=>{
      const el = CELEMENTS[a.symbol];
      const wrap = document.createElement("div");
      wrap.className = "c-atom-wrap";

      const btn = document.createElement("button");
      btn.className = "atom" + (cState.armedId===a.id ? " selected" : "");
      btn.style.background = el.bg;
      btn.style.color = el.fg;
      btn.title = el.name;
      btn.textContent = a.symbol;
      btn.addEventListener("click", ()=>armToggle(a.id));

      const rm = document.createElement("button");
      rm.className = "c-atom-remove";
      rm.textContent = "×";
      rm.title = "Remove atom";
      rm.addEventListener("click", (e)=>{ e.stopPropagation(); removeAtomC(a.id); });

      wrap.appendChild(btn);
      wrap.appendChild(rm);
      tray.appendChild(wrap);
    });
  }
```

Note the atom-click handler wraps `elx.dataset.atomhit` in `Number(...)`: SVG `data-*` attributes always read back as strings, but `cState.workspace`/`cState.bonds` ids are numbers compared with strict `===` throughout this file (in `armToggle`, `createBond`, `usedValence`, etc.), so the id must be coerced back to a number at the point it leaves the DOM.

- [ ] **Step 5: Remove `cRenderBondList`**

Find the entire function (from `function cRenderBondList(){` through its closing `}` — it builds `.bond-chip` elements for `#c-bondlist`, iterating `cState.bonds` and creating `sym-badge`/`order-glyph`/`plus`/`remove` elements). Delete it entirely — the popover added in Step 4 replaces it.

- [ ] **Step 6: Update `renderCovalent` to call the new render functions**

Find:

```javascript
  function renderCovalent(){
    cRenderDocket();
    cRenderReadouts();
    cRenderChamber();
    cRenderBondList();
    cRenderPalette();
    document.getElementById("c-btn-assemble").disabled = cState.won || cState.bonds.length===0;
    document.getElementById("c-btn-discard").disabled = cState.workspace.length===0;
  }
```

Replace with:

```javascript
  function renderCovalent(){
    cRenderDocket();
    cRenderReadouts();
    cRenderStage();
    cRenderTray();
    cRenderPalette();
    document.getElementById("c-btn-assemble").disabled = cState.won || cState.bonds.length===0;
    document.getElementById("c-btn-discard").disabled = cState.workspace.length===0;
  }
```

- [ ] **Step 7: Verify — basic chain and branch**

Open `index.html`, Covalent tab. Spawn three C atoms. Click the first C (tray), click the second C (tray) — they should disappear from the tray and appear as two connected circles on the stage above, joined by a line.

Click one of the two stage circles, then click the third C (still in tray) — it should bond and join the stage, positioned near the atom you clicked (not stacked on top of it).

Expected: all three atoms visible as circles on the `#c-stage` SVG, connected by lines matching the bonds formed, animating into position over roughly half a second rather than snapping instantly.

- [ ] **Step 8: Verify — ring closure in "awkward" order (the scenario that motivated this design)**

Reset the covalent mode (`Reset` button). Spawn three C atoms (call them by click order C1, C2, C3). Bond C1 to C2. Bond C1 to C3 (both branches off the same atom). Now click C2 (on stage) then C3 (on stage) to close the ring.

Expected: the three atoms settle into a roughly equilateral triangle (not a flattened/degenerate shape) within about a second of the ring-closing click. This directly reproduces and verifies the fix for the layout bug found during design brainstorming.

- [ ] **Step 9: Verify — bond order popover**

With the ring from Step 8 still on stage, click one of the three bond lines.
Expected: a small popover with `−`, a number, `+`, and `×` appears near that bond's midpoint. Click `+`.
Expected: that specific bond becomes a double line (two parallel strokes); moves-left decreases by 1; the other two bonds remain single lines. Click `−` on the same bond.
Expected: it returns to a single line; moves-left does NOT change (downgrade is free). Click `×`.
Expected: that bond disappears entirely (the ring opens into a two-bond chain); the layout re-settles.

- [ ] **Step 10: Verify — atom removal**

Click the small `×` badge attached to one of the remaining stage atoms.
Expected: that atom and any bonds touching it disappear; remaining atoms re-settle via the layout engine.

- [ ] **Step 11: Commit**

```bash
git add index.html
git commit -m "Replace covalent bench/chip-list with live SVG stage + tray"
```

---

### Task 6: Rework Assemble to celebrate in place; remove dead reveal code

**Files:**
- Modify: `index.html`

Since the stage is now always a live diagram of the molecule, the separate end-of-round SVG regeneration (`layoutMolecule`/`moleculeSVG`) is redundant. This task flashes the matched cluster directly on the stage instead.

- [ ] **Step 1: Replace `cAttemptAssemble`**

Find:

```javascript
  function cAttemptAssemble(){
    const comps = getComponents();
    if(comps.length===0){ setCFeedback("neutral","Click two atoms on the bench to start a bond — the first shared pair is free."); return; }
    const target = CMOLECULES[cState.challengeIndex].atoms;

    for(const comp of comps){
      const counts = symbolCounts(comp);
      if(countsEqual(counts,target) && isComplete(comp)){
        const molecule = CMOLECULES[cState.challengeIndex];
        const ids = comp.slice();
        const compAtoms = ids.map(id=>cState.workspace.find(a=>a.id===id)).filter(Boolean);
        const compBonds = cState.bonds.filter(b=>ids.includes(b.aId) && ids.includes(b.bId));
        const revealSVG = moleculeSVG(compAtoms, compBonds);
        ids.forEach(id=>{
          const tile = document.querySelector('[data-catom-id="'+id+'"]');
          if(tile) tile.classList.add("leaving");
        });
        setTimeout(()=>{
          cState.workspace = cState.workspace.filter(a=>!ids.includes(a.id));
          cState.bonds = cState.bonds.filter(b=>!ids.includes(b.aId) && !ids.includes(b.bId));
          cState.score++;
          renderCovalent();
          document.getElementById("c-chamber").innerHTML =
            '<div class="molecule-reveal">' + revealSVG +
            '<span class="reveal-name">' + molecule.name + '</span>' +
            '<span class="reveal-formula">' + molecule.formula + '</span>' +
            '</div>';
          cCelebrate(molecule.name);
        }, 160);
        return;
      }
    }

    for(const comp of comps){
      const counts = symbolCounts(comp);
      if(countsEqual(counts,target)){
        const short = comp.map(id=>{
          const atom = cState.workspace.find(a=>a.id===id);
          return {symbol:atom.symbol, rem: CELEMENTS[atom.symbol].capacity - usedValence(id)};
        }).find(x=>x.rem>0);
        if(short){
          setCFeedback("error", CELEMENTS[short.symbol].name + " still needs " + short.rem + " more shared pair" + (short.rem>1?"s":"") + " — add or upgrade a bond.");
        } else {
          setCFeedback("error","Check your bond orders — the atoms are right but something's not adding up.");
        }
        return;
      }
    }

    if(comps.some(c=>isComplete(c))){
      setCFeedback("neutral","That's a fully-bonded, valid molecule — just not " + CMOLECULES[cState.challengeIndex].name + ". Check which elements you're using.");
      return;
    }
    setCFeedback("error","Not quite — check which elements are bonded and whether every atom's shared pairs add up.");
  }
```

Replace with:

```javascript
  function cAttemptAssemble(){
    const comps = getComponents();
    if(comps.length===0){ setCFeedback("neutral","Click two atoms on the bench to start a bond — the first shared pair is free."); return; }
    const target = CMOLECULES[cState.challengeIndex].atoms;

    for(const comp of comps){
      const counts = symbolCounts(comp);
      if(countsEqual(counts,target) && isComplete(comp)){
        const molecule = CMOLECULES[cState.challengeIndex];
        const ids = comp.slice();
        ids.forEach(id=>{
          const tile = document.querySelector('[data-atomid="'+id+'"]');
          if(tile) tile.classList.add("leaving");
        });
        setTimeout(()=>{
          cState.workspace = cState.workspace.filter(a=>!ids.includes(a.id));
          cState.bonds = cState.bonds.filter(b=>!ids.includes(b.aId) && !ids.includes(b.bId));
          cState.score++;
          cState.openBondId = null;
          renderCovalent();
          cCelebrate(molecule.name);
        }, 220);
        return;
      }
    }

    for(const comp of comps){
      const counts = symbolCounts(comp);
      if(countsEqual(counts,target)){
        const short = comp.map(id=>{
          const atom = cState.workspace.find(a=>a.id===id);
          return {symbol:atom.symbol, rem: CELEMENTS[atom.symbol].capacity - usedValence(id)};
        }).find(x=>x.rem>0);
        if(short){
          setCFeedback("error", CELEMENTS[short.symbol].name + " still needs " + short.rem + " more shared pair" + (short.rem>1?"s":"") + " — add or upgrade a bond.");
        } else {
          setCFeedback("error","Check your bond orders — the atoms are right but something's not adding up.");
        }
        return;
      }
    }

    if(comps.some(c=>isComplete(c))){
      setCFeedback("neutral","That's a fully-bonded, valid molecule — just not " + CMOLECULES[cState.challengeIndex].name + ". Check which elements you're using.");
      return;
    }
    setCFeedback("error","Not quite — check which elements are bonded and whether every atom's shared pairs add up.");
  }
```

`getComponents`, `symbolCounts`, `isComplete`, `countsEqual` are unchanged — they operate purely on atom-id graphs and don't reference layout at all. The `data-atomid` selector matches the `<g data-atomid="...">` wrapper added around each atom's circle+label in Task 5, so the whole atom group (circle + symbol + valence badge) gets the shrink animation, not just the circle.

- [ ] **Step 2: Delete the now-unused `layoutMolecule` and `moleculeSVG` functions**

Find and delete both functions in full — from `function layoutMolecule(atoms, bonds){` through the end of `function moleculeSVG(atoms, bonds){...}` (everything between and including both function bodies). Nothing calls either of them anymore after Step 1.

- [ ] **Step 3: Remove the now-unused `.molecule-reveal` CSS**

Find:

```css
  .molecule-reveal{
    display:flex;flex-direction:column;align-items:center;gap:6px;
    padding:10px 0 4px;width:100%;
  }
  .molecule-reveal svg{width:170px;height:auto;max-width:100%;}
  .molecule-reveal .reveal-name{
    font-family:"Fraunces", Georgia, serif;font-weight:600;font-size:16px;color:var(--ink);
  }
  .molecule-reveal .reveal-formula{
    font-family:"IBM Plex Mono", monospace;font-size:12.5px;font-weight:600;
    background:var(--accent-soft);color:var(--accent);padding:2px 9px;border-radius:5px;
  }
```

Delete this block entirely.

- [ ] **Step 4: Verify — successful assembly**

Reset covalent mode. Build water: spawn O, H, H. Bond O-H, then O-H again (both bonds single, both fill valence — O capacity 2, H capacity 1 each). Click "Assemble molecule".

Expected: the three atoms flash/shrink in place on the stage, then disappear; the stage's success flash animation plays (green inset flash on `#c-shell`) plus confetti (unless `prefers-reduced-motion` is on); feedback message shows "✓ Water assembled..."; score increments to 1/6; the next docket request (Hydrogen Chloride) appears; moves-left reflects what was spent.

- [ ] **Step 5: Verify — near-miss feedback still works**

Spawn a lone Cl atom (capacity 1) and click "Assemble molecule" without bonding it to anything.
Expected: feedback explains it's not a match (Cl alone isn't the current target) — same wording logic as before, unchanged.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Celebrate matched molecules in place on the stage; remove dead reveal code"
```

---

### Task 7: Cleanup, reset/clear-bench correctness, and full scenario pass

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Cancel any in-flight relaxation on Clear/Reset**

Find:

```javascript
  function clearBenchC(){
    cState.workspace = [];
    cState.bonds = [];
    cState.armedId = null;
    setCFeedback("neutral","Bench cleared — no move cost.");
    renderCovalent();
  }
```

Replace with:

```javascript
  function clearBenchC(){
    if(relaxRafId){ cancelAnimationFrame(relaxRafId); relaxRafId = null; }
    cState.workspace = [];
    cState.bonds = [];
    cState.armedId = null;
    cState.openBondId = null;
    setCFeedback("neutral","Bench cleared — no move cost.");
    renderCovalent();
  }
```

Find:

```javascript
  function resetCovalent(){
    cState = cFreshState();
    setCFeedback("neutral","Bench reset. Click two atoms on the bench to start a bond — the first shared pair is free.");
    renderCovalent();
  }
```

Replace with:

```javascript
  function resetCovalent(){
    if(relaxRafId){ cancelAnimationFrame(relaxRafId); relaxRafId = null; }
    cState = cFreshState();
    setCFeedback("neutral","Bench reset. Click two atoms on the bench to start a bond — the first shared pair is free.");
    renderCovalent();
  }
```

Without this, clicking Reset or Clear mid-animation leaves a stale `requestAnimationFrame` loop running against the just-replaced (or emptied) state — harmless in practice since it operates on an empty workspace and naturally terminates, but wasteful and worth cancelling cleanly.

- [ ] **Step 2: Full interactive scenario pass**

Open `index.html`. Work through each scenario below in the Covalent tab, resetting between scenarios as needed. This exercises everything the design spec called for.

1. **Independent fragments merge**: build a 3-carbon chain (C1-C2-C3), then separately build a second chain (C4-C5-C6) by clicking two fresh tray atoms together. Confirm both chains appear on the stage, visually separated (not overlapping). Then click C3 and C4 (both on stage) to join them into one 6-carbon chain. Confirm it re-settles into a single connected zigzag with no overlaps.
2. **Ring on top of a chain**: from the joined 6-chain, bond C6 back to C1, closing a 6-membered ring. Confirm it settles into a hexagon-ish shape without any bond lines crossing through unrelated atoms.
3. **Capacity limits respected**: try to add a 5th bond to a carbon that already has 4 shared pairs used (e.g. via repeated `+` clicks on bonds attached to one atom). Confirm the popover's `+` either does nothing or the feedback message explains the atom is at capacity — capacity is never exceeded.
4. **Moves economy**: note moves-left, spawn several atoms (each spawn costs 1 move), upgrade a bond order once (costs 1 move), downgrade it back down (free), remove a bond (free), remove an atom (free — only spawning costs a move). Confirm moves-left only decreased by the number of spawns plus the one upgrade.
5. **Win state**: complete all 6 target molecules in sequence (methane, water, HCl, O₂, CO₂, N₂ — refer to `CMOLECULES` for exact composition) and confirm the "Docket cleared" banner appears with the correct remaining-moves count, and "Run it again" resets cleanly.
6. **Mode switching**: switch to Ionic mode and back to Covalent mid-build (with atoms on the stage). Confirm the covalent stage/tray state is preserved and still interactive after switching back.

- [ ] **Step 3: Accessibility spot checks**

In DevTools, use the rendering/device toolbar (or your OS accessibility settings) to enable "prefers-reduced-motion: reduce", then reload `index.html`.
Expected: bonding atoms positions the stage instantly (no visible settling animation) since `runRelax` runs all ticks synchronously under reduced motion — confirmed by the layout still ending up correct (e.g. redo the ring-closure scenario from Task 5 Step 8) without an animated transition, and confetti does not appear on a successful assemble (this already worked before this feature, via the pre-existing `spawnConfetti` check — confirm it's still true).

Toggle to dark mode (OS-level or browser dev tools `prefers-color-scheme: dark` emulation) and reload.
Expected: the stage's bond lines, atom circles, popover, and valence badges all use the existing CSS custom properties (`var(--accent)`, `var(--surface)`, `var(--muted)`, etc.) and read correctly in dark mode, same as the rest of the page — no hardcoded light-mode colors were introduced in Tasks 1–6 (all stage SVG fills/strokes use `var(--...)` tokens, not literal hex values).

- [ ] **Step 4: Final commit**

```bash
git add index.html
git commit -m "Clean up relax cancellation on reset/clear; verify full covalent stage flow"
```

---

## Self-review notes

- **Spec coverage**: tray→stage transition (Task 5), order-independent layout via relaxation (Task 1/3, verified Task 5 Step 8), bond-order popover replacing the chip list (Task 4/5, verified Task 5 Step 9), per-atom removal as the undo mechanism (Task 3/5, verified Task 5 Step 10), explicit Assemble button with in-place celebration (Task 6), reduced-motion and dark-mode compatibility (Task 7 Step 3), ionic mode / content set / difficulty tiers left untouched (no task modifies `ELEMENTS`, `CHALLENGES`, ionic-mode functions, or `CMOLECULES`/`CELEMENTS`).
- **Type/naming consistency**: `armToggle`, `createBond`, `removeAtomC`, `usedValence`, `cState`, `cRenderDocket`, `cRenderReadouts`, `cRenderPalette`, `getComponents`, `symbolCounts`, `isComplete`, `countsEqual`, `cCelebrate`, `cSpawnConfetti` are all reused with their existing names/signatures throughout — no renamed duplicates. New names (`seedPosition`, `relaxTick`, `runRelax`, `cRenderStage`, `cRenderTray`, `setBondOrder`) are each defined exactly once and referenced consistently by that same name in every later task.
- **No placeholders**: every step above contains complete, exact code — no "similar to Task N" references, no TBD/TODO markers.
