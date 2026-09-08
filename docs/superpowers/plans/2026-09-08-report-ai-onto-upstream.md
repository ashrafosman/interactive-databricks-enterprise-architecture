# Re-port AI Assistant onto Upstream as a Shared-Template Sub-App — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adopt upstream's modular rewrite as the fork's new `main`, and re-apply the AI Architecture Assistant + FastAPI backend as a self-contained sub-app at `app/ai/` that reuses upstream's shared architecture templates (so new templates are automatically usable).

**Architecture:** Reset `main` to `upstream/main` (our work preserved on a backup branch). Build `app/ai/index.html` as a copy of upstream's modular `app/index.html` with our AI assistant grafted in, using `<base href="../">` so all template fetches resolve against the shared `app/`. The key adaptation: upstream loads industries lazily/asynchronously from YAML (`loadIndustry(id)` is async, `INDUSTRIES` starts empty), so our AI grounding must `await loadIndustry(id)` before snapshotting an industry board.

**Tech Stack:** Single-file vanilla JS (`app/ai/index.html`), FastAPI backend (`app/ai/app.py`), Databricks-hosted `databricks-claude-sonnet-5`. No JS test framework — verify in-browser via Chrome DevTools MCP against the live model.

## Source of truth
- **AI code to port:** the fork's current `main` BEFORE reset — preserved as branch `pre-upstream-backup`. Files: `app/index.html` (2.9 MB monolith; the AI CSS/DOM/JS blocks), `app/app.py`, `app/requirements.txt`, `app/app.yaml`. Reference via `git show pre-upstream-backup:app/index.html`.
- **New base:** `upstream/main`'s `app/index.html` (793 KB modular) and its `app/architectures/*.yaml`, `resources/`, `translations/`, `vendor/js-yaml.min.js`, `arch_schema.js`, `industry_icons.js`.

## Verified upstream anchors (in the NEW `app/index.html`)
- `<head>` opens line 3, `<title>` line 9, `</head>` line 2504. Insert `<base href="../">` right after `<head>` (line 3).
- `INDUSTRY_CATALOG = [` line ~3859; `const INDUSTRIES = {};` line ~3898.
- `async function applyIndustry(id, remember)` line ~3931 — **async**.
- `async function loadIndustry(id)` line ~4048 — fetches `architectures/<id>.yaml`, parses via `jsyaml` + `ArchSchema.toTerse`, populates `INDUSTRIES[id]`. Returns bool.
- `let INDUSTRY_BUILT = null;` line ~4077; populated from `architectures/manifest.json` line ~4090.
- `let refSnap = null;` line ~4361; `resolveAtom(name)` line ~4396; `boardSnapshot()` line ~4950; `build()` line ~5109; `selectTab(tab)` line ~5607; `makeTab(id,name,text,snap)` line ~5662.
- Upstream `app/app.yaml` runs `python main.py` (stdlib server); `app/main.py` is a `SimpleHTTPRequestHandler`.

## Global Constraints
- Do NOT modify upstream's `app/index.html`, `app/main.py`, or `app/app.yaml`. The reference explorer stays pristine. All new work is additive under `app/ai/`.
- Templates are never copied. `app/ai/index.html` reaches them via `<base href="../">`.
- Snapshot shape stamped by our code: `{schema:SCHEMA, industry, bands, rails, top, cloud}` — use the base file's `SCHEMA` constant, never a literal.
- Industry access is async: any code reading `INDUSTRIES[id]` must first `await loadIndustry(id)`.
- Keep the backend security posture: bind localhost when standalone; `/generate` OAuth via `WorkspaceClient`; `max_tokens=16000`; `finish_reason=="length"` → clean 502; `_content_text()` reasoning-block flattening.
- Frontend transport keeps `max_tokens:16000` + `stop_reason=="max_tokens"` guard.
- Verification asserts the VISIBLE board (`getBoundingClientRect().height>0`), not bare `.atom` counts. Use cache-busting `?v=` loads.

---

### Task 1: Back up current work, reset `main` to upstream

**Files:** none (git operations only).

**Interfaces:**
- Produces: branch `pre-upstream-backup` (our 9 commits + design/plan docs); `main` == `upstream/main` + the two docs commits cherry-picked on top.

