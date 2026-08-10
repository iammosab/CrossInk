# CrossInk — Shared Agent Guide

This is the canonical repo instruction file.
`CLAUDE.md` should point here so Codex and Claude read the same guidance.

Project: Open-source e-reader firmware for ESP32-C3 and ESP32-S3 devices.

This repo is a personal fork of [CrossPoint Reader](https://github.com/crosspoint-reader/crosspoint-reader) focused on typography and reading stats. It does not accept pull requests; upstream changes are pulled in by merging/cherry-picking from CrossPoint, which is why the Git Workflow section below emphasizes upstream PR references.

## Core Rules

- Role: Senior Embedded Systems Engineer for ESP-IDF / Arduino-ESP32 work.
- Support both constrained ESP32-C3 devices and PSRAM-equipped ESP32-S3 devices. Keep shared code safe for the C3 unless it is explicitly capability-gated; stability beats features.
- Cite file paths and line numbers before proposing non-trivial changes.
- Do not assume ESP-IDF or SDK API availability. Verify in `freeink-sdk/` or the live code.
- Do not claim performance or memory wins without explaining the mechanism, such as reduced heap churn, flash vs DRAM placement, or smaller stack use.
- Justify new heap allocations or explain why stack/static storage is not suitable.
- Explain fixes in plain language where possible, ideally in terms a Node / React developer would follow.
- After proposing or making a fix, say how to verify it on hardware.

## Persistent Context

- Read `.claude/CONTEXT.md` at session start for durable repo-specific gotchas.
- Keep `.claude/CONTEXT.md` short. Add only reusable findings, not turn-by-turn history.

## Repo Skills

- Do not read every `.claude/skills/*/SKILL.md` at session start.
- Use this section as an index. Read a local skill only when the task clearly matches its folder name or purpose.
- Current local skills:
  - `control-flow-clarity`: simplify confusing logic without behavior changes.
  - `refactor-for-review`: small refactors intended to reduce review risk.
  - `hal-and-abstractions`: HAL boundaries and platform abstraction work.
  - `heap-discipline`: memory allocation, lifetime, and fragmentation-sensitive work.
  - `scope-discipline`: keep changes narrow and avoid unrelated cleanup.
  - `custom-fonts`: font generation, conversion, and SD/built-in font work.
- Treat these as task-specific playbooks layered on top of this guide. If a skill conflicts with this file, prefer `AGENTS.md` and note the conflict.

## Hardware Constraints

- ESP32-C3 targets (Xteink X3/X4): single-core RISC-V at 160 MHz, no PSRAM, and about 380 KB usable internal RAM.
- ESP32-S3R8 targets (Seeed reTerminal Sticky/Xteink X4 Pro): dual-core Xtensa at up to 240 MHz with 8 MB PSRAM. PSRAM is slower than internal DRAM and is not suitable for every DMA, ISR, or latency-sensitive buffer.
- Current displays use an 800x480 1-bit e-ink framebuffer: `800 * 480 / 8 = 48000` bytes. Use runtime renderer dimensions because orientation and future device profiles may differ.
- Use one framebuffer only. C3 targets keep it in internal RAM; current S3 targets place it in PSRAM via `FREEINK_FB_PSRAM`.
- Storage is exposed through SdFat, but the transport is device-specific (SPI SD on X3/X4/Sticky and SDMMC on X4 Pro). On real hardware, only one reader can hold a file open at a time.

## Architecture Overview

Deeper reading: `docs/development/architecture.md` (system and reader-pipeline diagrams), `docs/activity-manager.md`, `docs/data-cache.md`, `docs/file-formats.md`, `docs/epub-render-modes.md`, `docs/i18n.md`.

- Entry point: `src/main.cpp` defines the global singletons (`GfxRenderer renderer`, `MappedInputManager mappedInputManager`, `ActivityManager activityManager`) and owns cross-cutting behavior no single activity can: boot ordering, silent-restart tokens (reboot to reclaim heap, then land on a target screen), global power-button actions (`src/GlobalActions.h`), auto-sleep timing, and deep sleep entry.
- Activity model (`src/activities/`): Android-style single-screen manager. `ActivityManager` holds one current `Activity` plus a back stack. `replaceActivity()` / `pushActivity()` / `popActivity()` only record a pending transition that is applied at the end of `ActivityManager::loop()`, so an activity can navigate from inside its own callback without deleting itself mid-call. One shared FreeRTOS render task serves the whole app: activities call `requestUpdate()` and the task invokes `currentActivity->render()` under the global `RenderLock`; activities never own render tasks. Sub-flows return data via `startActivityForResult()` / `setResult()` / `finish()`.
- Reader pipeline (`lib/Epub`): `ReaderActivity` sniffs the file format (EPUB / XTC / TXT / BMP) and replaces itself with the matching reader. For EPUB: `Epub` parses container/OPF/TOC/CSS once and caches the result (`book.bin`); a `Section` (one spine item) streams its XHTML through `ChapterHtmlSlimParser` (expat SAX) into laid-out, serialized `Page` objects in `sections/<n>.bin`. Section builds are incremental and resumable (`startBuild()` / `buildSomeMore()`), writing to a `.part` file swapped atomically on commit. `EpubReaderActivity` holds one `Section` at a time; layout inputs come from `CrossPointSettings::readerRenderSpec()`, and a fallback ladder of `EpubRenderMode`s handles chapters that fail under memory pressure. Page turns are cache reads, never re-parses.
- Rendering: `GfxRenderer` (`lib/GfxRenderer/`) owns the single 1-bit framebuffer, orientation handling, and the font registry over `HalDisplay`. `EpdFont` / `EpdFontFamily` (`lib/EpdFont/`) supply glyphs: built-in fonts compiled into flash, `.cpfont` SD fonts loaded one point size at a time.
- HAL (`lib/hal`): the only code allowed to include SDK classes. App code reaches hardware through singletons: `display` (`HalDisplay`), `gpio` (`HalGPIO`), `Storage` (`HalStorage` / `HalFile` over SdFat), `powerManager`, `halClock`, `halTiltSensor`, `Frontlight`, plus `HalSystem` and `HalSpiBus`. `freeink-sdk/` is a git submodule whose libs are symlinked individually in `platformio.ini`. The simulator envs replace this whole layer (`lib_ignore = hal`, `-DSIMULATOR`) with SDL-backed mocks from the `crossink-simulator` dependency.
- Persistence: JSON-backed stores derive from `PersistableStore<T>` (`lib/Serialization/`) — Meyers singleton with mutex-guarded `saveToFile()` / `loadFromFile()`; ArduinoJson template machinery is deliberately instantiated once in `PersistableStore.cpp` to keep flash cost flat. Members include `CrossPointSettings` (`SETTINGS`), `CrossPointState` (`APP_STATE`), `RecentBooksStore`, `WifiCredentialStore`, and `OpdsServerStore`, all under `/.crosspoint/`. Per-book binary stores (`BookmarkStore`, `ClippingStore`, progress, stats) use their own files; see `docs/data-cache.md`.
- Network (`src/network/`): `CrossPointWebServer` (plus WebSockets and WebDAV handlers) serves the generated web portal; OTA lives in `OtaUpdater` / `OtaBootSwitch` / `FirmwareFlasher` and is excluded from simulator builds.
- i18n: `tr(STR_*)` indexes flat per-language `const char*` tables in flash, generated by `scripts/gen_i18n.py` (a PlatformIO pre-script) from `lib/I18n/translations/*.yaml`; English is the fallback for missing keys.
- Build hooks: `platformio.ini` `extra_scripts` run codegen before every build (`build_web.py`, `gen_i18n.py`, `git_branch.py`, library patch scripts) and size/packaging checks after.

## Resource Rules

1. Keep local stack usage small. Anything meaningfully larger than 256 bytes should be justified.
2. Avoid repeated heap churn in loops. Allocate once in `onEnter()`, reuse, and free in `onExit()`.
3. Large constant tables should be `static const` so they live in flash, not DRAM.
4. Avoid `std::string` and Arduino `String` in hot paths. Prefer `string_view`, `char[]`, and `snprintf`.
5. All user-facing UI strings must use `tr(STR_*)`. Logs may be hardcoded.
6. Prefer `constexpr` for compile-time constants.
7. Reserve `std::vector` capacity before push loops.
8. Debounce persistent writes. Do not write progress on every page turn.
9. `new` is not nothrow on ESP32. With exceptions disabled, bare `new` calls `abort()` on allocation failure instead of returning `nullptr`. Use `new (std::nothrow)` or `makeUniqueNoThrow<T>()` from `lib/Memory/Memory.h` for fallible allocations.
10. Prefer `makeUniqueNoThrow<T>()` / `makeUniqueNoThrow<T[]>()` for owned heap allocations so cleanup is automatic on early returns.
11. Use raw `malloc` or `new (std::nothrow)` only when a C or SDK API takes ownership; add a short comment explaining that ownership transfer.
12. Treat PSRAM as a device capability, not a universal assumption. Keep shared paths within C3 limits or gate S3-only allocations behind the relevant board/capability macro, and handle PSRAM allocation failure.

## HAL And Platform Rules

- Use HAL classes, not SDK classes, in app code.
- File I/O uses `FsFile`, not Arduino `File`.
- Always close files explicitly.
- Use `MappedInputManager::Button::*` enums for button logic.

## C++ / Embedded Gotchas

- `string_view::data()` is not null-terminated. Do not pass it directly to C APIs.
- ISR handlers need `IRAM_ATTR`, and ISR-read data must be in DRAM, not flash-only storage.
- Never call `xSemaphoreTake()` from an ISR. Use ISR-safe give APIs.
- Do not cast unaligned `uint8_t*` data to wider pointer types. Use `memcpy`.
- No exceptions. No `abort()`. Log before returning failure.
- Avoid `std::function` in hot paths and library code; prefer function pointers or a small context/callback struct.
- Keep template use deliberate. If a template is needed in shared code, consider explicit instantiation in a `.cpp` file to avoid repeated binary growth.

## Error Handling

- Prefer `LOG_ERR(...)` plus `return false` for recoverable failures.
- Prefer `LOG_ERR(...)` plus a known fallback when the app can continue safely.
- Use `assert(false)` only for truly impossible fatal states.
- Use `ESP.restart()` only for intentional recovery flows, such as completing OTA.
- Always log before returning failure from allocation, file, parse, network, or hardware paths.

## Activity Lifecycle

- Activities are heap-allocated and deleted on exit.
- Allocate long-lived buffers and tasks in `onEnter()`.
- Free resources in reverse order in `onExit()`.
- Delete FreeRTOS tasks before the activity is destroyed.
- Close open file handles in `onExit()`.
- Typical task stacks:
  - 2048 bytes for simple rendering work
  - 4096 bytes for network or EPUB parsing work

## UI And Input

- Do not hardcode screen dimensions like `800` or `480`; use renderer dimensions and orientation helpers.
- Use `renderer.getOrientedViewableTRBL()` for layout that must stay inside usable bezel-safe bounds.
- Use logical `MappedInputManager::Button::*` values in activities; raw hardware button indices belong only in button-mapping code.
- Route UI drawing through `UITheme` / `GUI` where practical so fonts, spacing, and orientation behavior stay consistent.
- User-facing text must be translated with `tr(STR_*)`; logs can remain hardcoded.

## Build And Verification

- PlatformIO is the source of truth. Personal overrides belong in `platformio.local.ini`.
- Host environment may be macOS, Linux, WSL, or Windows Git Bash. Check `uname -s` before recommending platform-specific shell commands.
- Logging uses `LOG_INF`, `LOG_DBG`, and `LOG_ERR`.
- The simulator env in this repo is `simulator`.
- For simulator work, build from this firmware repo unless the change belongs in `crossink-simulator` itself.
- Common validation commands:
  - `pio run -e simulator` for simulator-facing UI/reader work; add `-t run_simulator` to launch the SDL window.
  - `pio run -e default` for the ESP32-C3 X3/X4 firmware.
  - `pio run -e sticky` for the ESP32-S3 Sticky firmware.
  - There is no `x4-pro` hardware env in this fork; the X4 Pro exists only as the `x4-pro-simulator` env.
  - `pio check -e default --fail-on-defect low --fail-on-defect medium --fail-on-defect high` for static analysis.
  - `bin/clang-format-fix -g` to format modified tracked C/C++ files (requires clang-format 21+; skips generated trees). Enable the matching pre-commit hook with `git config core.hooksPath .githooks`.
- Host unit tests are CMake + GoogleTest, independent of PlatformIO and not run in CI:
  - All suites: `cmake -S test -B build/test && cmake --build build/test && ctest --test-dir build/test --output-on-failure`.
  - Single suite: build its target then filter, e.g. `cmake --build build/test --target TextPoolTest && ctest --test-dir build/test -R TextPool`. Suites live in `test/<name>/`, each with local stubs for firmware types.
- `./scripts/run_simulator_smoke_test.py` is the end-to-end regression tripwire: builds `env:simulator`, boots headless with a test EPUB, turns pages, and exercises reader menus. See `test/README` for flags and what it does and does not cover.
- CI (`.github/workflows/ci.yml`) runs three required checks on PRs: `bin/clang-format-fix` (fails on diff), `pio check` at the three fail-on-defect levels, and firmware builds for `default` and `sticky` with PlatformIO Core pinned to 6.1.19 (`bin/pio-check-ci` warns on version drift locally).
- For crash debugging, check serial logs, internal heap with `ESP.getFreeHeap()` and `ESP.getMaxAllocHeap()`, task stack high-water marks, and whether cache files need clearing. On S3 targets, also inspect PSRAM free space and largest allocatable block; abundant PSRAM does not prove that internal-RAM or DMA-capable allocations can succeed.
- Hardware verification should mention the concrete device path to test, expected UI/log behavior, and any cache reset needed.

## Generated Files

- Do not edit generated files directly.
- Web portal headers under `src/network/html/*.generated.h` are built by `scripts/build_web.py` from sources in `web/`: pages compose `web/templates/base.html` (shared chrome) with `web/pages/<slug>.{html,css,js}`, plus shared assets `web/assets/style.css` (served at `/style.css`) and `web/assets/logo.png` (served at `/logo.png`). Edit the `web/` sources, never the generated headers.
- I18n generated files under `lib/I18n/` come from `lib/I18n/translations/*.yaml` via `scripts/gen_i18n.py`.

## Cache Format

- EPUB cache lives under `.crosspoint/epub_<hash>/`.
- If you change binary cache layouts, bump the format version first and document it in `docs/file-formats.md`.
- Cache identity is tied to the book path hash; moving or renaming a book creates a different cache.
- Clear the relevant `.crosspoint/epub_<hash>/` cache when testing EPUB parser, layout, image, or binary cache format changes that may otherwise reuse stale output.

## Git Workflow

- Check `git status --short` before edits and before reporting results. Preserve unrelated user changes.
- When resolving merge, rebase, or cherry-pick conflicts, inspect the relevant commit messages for upstream PR references such as `#2608`. Open the PR in its source repository and read its description and changed files before resolving the conflict so the intended behavior is understood.
- Do not resolve conflicts by automatically keeping CrossInk's current implementation or by discarding the upstream change wholesale. Preserve or adapt the upstream intent unless it is already fully implemented, would introduce a regression, or would substantially and unjustifiably change CrossInk's UX or behavior. When rejecting an upstream change, state the concrete reason.
- If a referenced PR cannot be accessed, inspect the source commit diff and nearby history, then report that the PR intent could not be verified instead of guessing.
- Do not commit unless the user explicitly asks or committing is part of the skill utilized.
- Before staging, ensure ignored/generated/local files such as `.pio/`, `*.generated.h`, `compile_commands.json`, and `platformio.local.ini` are not included.
- Branch names should use repo-style prefixes such as `feat/`, `fix/`, `docs/`, `refactor/`, `test/`, or `chore/`.
- Suggested commit messages should follow `<type>: <short summary>`, using types like `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, or `perf`.

## Changelog

When new features are added or issues are fixed, make sure to add an entry to `CHANGELOG.md` with the user-facing description of the change. Types of changes should have their own section.

### Changelog Guiding Principles

- Changelogs are _for humans_, not machines.
- There should be an entry for every single version.
- The same types of changes should be grouped.
- Versions and sections should be linkable.
- The latest version comes first.
- The release date of each version is displayed.

### Types of Changelog Changes

- Added - for new features.
- Changed - for changes in existing functionality.
- Deprecated - for soon-to-be removed features.
- Removed - for now removed features.
- Fixed - for any bug fixes.
- Security - in case of vulnerabilities.
