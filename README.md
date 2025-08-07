# Project Specification – Neovim Config PWA (Native‑First Neovim Config Generator)

**Owner:** (TBD)
**Prepared for:** Implementation contractor
**Last updated:** 2025‑08‑07

---

## 0) Executive Summary

Build an installable **Progressive Web App** that lets users compose and export a **native‑only Neovim configuration** (Lua) with zero plugins by default. The app guides users through options, keymaps, and autocmds; previews the emitted `init.lua`; and supports exporting a single file or a small module tree. Later phases add import of existing configs, conflict warnings, optional “plugin layer” suggestions, and an (off‑by‑default) **legacy Vimscript output** mode.

**Deliverable of Phase 1:** A production‑grade PWA that generates a working, clean `init.lua` using only built‑in Neovim features, with local persistence and a basic preset system.

---

## 1) Goals & Non‑Goals

### Goals (Phase 1)

* Generate **native Neovim** config (Lua) without external plugins.
* Provide **presets** (Minimal, Vim‑ish, IDE‑ish) and a **from‑scratch** builder.
* **Live preview** of emitted Lua; **download** as `init.lua`.
* Optional **split output** layout (`lua/app/{options.lua,keymaps.lua,autocmds.lua}`) in addition to single file.
* **Local persistence** (IndexedDB) for user selections and custom presets.
* **Installable PWA** (manifest, basic SW; offline app shell).

### Non‑Goals (Phase 1)

* Installing LSP servers, managing plugins, Treesitter, themes, etc.
* Full import/parse of arbitrary `init.vim`/`init.lua` (defer to Phase 2).
* **Legacy Vimscript output** (defer to Phase 3+).

---

## 2) Users & Use‑Cases

* **New NVim users**: click through sane defaults, export a clean config.
* **Power users**: compose opinionated native config, keep it portable.
* **Instructors/teams**: share presets to standardize base configs.

**Primary workflows**:

1. Choose preset → tweak → export.
2. Start from scratch → tweak → export.
3. (Phase 2) Import existing config → map → review unmapped → export.

---

## 3) Product Requirements

### 3.1 Feature List (Phase 1)

* Options panels: Numbers, Indentation, Search, Splits, Clipboard, Colors, Statusline, Misc.
* Keymaps panel: a small, curated set of native mappings (e.g., netrw open, terminal toggle, quickfix nav).
* Autocmds panel: yank‑highlight, trailing whitespace toggle (optional), filetype‑sensible tweaks.
* Presets: Minimal, Vim‑ish (closer to legacy defaults), IDE‑ish (ergonomic but native).
* Live Lua preview with syntax highlighting (client‑only is fine).
* Export: download single file; optional split layout.
* Local persistence of last selections and saved presets.
* Path guidance for **Windows** and **Unix** targets.

### 3.2 Constraints & Notes

* Output must be **deterministic**, stable ordered, and lint‑friendly.
* No telemetry. All data local unless user explicitly exports/imports.
* Avoid requiring network at runtime (except optional preset sharing in later phases).

---

## 4) Architecture & Tech Choices

* **Framework:** SvelteKit + TypeScript.
* **Styling:** TailwindCSS (utility‑first, consistent spacing/typography).
* **Validation:** Zod for schema typing/guards.
* **State:** Svelte stores; IndexedDB for persistence (via idb or minimal wrapper).
* **Build:** Vite; adapter‑auto.
* **Testing:** Vitest (unit), Playwright (e2e for export UX). Optional snapshot tests for Lua output.
* **Formatting/Lint:** Prettier + ESLint; commit hooks via simple npm script.

**High‑level:**

* **Schema layer**: JSON/TS definitions of options, keymaps, autocmds, presets.
* **Generator**: Pure functions `selections -> OutputFiles` (no DOM), deterministic order.
* **UI**: Panels bound to schema; generator renders to preview and download.
* **PWA**: Manifest + Service Worker caching shell and schema; offline by default.

---

## 5) Data Model & File Emission

### 5.1 Types

```ts
// Core types
export type OptionValue = boolean | number | string | string[];
export interface OptionSpec { id: string; type: 'boolean'|'number'|'string'|'enum'|'stringArray'; default: any; enumValues?: string[]; label: string; description?: string; min?: number; max?: number; }
export interface KeymapSpec { mode: 'n'|'i'|'v'|'t'|string[]; lhs: string; rhs: string | 'LUA_FN'; desc?: string; silent?: boolean; noremap?: boolean; condition?: string; }
export interface AutocmdSpec { event: string; pattern?: string; body: string; group?: string; }
export interface Catalog { options: OptionSpec[]; keymaps: KeymapSpec[]; autocmds: AutocmdSpec[]; }
export interface Selections { options: Record<string, OptionValue>; keymaps: KeymapSpec[]; autocmds: AutocmdSpec[]; legacyMode?: boolean; }
```

### 5.2 Generation Rules

