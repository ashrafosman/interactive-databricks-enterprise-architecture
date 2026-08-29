# Port the AI Architecture Assistant into the fork — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Port the AI Architecture Assistant (chat → generates a tailored, industry-detected architecture tab) from the `Arch2` prototype onto this fork's further-evolved `app/index.html`, and give the fork a FastAPI backend so the assistant can call a Databricks-hosted model.

**Architecture:** The fork already has the exact board engine the assistant drives (`build`, `byId`, `nameIndex`/`roleIndex`/`resolveAtom`, `applyReference`/`paintRef`, `ref-banner`, `INDUSTRIES`/`INDUSTRY_CATALOG`, tabs) AND a **superset** of our per-tab board machinery (`refSnap` + `parkActiveBoard`/`loadTabBoard`/`boardSnapshot`/`loadBoardSnap`/`persistCustom`, `makeTab(id,name,text,snap)`). So we DROP our `CANON`/`setActiveArch`/`captureCanon` and rebase our AI code onto the fork's `boardSnapshot`/`refSnap`/`snap` model. The assistant is otherwise absent from the fork (zero conflict — purely additive).

**Tech Stack:** Single-file vanilla JS (`app/index.html`, ~2.97 MB, SCHEMA 26). FastAPI backend replaces the fork's stdlib static server. No JS test framework — verify in-browser via the local backend + Chrome DevTools.

## Source of truth (the Arch2 prototype)
`/Users/ashraf.osman/Documents/Dev/Vibes/Arch2/index.html` — our working AI implementation. Lift code from here; adapt per the anchors below. Our `app.py`/`requirements.txt`/`app.yaml` are the backend source.

