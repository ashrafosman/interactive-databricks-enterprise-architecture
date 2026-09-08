# Re-port the AI Assistant onto upstream, as a shared-template sub-app — Design

**Date:** 2026-09-08
**Repo:** `ide-fork` (fork of amralieg/interactive-databricks-enterprise-architecture)

## Problem

The fork's `main` is **9 ahead / 24 behind** `upstream/main`. Those 24 upstream commits
are a structural rewrite: `app/index.html` shrank 2.9 MB → 793 KB by extracting all 63
industry architectures into **canonical YAML** (`app/architectures/*.yaml`), lazy-loaded
via `vendor/js-yaml.min.js` + `arch_schema.js`; plus an i18n translation layer
(`translations/<lang>/*.json`), social-share Open Graph cards (`share/`), and a 2025 logo
refresh. Our 9 commits (AI Architecture Assistant + FastAPI `/generate` backend + JSON
truncation fix + edit-mode-on-generated-tabs) live **inline in the old monolithic
`app/index.html`**. A plain merge collides irreconcilably in that one file.

## Goal

Adopt upstream's modern modular base, and re-apply the AI assistant as a **separate
sub-app** at `app/ai/` that **reuses the same architecture templates** — so any industry
YAML added later (upstream or ours) is automatically available to the AI app, with zero
copying.

## Approach (chosen)

**Reset `main` to `upstream/main`** (work preserved on a backup branch first), then build
`app/ai/` as a modular-based copy of upstream's `index.html` with the AI assistant grafted
on. `app/ai/` reaches shared templates via `<base href="../">`, so all relative fetches
(`architectures/<id>.yaml`, `resources/*.json`, `translations/*`, `vendor/js-yaml.min.js`,
`arch_schema.js`, `industry_icons.js`) resolve against `app/` — the single source of truth.

Rejected: (a) `git merge` + manual conflict resolution — the 2.9 MB-vs-793 KB index.html
conflict with incompatible data-loading models is unresolvable in practice; (b) sibling
`ai-app/` with backend-served shared assets — more backend plumbing than needed; (c)
symlinked templates — fragile across sync/deploy.

## Target layout

```
ide-fork/
  app/
    index.html            # upstream reference explorer — UNTOUCHED
    architectures/*.yaml   # 63 canonical templates — SHARED source of truth
    resources/*.json       # links/references/accelerators/connectors — SHARED
    translations/<lang>/*  # i18n — SHARED
    vendor/js-yaml.min.js   # SHARED
    arch_schema.js, industry_icons.js, assets/, share/  # SHARED
    ai/
      index.html          # modular base + AI assistant; <base href="../">
      app.py              # FastAPI: serves ai/index.html + /generate + proxies ../ assets
      requirements.txt
      app.yaml            # Databricks App manifest (uvicorn command, SERVING_ENDPOINT)
```

## Shared-template mechanism

`app/ai/index.html` starts as a byte copy of `app/index.html`, then:
- Insert `<base href="../">` in `<head>` so every relative fetch/`<script src>` resolves
  against `app/` (one dir up). This is what makes new templates instantly usable.
- Anchor-relative links that must point back into `ai/` (if any) are made explicit.

Because the base app lazy-loads industries from YAML, the AI app inherits that loader —
`loadIndustry(id)` fetches `architectures/<id>.yaml` from the shared dir. New YAML files
need no code change to be pickable by the AI.

## AI assistant adaptation (the real work)

Our AI code was written against the **old monolithic** data model (inline `ARCH`,
fully-populated `INDUSTRIES`, synchronous `applyIndustry`). Upstream's model is **async /
lazy**. Concrete deltas, against verified upstream anchors:

- `applyIndustry(id, remember)` is now **async** (upstream line ~3931) and `INDUSTRIES` is
  an initially-empty `{}` (line ~3898) populated by `loadIndustry(id)` (~line 4048). Our
  `industryArch(id)` / two-phase `aiSend` must **`await loadIndustry(id)`** before reading
  `INDUSTRIES[id]` or snapshotting an industry board.
- `INDUSTRY_BUILT` (manifest-driven Set, ~line 4077/4090) is the real "is this industry
  built" gate — `industryPromptList()` and industry detection ground on it + `INDUSTRY_CATALOG`.
- Board primitives are unchanged and reusable as-is: `boardSnapshot()` (~4950),
  `makeTab(id,name,text,snap)` (~5662), `selectTab(tab)` (~5607), `refSnap` (~4361),
  `resolveAtom(name)` (~4396), `build()` (~5109). Our `filterArch`/`aiCreateTab` rebase
  onto these exactly as they did in the last fork port (snap = `{schema,industry,bands,
  rails,top,cloud}`; `src = base || refSnap || boardSnapshot()`).
- Keep the `paintRef` `__tab__` guard and the edit-mode-on-generated-tabs behavior.
- Backend (`app.py`) is our existing one verbatim: `/generate` → `databricks-claude-sonnet-5`
  via OpenAI-compat client + WorkspaceClient OAuth, `_content_text()` reasoning-block
  flattening, `max_tokens=16000`, `finish_reason=="length"` clean-error. Serves
  `ai/index.html`; add a route to proxy `../architectures`, `../resources`, `../translations`,
  `../vendor`, `../assets`, `arch_schema.js`, `industry_icons.js` when running standalone.
- Frontend transport (`aiCallApi`) keeps `max_tokens:16000` + `stop_reason=="max_tokens"`.

## Deployment

`app/ai/` deploys as its own Databricks App (its `app.yaml` runs `uvicorn app:app`). For
the shared assets to be present in the deployed app, the sync step includes `app/` (so the
templates ship alongside), OR the backend proxies them from a co-located `app/`. Decision
deferred to the plan's deploy task; default: sync the whole `app/` tree and serve
`ai/index.html` with `<base href="../">` resolving within the same deployed source root.

## Verification

Per fork convention: run the sub-app backend locally
(`DATABRICKS_PROFILE=arch-demo uvicorn app:app` from `app/ai/`), drive via Chrome DevTools
MCP with a cache-busting `?v=`. Assert against the **visible** board
(`getBoundingClientRect().height>0`), not bare `.atom` counts. Test matrix:
- Reference board loads (upstream modular explorer intact).
- A built-industry prompt (e.g. banking) → `await loadIndustry` → new tab from that
  industry's YAML template, board visible, industry-specific atom present.
- IDEC-style data-source-heavy prompt → no JSON truncation (the fix), clean console.
- Adding a hypothetical new `architectures/foo.yaml` is reachable by the AI with no code
  change (spot-check the fetch path resolves through `<base href="../">`).
- Regression: upstream `app/index.html` unchanged and still works.

## Non-goals

- No changes to upstream's `app/index.html` behavior.
- No re-monolithizing the data. The AI app consumes the modular templates as-is.
- No new industries authored in this effort.
```