* Render headers and leader settings first.
* Emit options ordered by category then id.
* For booleans: `o.<name> = true|false`.
* For numbers: `o.<name> = <num>`.
* For strings/enums: `o.<name> = "<escaped>"`.
* Keymaps: generate `local map = vim.keymap.set` then 1 line per mapping.
* Autocmds: create a named augroup once; add autocmds with `callback = function() ... end`.
* Escape `'` and `\` correctly in strings.
* Ensure newline at EOF.

### 5.3 Output Layouts

* **Single file:** `init.lua` (default).
* **Split layout:**

  * `init.lua` → `require('app.options')`, `require('app.keymaps')`, `require('app.autocmds')`
  * `lua/app/options.lua`, `lua/app/keymaps.lua`, `lua/app/autocmds.lua`

---

## 6) UX & Screens

### 6.1 Navigation

* `/builder` is the primary route. Keep left panel (controls) + right panel (preview).
* Top bar: Preset picker (dropdown), Save Preset, Reset, Install (PWA prompt).

### 6.2 Panels (Phase 1)

* **General**: Mapleader (`<space>` default), Maplocalleader (`,` default).
* **Numbers**: number, relativenumber.
* **Indent**: expandtab, shiftwidth, tabstop, smartindent (optional).
* **Search**: ignorecase, smartcase.
* **Splits**: splitright, splitbelow.
* **Clipboard**: clipboard=`unnamedplus` (toggle).
* **Colors**: termguicolors.
* **Statusline**: laststatus (0‑3) and statusline format string.
* **Keymaps**: netrw toggle (`<leader>e`), terminal toggle (`<A-`>`), quickfix (`\[q`/`]q\`) (optional).
* **Autocmds**: TextYankPost highlight.

Each control displays a short help tooltip with *native help tags* (e.g., `:h 'ignorecase'`).

### 6.3 Preview

* Right pane shows generated `init.lua`. Provide Copy and Download buttons. Simple code coloring is sufficient; no server.

---

## 7) Platform Considerations

* **Windows path**: `%LOCALAPPDATA%\nvim\init.lua`
* **Unix path**: `~/.config/nvim/init.lua`

Provide per‑OS placement instructions after export, plus a PowerShell button to open the folder (non‑executing in web; show a copyable command).

---

## 8) Accessibility & i18n

* Use semantic elements and visible focus styles.
* Keyboard‑first navigation; all controls reachable by tab.
* ARIA labels for toggles/inputs.
* (Phase 2) Externalize strings for localization; default en‑US.

---

## 9) Performance

* Keep schema small and statically imported.
* Lazy‑render preview string; recompute only on selection changes.
* Service Worker caches app shell; no external fonts by default.

---

## 10) Privacy & Security

* No telemetry.
* Data persists locally via IndexedDB; explicit user action for export/import.
* CSP: default SvelteKit; avoid inline scripts except necessary event handlers.

---

## 11) Testing Strategy

### 11.1 Unit

* Generator snapshot tests for each preset (stable output).
* Escaping edge cases (quotes, backslashes, `%` in statusline).
* Option guards (e.g., smartcase only useful if ignorecase true → warning text exists).

### 11.2 E2E (Playwright)

* Build → open `/builder` → toggle options → preview updates → download → file content matches snapshot.
* PWA install prompt appears; app loads offline after first visit.

### 11.3 Manual QA Matrix

* Browsers: Chromium, Firefox, Safari (latest).
* OS guidance correctness (Windows/Unix).
* NVim smoke test: `nvim --clean -u <downloaded/init.lua> +qa` prints no errors.

---

## 12) Delivery Plan & Milestones

### Phase 1 (2–3 weeks)

* **W1**: Project setup, schema + generator, Minimal preset, basic UI panels, live preview.
* **W2**: Presets (3), split‑output option, download flow, IndexedDB persistence, PWA manifest + SW shell.
* **W3**: Tests (unit/e2e), docs, polish, release.

### Phase 2 (2–3 weeks)

* Import existing `init.vim`/`init.lua` (best‑effort map + unmapped report).
* Conflict/warning system (non‑blocking).
* OS placement helper UI; copyable commands.

### Phase 3 (3–4 weeks)

* Advice cards: native workflows vs plugin alternatives (no plugin install).
* **Legacy (Vimscript) output** behind toggle.
* Share presets via URL or gist (user‑initiated, no secrets).

---

## 13) Acceptance Criteria (Phase 1)

* User can open `/builder`, pick a preset, modify options, and **download** a working `init.lua`.
* Output passes `nvim --clean -u init.lua +qa` without errors.
* App installs as a PWA and launches offline (previously visited).
* Selections persist across reloads.
* Codebase includes unit tests for generator and at least one Playwright e2e.

---

## 14) Repo Layout & Tooling

```
/ (repo root)
  src/
    lib/
      schema/        # types + catalog
      generator/     # emit + render
      presets/       # built‑in presets
      stores/        # svelte stores (ui, selections)
      ui/            # small reusable components
    routes/
      builder/
  static/
    manifest.webmanifest
  tests/             # vitest + playwright
  package.json
  svelte.config.js
  tailwind.config.js
  vite.config.ts
```

**Scripts**

* `dev`: sveltekit dev
* `build`: sveltekit build
* `preview`: sveltekit preview
* `test`: vitest run
* `test:e2e`: playwright test

---

## 15) Coding Standards

* TypeScript strict mode on.
* No DOM‑dependent code in the generator.
* Keep emitters small, pure, and covered by tests.
* Avoid clever string concatenation; centralize escaping helpers.

---

## 16) Detailed Spec – Options, Keymaps, Autocmds (Phase 1 Catalog)

### 16.1 Options (initial)

* `number: boolean` (default: true)
* `relativenumber: boolean` (false)
* `expandtab: boolean` (true)
* `shiftwidth: number` (2, 1..12)
* `tabstop: number` (2, 1..12)
* `ignorecase: boolean` (true)
* `smartcase: boolean` (true)
* `splitright: boolean` (true)
* `splitbelow: boolean` (true)
* `clipboard: enum('','unnamedplus')` ('unnamedplus')
* `termguicolors: boolean` (true)
* `laststatus: number` (3)
* `statusline: string` (default string with `%` codes)

### 16.2 Keymaps (initial)

* `<leader>e` → `:Lexplore<CR>` (netrw)
* `<A-`>\` (Alt+backtick) → toggle terminal split (Lua fn)
* (Optional) Quickfix: `[q` / `]q` → `:cprev` / `:cnext`

### 16.3 Autocmds (initial)

* `TextYankPost` → `vim.highlight.on_yank()`

---

## 17) UI Copy & Help Hints

* Each control has a one‑line description + a `:help` reference (e.g., `:h 'smartcase'`).
* Statusline input includes a link to `:h statusline`.
* Clipboard toggle warns about WSL/remote edge cases.

---

## 18) Edge Cases & Validation

* Show a *hint* (non‑blocking) if `smartcase=true` while `ignorecase=false`.
* Escape `%` in statusline correctly; preserve literal `%%`.
* Prevent NaN on numeric inputs; clamp to min/max.

---

## 19) Roadmap Notes (Beyond Phase 1)

* Config **import** with line‑level mapping + “unmapped” report.
* **Advice cards** that list native alternative commands/workflows vs popular plugin equivalents (no auto‑install).
* **Legacy Vimscript output** toggle: new emitter that mirrors selections into `set`, `nnoremap`, and `autocmd` (with feature detection where needed). Will require mapping table + test suite.
* Optional **share**: serialize selections to URL (compressed) or create gists via user token.

---

## 20) Example Generated Output (Minimal Preset)

```lua
-- Generated by Neovim Config PWA (native only)
vim.g.mapleader = ' '
vim.g.maplocalleader = ','

local o, g = vim.opt, vim.g
o.number = true
o.relativenumber = false
o.expandtab = true
o.ignorecase = true
o.smartcase = true
o.splitright = true
o.splitbelow = true
o.termguicolors = true
o.shiftwidth = 2
o.tabstop = 2
o.laststatus = 3
o.clipboard = "unnamedplus"
o.statusline = "%#StatusLine# %f %m %= %y %p%% %l:%c "

local map = vim.keymap.set
map({'n','t'}, '<A-`>', function()
  local bt = vim.bo.buftype
  if bt == 'terminal' then vim.cmd('hide') else vim.cmd('botright split | terminal') end
end, { silent = true, noremap = true, desc = 'Toggle terminal' })
map('n', '<leader>e', ':Lexplore<CR>', { silent = true, noremap = true, desc = 'Open netrw' })

local aug = vim.api.nvim_create_augroup('AppBasics', { clear = true })
vim.api.nvim_create_autocmd('TextYankPost', { group = aug, callback = function() vim.highlight.on_yank() end })
```

---

## 21) Handoff Checklist

*

---

## 22) Risks & Mitigations

* **String escaping bugs** → Centralized helpers + tests.
* **NVim version drift** → Document minimum NVim (0.9+), test on 0.9/0.10 locally.
* **Import complexity** → Phase 2 only; treat as best‑effort with explicit unmapped list.

---

## 23) License & Contribution

* License: MIT.
* PRs require lint + tests green; snapshot updates reviewed.
* Issue templates: bug/feature/question.

---

## 24) Kickoff Instructions (Contractor)

1. Scaffold SvelteKit repo and install dependencies.
2. Implement schema and minimal generator with snapshot tests.
3. Build `/builder` UI with panels and preview.
4. Implement download + split output.
5. Add IndexedDB persistence.
6. Add PWA manifest + SW (cache shell, schema).
7. Polish, docs, and deliver Phase‑1 release build plus test report.