## Fork integration anchors (verified against `app/index.html` @ 8a5d798)
- CSS insert point: just before `</style>` (line ~2208).
- AI DOM insert point: just after `<div class="tip" id="tip" ...>` (line ~2390), before the main `<script>` (2392).
- AI JS insert point: near the END of the main script, before its `</script>` (line ~17692) — after all engine/tab functions are defined.
- CSS variables present and identical: `--brand`, `--card`, `--line`, `--ink`, `--bg`, `--muted`, `--brand-wash`, `--brand-bd`, `--well` → our `.ai-*` CSS themes correctly.
- Engine our AI relies on (all present, compatible): `resolveAtom(name)` (nameIndex||roleIndex), `applyReference(ref)`, `paintRef`, `clearReference`, `activeRef`, `refBanner`, `openDetail` reads `activeRef.map[t.n]`, `esc`, `svg`, `INDUSTRIES`, `INDUSTRY_CATALOG` (`[id,label]`), `RAIL_IDS=["src","ing","ppl","cons"]`, `SCHEMA=26`, `BASE`, `medallionStages`, `prioritiseUseCases`, `industryLabel`.
- Board model to rebase onto: `boardSnapshot()` → `{schema,industry,bands,rails,top,cloud}`; `refSnap` (reference board's parked snapshot); `activeBoard` pointer; `parkActiveBoard()`/`loadTabBoard(id)`/`loadBoardSnap(s)`/`rebuildBoard()`; `makeTab(id,name,text,snap)` (snap = a board); `selectTab(tab)` parks/loads/rebuilds; `persistCustom()` already routes reference vs active tab; `persistTabs()`.
- The fork's `+` tab calls `cloneCurrent()` (not addTab). Leave it alone.

## Global Constraints
- All frontend edits in `app/index.html`'s markup/CSS/main `<script>`. No frameworks in the page, plain DOM.
- Do NOT reintroduce `CANON`/`setActiveArch`/`captureCanon` — use the fork's `refSnap`/`boardSnapshot`/`loadBoardSnap`/`parkActiveBoard`.
- A generated tab is a normal clone-style tab: `makeTab(id, title, "", snap)` where `snap` is the filtered board `{schema:SCHEMA, industry, bands, rails, top, cloud}`. Then `selectTab(tab)`.
- `filterArch` and `industryArch` build from a base snapshot (`refSnap` or an industry board), never from a `CANON` global.
- `SCHEMA` is the fork's (26); every snapshot our code stamps must use the fork's `SCHEMA` constant, not a literal.
- Keep the fork's `applyIndustry` behavior intact (incl. `prioritiseUseCases`, GoatCounter tracking) — only extract its pure transform.
- Verification: from `app/`, run the FastAPI backend (`uvicorn app:app --port 8791` with `DATABRICKS_PROFILE=arch-demo`, serving `app/index.html`), drive via Chrome DevTools MCP with a cache-busting `?v=`. Assert VISIBLE board (`getBoundingClientRect().height>0`), not bare `.atom` counts.

---

### Task 1: FastAPI backend for the fork

**Files:** `app/app.py` (create), `app/requirements.txt` (create), `app/app.yaml` (modify), keep `app/main.py` (unused fallback, or delete — see step).

**Interfaces:** Produces `/health` and `POST /generate {system,user,model}->{text}` served alongside `app/index.html`, calling `databricks-claude-sonnet-5`.

- [ ] **Step 1:** Copy `Arch2/app.py` → `app/app.py`. Change `HTML_PATH` to serve `app/index.html` from the app dir (it already resolves `index.html` next to `app.py`, and both live in `app/`, so no change needed — verify the path). Keep the reasoning-model content flattening (`_content_text`) and the no-`temperature` call (sonnet-5 rejects it).
- [ ] **Step 2:** Copy `Arch2/requirements.txt` → `app/requirements.txt` (fastapi, uvicorn, openai, databricks-sdk).
- [ ] **Step 3:** Replace `app/app.yaml` command with the uvicorn command from `Arch2/app.yaml` (`uvicorn app:app --host 0.0.0.0 --port 8000`) and the `SERVING_ENDPOINT=databricks-claude-sonnet-5` env. (The fork's Databricks App port is injected as `DATABRICKS_APP_PORT`; keep `--port 8000` or read the env — match the fork's platform expectation. If the platform injects the port, bind to it.)
- [ ] **Step 4:** Decide `main.py` — the fork's static server is now superseded; leave it in place (harmless, not referenced by app.yaml) OR delete it. Recommend leaving it with a one-line note, to minimize churn in the PR.
- [ ] **Step 5 (verify):** From `app/`, `pip install -r requirements.txt` in the venv, `DATABRICKS_PROFILE=arch-demo uvicorn app:app --port 8791`. `curl /health` → ok; `curl / | grep -c "<title"` → 1 (serves the fork page); `curl -X POST /generate` with a tiny prompt → `{text: "..."}`. Confirm the served page is the fork's (SCHEMA 26 present).
- [ ] **Step 6:** Commit: `feat(app): FastAPI backend serving the diagram + /generate for the AI assistant`.

---

### Task 2: AI CSS + DOM

**Files:** `app/index.html`.

- [ ] **Step 1:** Lift the AI CSS block from `Arch2/index.html` lines ~1918–1964 (`.ai-fab` … `.ai-key-row`) PLUS the `.ai-indhint` rules (~1947-1953) and insert just before `</style>` (~2208). These reference `--brand/--card/--line/--ink/--bg/--muted/--brand-wash/--brand-bd/--well`, all present in the fork.
- [ ] **Step 2:** Lift the AI DOM from `Arch2/index.html` — the `<button class="ai-fab" …>` through the closing `</section>` of `#ai-panel` (lines ~2149–2178, INCLUDING the `<div class="ai-indhint" id="ai-indhint" hidden>`). Insert immediately after `<div class="tip" id="tip" role="tooltip"></div>` (~2390), before the main `<script>`.
- [ ] **Step 3 (verify):** cache-busting load; `document.getElementById("ai-fab")` exists; clicking it shows `#ai-panel`; CSS themes in light/dark. No console errors. (No JS wired yet — the FAB just opens an empty panel; that's fine until Task 3.)
- [ ] **Step 4:** Commit: `feat(ui): AI assistant FAB, panel, and industry type-ahead DOM/CSS`.

---

### Task 3: AI JS — lift + rebase onto the fork's board model

**Files:** `app/index.html` (main `<script>`, before its `</script>` ~17692).

This is the core adaptation. Lift the whole AI JS block from `Arch2/index.html` (from `const AI_MODEL = "claude-sonnet-5";` through the end of the AI wiring), then make these SPECIFIC changes:

- [ ] **Step 1: Lift the transport + prompt + chat UI verbatim** — `AI_MODEL`, `aiState`, `componentCatalog`, `catalogFromArch`, `archAtomSet`, `industryPromptList`, `aiSystemPrompt(catalogText)`, `aiExtractJson`, `aiBridgeHealthy`, `aiCallBridge`, `aiCallApi`, `aiExtractJson`, the `aiEls`/`aiAddMsg`/`aiSetBusy`/`aiRefreshConn` chat UI, the FAB/x/send/keydown/suggestion/clear/key-toggle wiring, and the type-ahead chip block (`IND_SYNONYMS`, `IND_STOP`, `aiIndustryMatches`, `aiIndHint`). These depend only on engine primitives the fork already has (`resolveAtom`, `INDUSTRY_CATALOG`, `INDUSTRIES`, `applyIndustry`, `esc`, `refBanner`, `activeRef`, `clearReference`, `tabsEl`). `componentCatalog()` walks `byId` — identical in the fork.

- [ ] **Step 2: Extract `applyIndustryTo(fields,id)` from the FORK's `applyIndustry`** (fork line ~13066). Copy the fork's transform body (BASE reset of rails/top, medallion stage copy via a LOCAL findStages over `fields.bands`, industry overlay incl. `prioritiseUseCases(fields.top.secs)`) into a pure `applyIndustryTo(fields,id)`. Then rewrite the fork's `applyIndustry` to call `applyIndustryTo(ARCH,id); ARCH.industry = INDUSTRIES[id]?id:"generic";` and keep ALL its side-effects (activeRef clear, persistCustom+tracking, ind-active, build, paintIndustryMenu, syncIndustryBtn, drawer, fitBoard) unchanged. Verify parity (fork `applyIndustry` result == `applyIndustryTo` on a clone).

- [ ] **Step 3: Add `filterArch(selected, base)` rebased onto snapshots.** Same walker/prune as Arch2, but the source is `base || refSnap` (NOT `CANON`). Use `boardSnapshot()` semantics for the shape. Return `{schema:SCHEMA, industry:(src.industry||"generic"), bands, rails, top, cloud}`. Since the reference board's parked snapshot is `refSnap`, and `refSnap` is null until first park, guard: `const src = base || refSnap || boardSnapshot();`.

- [ ] **Step 4: Add `industryArch(id)`** — deep-clone `(refSnap || boardSnapshot())`'s `{bands,rails,top,cloud}`, run `applyIndustryTo(clone,id)`, set `clone.industry=id`, return. Never mutates the live board.

- [ ] **Step 5: Add `catalogFromArch`/`archAtomSet`** verbatim from Arch2 (they walk an arch object; shape identical).

- [ ] **Step 6: `aiCreateTab(r, baseArch)` rebased onto the fork's tab model.** Validate picks via `archAtomSet(baseArch)` when given else `resolveAtom`; build `snap = filterArch(selected, baseArch || null)`; then create the tab the FORK way:
  ```js
  const id = "custom" + (++tabSeq);
  if(activeRef){ clearReference(); refBanner.classList.remove("show"); }
  const tab = makeTab(id, title, "", snap);   // fork: snap is the tab's board
  selectTab(tab);                              // fork: parks current, loads snap, rebuilds
  persistTabs();
  return { count, title, industry: baseArch ? baseArch.industry : null };
  ```
  (No `setActiveArch`, no notes-overlay swap needed for board switching — the fork's `selectTab`→`loadTabBoard`→`rebuildBoard` renders the snap. For the usage-notes overlay, after `selectTab`, set `activeRef` to `{id:"__tab__", name:title, blurb, map:notes}` and show the banner WITHOUT dimming — reuse the Arch2 `setActiveNotes` behavior, but call it after selectTab so rebuildBoard's `paintRef(activeRef)` — if any — doesn't dim. Verify the fork's rebuild path re: activeRef and add a `__tab__` guard in the fork's `paintRef` if it dims like Arch2's did.)

- [ ] **Step 7: `aiSend` two-phase flow** verbatim from Arch2 (phase-1 detect on generic catalog; phase-2 re-select grounded on `catalogFromArch(industryArch(indId))` when `INDUSTRIES[result.industry]`; then `aiCreateTab(result, baseArch)`). Hoist `convo` for the phase-2 bridge call.

- [ ] **Step 8: Type-ahead chip target** — the chip's click does `applyIndustry(m.id, true)` after selecting the Reference tab first (`selectTab(tabsEl.querySelector('.tab[data-tab="reference"]'))`), same as Arch2. Confirm the fork's Reference tab id is `"reference"` (it is).

- [ ] **Step 9 (verify, real model):** cache-busting load; banking prompt → new tab, `arch.industry`/snap industry "banking", board VISIBLE, at least one banking-specific atom present (in `archAtomSet(industryArch("banking"))` but not `resolveAtom`-able generically), Reference board unchanged, console clean. Generic prompt → single call, CANON-equivalent (refSnap) board. Type "hospital" → Healthcare chip → picking switches Reference board. `node --check` on the extracted main script OK.

- [ ] **Step 10:** Commit: `feat(ai): architecture assistant — generate industry-detected tabs, type-ahead`.

---

### Task 4: Docs + regression + deploy prep

**Files:** `README.md` (fork), `app/index.html` (only if a regression fix is needed).

- [ ] **Step 1: Regression sweep** (in-browser): the fork's existing features still work — Reference board builds; Industry menu switches board (via the refactored `applyIndustry`→`applyIndustryTo`); cloud/palette/shape/stage toggles; `cloneCurrent` (+ tab); export/download; existing persisted boards load (SCHEMA 26). Reuse visible-atom assertions.
- [ ] **Step 2: Update `README.md`** — add an "AI Architecture Assistant" section: what it does (chat → industry-detected, tailored tab), the two-phase grounding, the FastAPI backend + `SERVING_ENDPOINT`, and the type-ahead chip. Match the README's existing tone/structure.
- [ ] **Step 3:** Commit: `docs: document the AI Architecture Assistant + FastAPI backend`.

---

### Task 5: Deploy to the FEVM app

**Files:** none (deploy actions).

- [ ] **Step 1:** From `ide-fork/app`, sync to the FEVM workspace app source path (the `arch-explorer` app in workspace `fevm-arch-explorer-demo`, profile `arch-demo`) — `databricks sync app/ /Workspace/Users/ashraf.osman@databricks.com/arch-explorer -p arch-demo` (exclude nothing app-breaking; include `index.html`, `app.py`, `app.yaml`, `requirements.txt`).
- [ ] **Step 2:** `databricks apps deploy arch-explorer --source-code-path /Workspace/Users/ashraf.osman@databricks.com/arch-explorer -p arch-demo`. Confirm "App started successfully".
- [ ] **Step 3:** The app SP already can query `databricks-claude-sonnet-5` (confirmed working in the current deploy). If a fresh grant is needed, surface it to the user (IAM change — do not self-authorize).
- [ ] **Step 4:** Report the app URL; user verifies in-browser (SSO required, can't be driven headlessly).

---

## Self-Review
- Spec coverage: backend (T1), UI (T2), AI logic rebased onto fork board model (T3), docs/regression (T4), deploy (T5).
- No placeholders: every step names exact files, fork anchors, and the specific rebase changes.
- Type consistency: `filterArch(selected, base)`, `industryArch(id)->snap`, `aiCreateTab(r, baseArch)->{count,title,industry}`, `applyIndustryTo(fields,id)`. Snapshot shape `{schema,industry,bands,rails,top,cloud}` matches the fork's `boardSnapshot`.
- Key risk called out (T3 Step 6): the fork's `paintRef`/rebuild may or may not dim on a `__tab__` ref — verify and add the guard only if needed (in Arch2 it did).