- [ ] **Step 1:** Confirm clean tree and current HEAD.
  Run: `cd /Users/ashraf.osman/Documents/Dev/Vibes/ide-fork && git status -sb && git rev-parse main`
  Expected: clean; HEAD is the docs-commit on top of `ad48079`.

- [ ] **Step 2:** Create the backup branch pointing at current `main`.
  Run: `git branch pre-upstream-backup main`
  Expected: branch created (no output).

- [ ] **Step 3:** Save the two doc commits' hashes so they survive the reset.
  Run: `git log --oneline -3 main` — note the design-doc and plan commit SHAs (the AI code is safe on `pre-upstream-backup`).

- [ ] **Step 4:** Reset `main` to upstream.
  Run: `git checkout main && git reset --hard upstream/main && git log --oneline -1`
  Expected: HEAD is `3e75c44` (upstream tip).

- [ ] **Step 5:** Restore the design + plan docs onto the new base (they live in `docs/superpowers/`, untouched by upstream).
  Run: `git checkout pre-upstream-backup -- docs/superpowers/specs/2026-09-08-report-onto-upstream-ai-subapp-design.md docs/superpowers/plans/2026-09-08-report-ai-onto-upstream.md && git add docs/ && git commit -m "docs: carry AI sub-app design+plan onto upstream base"`
  Expected: commit created.

- [ ] **Step 6 (verify):** Upstream base intact + our docs present.
  Run: `test -f app/architectures/banking.yaml && test -f app/index.html && test -f docs/superpowers/plans/2026-09-08-report-ai-onto-upstream.md && wc -c app/index.html`
  Expected: all exist; `app/index.html` ~793 KB (proves the modular base is active).

---

### Task 2: Scaffold `app/ai/` — modular base copy + `<base href>` + shared-asset reach

**Files:**
- Create: `app/ai/index.html` (copy of `app/index.html` + `<base href="../">`)

**Interfaces:**
- Produces: `app/ai/index.html` that renders the upstream reference board by fetching all templates from `../` (the shared `app/`). No AI code yet.

- [ ] **Step 1:** Copy the modular base into the sub-app.
  Run: `mkdir -p app/ai && cp app/index.html app/ai/index.html`

- [ ] **Step 2:** Insert `<base href="../">` immediately after `<head>` (line 3) so every relative fetch/script resolves against `app/`.
  Edit `app/ai/index.html`: after the `<head>` line, add a new line `<base href="../">`.

- [ ] **Step 3 (verify fetch resolution):** Start upstream's stdlib server from `app/` and load the sub-app.
  Run (background): `cd app && python3 main.py` (serves `app/` at `:8000`).
  Drive via Chrome DevTools MCP: navigate to `http://localhost:8000/ai/index.html?v=1`. Assert:
  - `document.querySelector('base').getAttribute('href') === "../"`.
  - Network: `architectures/manifest.json`, `vendor/js-yaml.min.js`, `arch_schema.js` all 200 (resolved to `/…`, not `/ai/…`).
  - The reference board is VISIBLE: `document.querySelector('.board').getBoundingClientRect().height > 0`.
  - Console clean (no 404s, no JS errors).

- [ ] **Step 4:** Commit.
  Run: `git add app/ai/index.html && git commit -m "feat(ai): scaffold app/ai sub-app on the modular base with shared-template base href"`

---

### Task 3: Port the AI CSS + DOM into `app/ai/index.html`

**Files:**
- Modify: `app/ai/index.html`

**Interfaces:**
- Consumes: CSS/DOM blocks from `pre-upstream-backup:app/index.html` (`.ai-fab`…`.ai-key-row`, `.ai-indhint`; the `<button class="ai-fab">`…`#ai-panel` DOM incl. `#ai-indhint`).
- Produces: a working FAB that opens an (as-yet inert) panel; themed via existing `--brand/--card/--line/--ink/--bg/--muted/--brand-wash/--brand-bd/--well` tokens.

- [ ] **Step 1:** Extract the AI CSS block from the backup.
  Run: `git show pre-upstream-backup:app/index.html | grep -n "ai-fab\|ai-panel\|ai-indhint\|ai-key-row" | head` — locate the CSS block bounds, then extract that range to inspect.

