# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Standalone community backport of the RestedXP Guides addon to the **WoW 3.3.5a client (build 12340, Interface 30300)**, validated against **AzerothCore**. Code is Lua 5.1 targeting the stock 3.3.5a API; it must not depend on Questie, ElvUI, WeakAuras, etc. Only `RXPGuides.toc` ships — Cataclysm/MoP/Retail/SoD content and newer-client manifests are intentionally excluded, although Classic/TBC/WotLK data and dormant feature sources remain in the repo.

## Validation (the "test suite")

There is no build step. CI (`.github/workflows/validation.yml`, Ubuntu + `lua5.1` + `pwsh`) runs these from the repo root; run the same locally before committing:

```powershell
./tools/Validate-Runtime335.ps1 -RequireLua   # TOC/XML manifest resolution (case-sensitive), luac syntax check, then `lua tests/run.lua <root>`
./tools/Validate-Guides335.ps1                # guide structure; pinned against tests/runtime-surface.json
./tools/Validate-GuideTranslations335.ps1 -CoreOnly
./tools/Validate-QuestFlow335.ps1             # quest prerequisite/flow checks against AzerothCore-derived data
./tools/Validate-Talents335.ps1
./tools/Validate-Roadmap335.ps1
./tools/Validate-Privacy335.ps1
```

- The Lua unit/integration tests are `tests/run.lua` (which also loads `tests/guide-loading.lua`), invoked as `lua5.1 tests/run.lua .`. They mock the WoW API themselves; there is no per-test filter — failures print `FAIL: ...` to stderr.
- Paths are resolved **case-sensitively** because CI/release run on Linux; a wrong-case path in the TOC or an XML `<Script file=...>` passes on Windows but fails CI.
- `tests/runtime-surface.json` pins guide keys/content integration. Only update it for an intentional key migration or content integration (changing guide keys resets players' saved progress).
- CI also checks out the `localizations` branch commit pinned in `LOCALIZATIONS.lock` and validates translation packs; see `LOCALIZATION.md`. Translation tooling lives on that branch, not `main`.

In-game: `/reload` suffices for Lua-only edits; fully restart the client after changing TOC, XML manifests, libraries, fonts, or textures.

## Architecture

### Load order is the dependency graph
`RXPGuides.toc` is authoritative and order-sensitive. Every file begins with `local _, addon = ...` and extends the shared `addon` table. Key TOC constraints:
- `Compat\Bootstrap.lua` (+ Timer/Map/Inventory facades) **must load first**: it polyfills modern `C_*` / `Enum` APIs so upstream code's `C_Foo and C_Foo.Bar or _G.Bar` idiom works. It never clobbers an existing API (`def` only sets missing keys) and bails if the interface isn't 30300. Map/coordinate APIs are shimmed separately via Astrolabe in `libs\HBD335\`.
- The 3.3.5a TOC parser needs **backslash paths**, and nested XML `<Script>` paths resolve relative to the XML file's folder. That is why DB files are listed directly in the TOC and why `GuideList_335.xml` / `Talents_wotlk_335.xml` sit at the repo root (their `Guides\...` paths resolve from the root). Adding a guide = adding a `<Script>` line to `GuideList_335.xml`.

### Core runtime (`Core/`)
- `Services.lua` — `addon.services:Register(name, instance, legacyAliases)` / `Require(name)`. Legacy aliases expose the service as `addon.<alias>` for upstream code.
- `Runtime.lua` — `addon.runtime:Register{id, depends, phase, initialize, enable, disable, optional}`; subsystems are topo-sorted by `depends` and initialized at `ADDON_LOADED`. Optional subsystems run supervised through `addon.roadmap:RunOptional` so a failing feature disables only itself.
- `Scheduler.lua` — first-party timers; do not rely on global `C_Timer` (private-server UI packs install broken facades — tests assert RXP never calls it).
- `Addon.lua` — main addon object, events, slash commands (`/rxp`, `/rxpg`, `/rxpguides`).

### Guide engine (`Guide/`)
- Guides are Lua files under `Guides/` calling `RegisterGuide` with RestedXP guide text; `Loader.lua` parses them. `Parser.lua`, `Registry.lua`, `State.lua`, `Conditions.lua` are thin service facades over legacy globals (`addon.ParseGuide`, `addon.applies`, …).
- Directive implementations (`.accept`, `.turnin`, `.goto`, …) all live in the large `Guide/Directives/Handlers.lua` (registered into `addon.functions`). The small per-domain files (`Quest.lua`, `Navigation.lua`, …) only catalog them via `addon.directives:RegisterDomain`. A new directive must be added to a domain, or `ValidateLegacySurface` flags it as uncatalogued.
- Quest automation is built around 3.3.5a's title-based gossip/quest events (`QuestAutomation.lua`, `QuestAcceptState.lua`, `QuestRewardTransaction.lua`, `AutomationOrder.lua`).

### Guide content
- **Validated routes** (default) vs **Original** upstream snapshots (`Guides/Original/`) use separate guide keys/groups so progress doesn't collide.
- Many guides are generated: `tools/Build-*335.ps1` convert upstream sources (e.g. `Build-WotLKGuides335.ps1` converts Zygor Remaster routes from a sibling `..\ZygorGuidesViewerRM` checkout into `Guides/WotLK`) and generate DB tables (`questPrerequisites_335.lua`, `guideEnglishNames_335.lua`, dangerous mobs, location locales). Prefer fixing the generator plus its output over hand-editing only the generated file.
- `DB/wotlk/*_335.lua` are backport-specific data; unsuffixed DB files are upstream-derived.

### Features / UI / Localization
- `Features/` holds player features (targeting via secure buttons + nameplate scanning, item upgrades, inventory/junk, talents, speedrun suite, etc.); `UI/` holds frames. Roadmap services and their privacy/safe-mode rules are described in `FEATURES_335.md` — reports/exports must never include player/realm names, account IDs, or full GUIDs (enforced by `Validate-Privacy335.ps1`).
- Translations are display-only: they must never alter directives, IDs, coordinates, targets, guide keys, or anything feeding quest automation. Locale packs ship as separate `RXPGuides_Locale_<locale>` companion addons built by `tools/Build-LocalePackages335.sh`.

## Constraints to respect
- Protected actions (targeting, item use, equipping) must go through secure buttons and hardware input; nothing may run them automatically in combat.
- Don't guess on ambiguous legacy client data (e.g. taxi destinations that don't resolve uniquely) — skip automation instead.
- Keep existing guide keys and SavedVariables structures (`RXPData`, `RXPDB`, `RXPSettings`, `RXP335`, `RXPC*`) stable so updates don't reset character progress.
- `ARCHITECTURE.md` referenced in the README is gitignored (local notes); it may not exist.