- [ ] **Step 2:** Insert the AI CSS block just before `</style>` in `app/ai/index.html`. Confirm every custom prop it uses exists in the base (grep the base `:root` for `--brand-wash`, `--brand-bd`, `--well`).

- [ ] **Step 3:** Extract the AI DOM block (`<button class="ai-fab">` through `#ai-panel`'s close, including `#ai-indhint`) from the backup and insert it just before the main `<script>` (after the tooltip/`#tip` element if present in the base; else immediately before `<script>`).

- [ ] **Step 4 (verify):** Reload `http://localhost:8000/ai/index.html?v=2`. Assert:
  - `document.getElementById("ai-fab")` exists and is visible.
  - Clicking it toggles `#ai-panel` visible.
  - Light + dark: toggle theme, panel colors follow tokens.
  - Console clean. (No JS wiring yet — panel is inert; that's expected.)

- [ ] **Step 5:** Commit.
  Run: `git add app/ai/index.html && git commit -m "feat(ai): FAB, panel, and industry type-ahead DOM/CSS in the sub-app"`

---

### Task 4: Port the AI JS transport + prompt + chat UI (no board writes yet)

**Files:**
- Modify: `app/ai/index.html` (main `<script>`, before its `</script>`)

**Interfaces:**
- Consumes from backup: `AI_MODEL`, `aiState`, `componentCatalog`, `catalogFromArch`, `archAtomSet`, `industryPromptList`, `aiSystemPrompt(catalogText)`, `aiExtractJson`, `aiBridgeHealthy`, `aiCallBridge`, `aiCallApi`, the `aiEls`/`aiAddMsg`/`aiSetBusy`/`aiRefreshConn` chat UI, FAB/x/send/keydown/suggestion/clear/key-toggle wiring, and the type-ahead block (`IND_SYNONYMS`, `IND_STOP`, `aiIndustryMatches`, `aiIndHint`).
- Produces: a chat that connects (bridge `/health` → `/generate`), sends prompts, renders replies. `componentCatalog()` walks the base's `byId`. `industryPromptList()` grounds on `INDUSTRY_CATALOG` + `INDUSTRY_BUILT`.

- [ ] **Step 1:** Extract the whole AI JS block from `pre-upstream-backup:app/index.html` (from `const AI_MODEL` through the end of the AI wiring). Paste before the main `</script>` in `app/ai/index.html`.

- [ ] **Step 2:** Adapt `industryPromptList()` to the base's built-industry gate: enumerate `INDUSTRY_CATALOG` (`[id,label]`) and mark built via `INDUSTRY_BUILT` (Set from `architectures/manifest.json`), not a fully-populated `INDUSTRIES`. If `INDUSTRY_BUILT` is null (manifest missing), treat all catalog entries as candidates.

- [ ] **Step 3:** Keep `aiCallApi` at `max_tokens:16000` with the `stop_reason==="max_tokens"` throw. Keep `aiCallBridge` posting `{system,user,model}` to `/generate`.

- [ ] **Step 4:** `node --check` the extracted script body (extract main `<script>` contents to a temp `.js`, run `node --check`). Expected: OK, no syntax errors.

- [ ] **Step 5 (verify, real model):** Start the sub-app BACKEND (Task 6 backend) OR temporarily point at the existing bridge; simplest: defer live send to Task 7 and here assert only that the panel wires up:
  - Reload `?v=3`; type in the input → `#ai-indhint` shows industry matches when a known industry word is typed (e.g. "bank" → Banking chip).
  - Console clean. (Full send tested in Task 7 once board-write functions exist.)

- [ ] **Step 6:** Commit.
  Run: `git add app/ai/index.html && git commit -m "feat(ai): chat transport, prompt, type-ahead wired to the modular base"`

---

### Task 5: Board-write functions rebased onto the async modular model

**Files:**
- Modify: `app/ai/index.html` (main `<script>`)

**Interfaces:**
- Consumes: base primitives `boardSnapshot()`, `makeTab(id,name,text,snap)`, `selectTab(tab)`, `refSnap`, `resolveAtom(name)`, `build()`, `applyIndustry(id,remember)` (async), `loadIndustry(id)` (async), `INDUSTRIES`, `SCHEMA`, `paintRef`/`activeRef`/`clearReference`.
- Produces:
  - `applyIndustryTo(fields, id)` — pure transform (BASE reset of rails/top, medallion stage copy, industry overlay incl. `prioritiseUseCases` if present in base). Synchronous; assumes `INDUSTRIES[id]` is already loaded.
  - `async industryArch(id)` — `await loadIndustry(id)`, deep-clone `(refSnap || boardSnapshot())`'s `{bands,rails,top,cloud}`, run `applyIndustryTo(clone,id)`, set `clone.industry=id`, return snap. Never mutates the live board.
  - `filterArch(selected, base)` — `const src = base || refSnap || boardSnapshot();` prune to `selected`, return `{schema:SCHEMA, industry:(src.industry||"generic"), bands, rails, top, cloud}`.
  - `catalogFromArch(arch)` / `archAtomSet(arch)` — walk an arch object (shape identical to base's terse board).

- [ ] **Step 1:** Add `applyIndustryTo(fields,id)` by extracting the pure transform body from the base's `applyIndustry` (async, line ~3931). Copy only the field-mutation logic (rails/top reset from BASE, medallion stages over `fields.bands`, industry overlay); exclude side-effects (build, persist, tracking). Guard optional helpers: call `prioritiseUseCases(fields.top.secs)` only if that function exists in the base.

- [ ] **Step 2:** Add `async function industryArch(id)`: `if(!(await loadIndustry(id))) return null;` then deep-clone and `applyIndustryTo`. Return the snap tagged `industry:id`.

- [ ] **Step 3:** Add `filterArch(selected, base)` and `catalogFromArch`/`archAtomSet` verbatim from the backup, with `src = base || refSnap || boardSnapshot()`.

- [ ] **Step 4:** Add a `paintRef` `__tab__` guard: locate the base's `paintRef` (or the reference-highlight path); ensure a ref with `id==="__tab__"` does NOT dim the board (early-return before adding `.mapped`/`ref-mode`). If the base's `paintRef` already differs, add the guard at its top: `if(ref && ref.id === "__tab__") { /* show banner, no dim */ }`.

- [ ] **Step 5:** `node --check` the script body. Expected: OK.

- [ ] **Step 6 (verify):** Reload `?v=4`. From the console (DevTools MCP `evaluate_script`):
  - `await industryArch("banking")` returns an object with `industry:"banking"` and non-empty `bands`.
  - `archAtomSet(await industryArch("banking"))` is a non-empty Set including a banking-specific atom name.
  - The live board is UNCHANGED after these calls (still the reference board; `boardSnapshot().industry` unchanged).
  - Console clean.

- [ ] **Step 7:** Commit.
  Run: `git add app/ai/index.html && git commit -m "feat(ai): async industryArch + filterArch/catalogFromArch rebased on the lazy YAML model"`

---

### Task 6: FastAPI backend for the sub-app

**Files:**
- Create: `app/ai/app.py`, `app/ai/requirements.txt`, `app/ai/app.yaml`

**Interfaces:**
- Consumes: `pre-upstream-backup:app/app.py`, `:app/requirements.txt`, `:app/app.yaml`.
- Produces: `/health`, `POST /generate {system,user,model}->{text}` (or `{error}`), serves `app/ai/index.html` at `/`, and serves the shared `../architectures`, `../resources`, `../translations`, `../vendor`, `../assets`, `../arch_schema.js`, `../industry_icons.js` so the page's `<base href="../">` fetches resolve when the backend is the server.

- [ ] **Step 1:** Copy `git show pre-upstream-backup:app/app.py > app/ai/app.py`; `git show pre-upstream-backup:app/requirements.txt > app/ai/requirements.txt`.

- [ ] **Step 2:** In `app/ai/app.py` set `HTML_PATH = os.path.join(HERE, "index.html")` (serves `app/ai/index.html`). Keep `_content_text`, no-`temperature`, `max_tokens=16000`, `finish_reason=="length"` 502.

- [ ] **Step 3:** Add static routes for the shared parent dirs so `<base href="../">` works when this backend is the origin. Mount the parent `app/` dir:
  ```python
  from fastapi.staticfiles import StaticFiles
  PARENT = os.path.dirname(HERE)  # .../app
  # Serve shared assets from the parent app dir at the paths the page requests
  # relative to "../" (i.e. one level up from /ai/). Because <base href="../">
  # rewrites requests to "/architectures/...", mount those at root.
  for sub in ("architectures", "resources", "translations", "vendor", "assets", "share"):
      p = os.path.join(PARENT, sub)
      if os.path.isdir(p):
          app.mount("/" + sub, StaticFiles(directory=p), name=sub)
  ```
  Add explicit file routes for `/arch_schema.js` and `/industry_icons.js` (FileResponse from `PARENT`). Keep `/` , `/health`, `/generate` defined BEFORE the mounts so they win.

- [ ] **Step 4:** Create `app/ai/app.yaml` from the backup's (uvicorn command + `SERVING_ENDPOINT=databricks-claude-sonnet-5`), binding the platform port:
  ```yaml
  command: ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
  env:
    - name: SERVING_ENDPOINT
      value: databricks-claude-sonnet-5
  ```

- [ ] **Step 5 (verify):** From `app/ai/`, using the Arch2 venv (has fastapi/uvicorn/openai/databricks-sdk):
  Run: `DATABRICKS_PROFILE=arch-demo <venv>/bin/uvicorn app:app --port 8792` (background).
  - `curl -s localhost:8792/health` → `{"ok":true,...}`.
  - `curl -s localhost:8792/ | grep -c '<base href="../">'` → 1.
  - `curl -sI localhost:8792/architectures/banking.yaml` → 200 (shared template served).
  - `curl -sI localhost:8792/vendor/js-yaml.min.js` → 200.
  - `curl -s -X POST localhost:8792/generate -H 'content-type: application/json' -d '{"user":"say hi","system":""}'` → `{"text":"..."}`.

- [ ] **Step 6:** Commit.
  Run: `git add app/ai/app.py app/ai/requirements.txt app/ai/app.yaml && git commit -m "feat(app): FastAPI backend for app/ai serving the sub-app + shared templates + /generate"`

---

### Task 7: `aiCreateTab` + two-phase `aiSend`, end-to-end against the real model

**Files:**
- Modify: `app/ai/index.html` (main `<script>`)

**Interfaces:**
- Consumes: everything from Tasks 4–6.
- Produces:
  - `async aiCreateTab(r, baseArch)` — validate picks via `archAtomSet(baseArch)` when given else `resolveAtom`; `snap = filterArch(selected, baseArch||null)`; then base-native tab creation: `const id = "custom"+(++tabSeq); if(activeRef){clearReference(); refBanner.classList.remove("show");} const tab = makeTab(id, title, "", snap); selectTab(tab); persistTabs?.();` then set `activeRef = {id:"__tab__", name:title, blurb, map:notes}` and show the banner WITHOUT dimming. Return `{count, title, industry: baseArch?baseArch.industry:null}`. Zero resolved atoms → no tab.
  - `async aiSend()` — phase-1 detect on generic catalog (`componentCatalog()`), get `{reply,title,blurb,industry,components}`; if `INDUSTRY_BUILT?.has(result.industry)` (or catalog match) → `baseArch = await industryArch(indId)`, phase-2 re-select grounded on `aiSystemPrompt(catalogFromArch(baseArch))`, replace `result.components`; then `await aiCreateTab(result, baseArch)`.

- [ ] **Step 1:** Add `aiCreateTab` per the interface. Confirm the base's tab-id/seq mechanism (`tabSeq` or equivalent) and `persistTabs`/`persistCustom` names — grep the base; use its actual names. If the base uses `cloneCurrent()` semantics, mirror how it registers a custom tab.

- [ ] **Step 2:** Add the two-phase `aiSend`. Hoist `convo` so the phase-2 bridge call reuses context. Await the async `industryArch`.

- [ ] **Step 3:** `node --check`. Expected OK.

- [ ] **Step 4 (verify — real model, DevTools MCP against the Task 6 backend at :8792):**
  - **Banking prompt** ("Build an architecture for a retail bank doing fraud detection and regulatory reporting"): a new tab appears; `boardSnapshot().industry === "banking"`; board VISIBLE (`.board` height > 0); at least one atom present that is in `archAtomSet(await industryArch("banking"))` but not generically `resolveAtom`-able; the Reference tab still renders the upstream board when selected; console clean.
  - **IDEC-style heavy prompt** (12 data sources): generates cleanly, NO "unterminated string in JSON", console clean (confirms the max_tokens fix carried over).
  - **Generic prompt** ("a generic data platform"): single-phase; tab built from `refSnap`-equivalent; board visible.
  - **New-template reachability:** create `app/architectures/_smoketest.yaml` (copy banking.yaml, change name), add it to `architectures/manifest.json` as built; reload; confirm the AI/type-ahead can surface it and `await industryArch("_smoketest")` resolves via the shared dir — proving new templates need no sub-app code change. Then remove the smoketest file + manifest entry.

- [ ] **Step 5:** Commit.
  Run: `git add app/ai/index.html && git commit -m "feat(ai): generate industry-detected tabs via two-phase async selection"`

---

### Task 8: Docs, regression sweep, deploy

**Files:**
- Modify: `README.md`
- Create: `app/ai/README.md` (how to run the sub-app locally + deploy)

**Interfaces:** none new.

- [ ] **Step 1 (regression, in-browser):** With `app/ai/` backend running, confirm the base's own features are intact on BOTH `app/index.html` (unchanged) and `app/ai/index.html`: reference board builds; industry menu switches board (async `applyIndustry`); cloud/palette/shape/stage toggles; export YAML; `cloneCurrent` (+ tab); i18n language switch loads `translations/*`. Visible-atom assertions.

- [ ] **Step 2:** Add an "AI Architecture Assistant (`app/ai/`)" section to `README.md`: what it does, that it reuses the shared `architectures/*.yaml` templates via `<base href="../">` (new templates auto-available), the two-phase async industry detection, and the FastAPI backend + `SERVING_ENDPOINT`. Create `app/ai/README.md` with the exact local run command (`DATABRICKS_PROFILE=arch-demo uvicorn app:app --port 8000` from `app/ai/`) and the deploy note.

- [ ] **Step 3:** Commit.
  Run: `git add README.md app/ai/README.md && git commit -m "docs: document the app/ai AI assistant sub-app and shared-template reuse"`

- [ ] **Step 4 (deploy — surface to user, do not self-authorize IAM):** Sync the whole `app/` tree (base + `ai/` + shared templates) to the FEVM workspace app source path and deploy the `app/ai` app. Exact `databricks sync`/`apps deploy` commands and the target app name are confirmed with the user before running (the previous deploy targeted `arch-explorer` in `fevm-arch-explorer-demo`, profile `arch-demo`). Confirm "App started successfully"; report the URL for the user to verify under SSO.

- [ ] **Step 5:** Push `main` to `origin` (ashrafosman) after the user confirms the deploy is healthy.

---

## Self-Review
- **Spec coverage:** reset+backup (T1), scaffold+`<base>` shared reach (T2), CSS/DOM (T3), transport/chat/type-ahead (T4), async board-write functions (T5), backend serving sub-app + shared templates (T6), tab generation end-to-end + new-template reachability (T7), docs/regression/deploy (T8). All spec sections mapped.
- **Placeholder scan:** no TBD/TODO; each verify step lists concrete assertions; code shown for the non-obvious backend mount and `aiCreateTab` shape. The one deferred item (exact deploy commands/app name) is intentionally user-gated per the security constraint, not a placeholder.
- **Type consistency:** `industryArch` is async everywhere it's called (T5 defines, T7 awaits). `filterArch(selected, base)`, `aiCreateTab(r, baseArch)->{count,title,industry}`, snapshot `{schema,industry,bands,rails,top,cloud}` consistent T5↔T7. `loadIndustry(id)` awaited before any `INDUSTRIES[id]` read (constraint + T5/T7).
- **Key risk:** the base may name tab-seq/persist functions differently than the old fork — T7 Step 1 explicitly greps the base for the real names rather than assuming.
