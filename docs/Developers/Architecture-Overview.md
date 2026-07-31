# Architecture Overview

This page is a map of the codebase for newcomers: how the Gradle modules fit together, how the game state is modeled, how the UI is built, and how a turn actually gets processed. It complements (and doesn't replace) [Project structure and major classes](Project-structure-and-major-classes.md), which goes deeper on the game-state class hierarchy — a few class names below have since been renamed (`CivilizationInfo` → `Civilization`, `CityInfo` → `City`, `TileInfo` → `Tile`); this page uses the current names.

## Modules

Unciv is a [LibGDX](https://libgdx.com/) project, split into Gradle modules declared in [`settings.gradle.kts`](../../settings.gradle.kts):

| Module | Purpose |
| --- | --- |
| `core` | Platform-independent code: essentially the entire game — logic, models, rendering, and UI. This is where ~99% of development happens. |
| `desktop` | The desktop (Windows/Linux/macOS) launcher, packaging, Discord Rich Presence integration, and the desktop image packer. |
| `android` | The Android launcher/activity, plus the source-of-truth game assets (`android/assets/**` — images, sounds, and the `jsons/` ruleset data). These assets are bundled into every platform's build, so "android" here is misleading; it's really the assets module. |
| `server` | `UncivServer` — a small standalone Ktor server that relays multiplayer game files between clients, packaged as its own jar. |
| `tests` | Unit tests, plus developer tooling like `FasterUIDevelopment` (see [UI development](UI-development.md)). Run with `./gradlew tests:test`. |
| `buildSrc` | Build-time configuration (e.g. `BuildConfig`, shared Gradle logic) used by the other modules' build scripts. |

Because `core` has no platform dependencies, the same game logic runs unchanged on desktop, Android, and (via the assets bundled into the jar) anywhere a JVM exists. Platform modules only implement thin interfaces (file I/O, fonts, clipboard, audio quirks, display info) that `core` depends on abstractly — see `PlatformSpecific` and the various `*SaverLoader`, `*Font`, `*Display` classes under `desktop/src` and `android/src`.

Networking/serialization uses Kotlin Multiplatform serialization (`kotlinx.serialization` + Ktor) alongside the classic GDX `Json` (see `core/src/com/unciv/json/UncivJson.kt`) used for save files and ruleset JSON.

## Package layout (`core/src/com/unciv`)

```
com.unciv
├── logic/        Game state, turn processing, AI automation, multiplayer, save/load
├── models/        Ruleset definitions, stats, translations, metadata (settings), skins, tilesets
├── ui/            Scene2D screens and reusable widgets
├── json/          JSON (de)serialization helpers
├── utils/         Small platform-agnostic utilities
├── view/          Rendering helpers shared across screens
└── UncivGame.kt   The application entry point (see below)
```

### `logic/` — the simulation

| Package | Responsibility |
| --- | --- |
| `logic/` (root) — `GameInfo.kt`, `GameStarter.kt` | The game-state root and the "new game" bootstrap process. |
| `civilization/` — `Civilization.kt` | A player (nation + state), and its managers (tech, policies, diplomacy, gold, great people, religion, etc). |
| `city/` — `City.kt` | A city and its managers (population, construction, expansion, stats). |
| `map/` | `TileMap`, `map/tile/Tile.kt`, `map/mapunit/MapUnit.kt`, map generation. |
| `battle/` | Combat resolution between units/cities. |
| `trade/` | Diplomatic trade offers and evaluation. |
| `automation/` | The AI: `automation/city`, `automation/civilization`, `automation/unit` decide what the AI does each turn. |
| `multiplayer/` | Client-side multiplayer: game file syncing, the newer `apiv2` server protocol, friends, chat. |
| `files/` | Save/load and settings persistence (`UncivFiles`). |
| `github/` | Mod browsing/download from GitHub. |
| `simulation/` | Headless game simulation used to balance/tune the AI (see [Simulations](../Other/Simulations.md)). |
| `event/` | A lightweight in-process event bus. |

### `models/` — static/moddable data and shared value types

- `ruleset/` — `Ruleset` and everything it's made of: `Building`, `Policy`, `Belief`, `Victory`, plus `nation/`, `tech/`, `tile/`, `unit/`, `unique/` subpackages. This is the moddable "rules of the game" layer (see next section) and `validation/` for ruleset sanity-checking used by the modding tools.
- `stats/` — the `Stats` value type (gold/science/culture/food/etc.) used throughout yield calculations.
- `translations/` — the string translation engine (see [Translation generation](../Translating/Translation-generation.md)).
- `metadata/` — `GameSettings` and other persisted app-level (not game-state) settings.
- `skins/`, `tilesets/` — UI skin and tileset definitions/caches for moddable visuals.

### `ui/` — Scene2D presentation layer

Built on GDX's `scene2d`, mostly via the `Table` widget (see [UI development](UI-development.md)). Key subpackages:

- `screens/` — one subpackage per screen: `worldscreen/`, `cityscreen/`, `newgamescreen/`, `mapeditorscreen/`, `civilopediascreen/`, `diplomacyscreen/`, `overviewscreen/`, `pickerscreens/`, `multiplayerscreens/`, `savescreens/`, `modmanager/`, `victoryscreen/`, `devconsole/`, and `basescreen/` (the common `BaseScreen` all screens extend).
- `components/` — reusable widgets shared across screens (buttons, tables, input helpers, fonts).
- `images/` — `ImageGetter` and texture-atlas handling.
- `popups/` — modal dialogs (`ConfirmPopup`, etc).
- `audio/` — music/sound playback control.
- `crashhandling/` — wraps game logic so unexpected exceptions surface as a `CrashScreen` instead of silently corrupting state (see [Guiding principles: crash early, crash often](../Guiding-Principles.md)).

## The application entry point — `UncivGame`

`UncivGame` (`core/src/com/unciv/UncivGame.kt`) implements GDX's `Game` interface and is the root object every platform launcher (`DesktopLauncher`, `AndroidLauncher`) constructs. It:

- Holds the currently-loaded `GameInfo` (the game state, see below) and drives screen transitions (`Game.setScreen`).
- Owns long-lived singletons: `Settings` (`GameSettings`), `UncivFiles`, `MusicController`, `Multiplayer`, translation and ruleset/skin/tileset caches.
- Delegates platform-specific behavior (file access, fonts, display) through the `PlatformSpecific` interface implemented per-module.

## Game state vs. Ruleset — the core moddability split

Everything in Unciv falls into one of two categories, and this split is the single most important architectural decision in the codebase:

1. **Game state** — what's different about *this particular game in progress*: `GameInfo` → `Civilization` → `City`, and `TileMap` → `Tile` → `MapUnit`. This is what gets serialized into a save file.
2. **Ruleset** — the rules everything plays by: what technologies, buildings, units, policies, nations, terrains, and improvements *exist* and what they do. Defined in JSON under `android/assets/jsons/**` and loaded into a `Ruleset` object (`RulesetCache`), which is explicitly **not** part of save-game serialization — it's re-loaded from the mod/ruleset files every time.

Game-state objects reference ruleset objects **by name** (a string), and resolve that name against the active `Ruleset` at runtime. Since these are essentially the same objects for every mod, changing the ruleset (i.e. playing with a different mod combination) changes the entire game without touching the game-state code. This is what makes Unciv's modding model work — see [Mods](../Modders/Mods.md) and [Modding freedom in Open Source](Translations,-mods,-and-modding-freedom-in-Open-Source.md).

Because save files must stay small and game-state objects need live references to each other and to ruleset objects for performance, most links are stored as names on disk and rehydrated into real object references on load — see [Saved games and transients](Saved-games-and-transients.md) for how `setTransients()` and `@Transient` fields make this work, including the layered caching used for `Unique`s.

The "uniques" system (`models/ruleset/unique/`) is the generic mechanism that lets a small number of effect types be combined with parameters and conditionals to express almost all game behavior — see [Uniques](Uniques.md), [Unique parameters](../Modders/Unique-parameters.md), and the [modding philosophy](../Guiding-Principles.md#modding-philosophy-minimal-objects-maximum-interactions) of "minimal objects, maximum interactions."

## Turn processing

There isn't a single "NextTurn" class, but the process (triggered from `WorldScreen`) is architecturally significant: Unciv **clones the entire `GameInfo`** for each turn rather than mutating the live one in place. This is deliberate, for two reasons:

- **Thread safety** — turn processing happens off the render thread so the UI stays responsive; mutating the `GameInfo` the renderer is currently reading from would race.
- **Multiplayer reproducibility** — multiplayer works by shipping the entire game state around, so state mutation had to become "produce a new state" as part of that transition, which incidentally solved the threading problem too.

AI decision-making for the turn lives in `logic/automation/` (split by city/civilization/unit), and is what `simulation/` exercises headlessly for AI balance testing.

## Rendering the map

The map is the most performance-sensitive part of the UI. Each tile is drawn in **layers** (terrain, then resources, then improvements, then units, etc.) across the *entire visible map* before moving to the next layer — not tile-by-tile — because rebinding GL textures is the expensive part of rendering, and grouping by layer/category minimizes texture swaps. `TileGroup` composes a single tile's layers (used standalone for Civilopedia/map editor previews); `TileGroupMap` "steals" those images into shared `TileMapLayer` containers, one per layer type, for efficient whole-map rendering. See [Map rendering](Map-rendering.md) for the full explanation, including how the image atlases themselves are packed by category to respect GL texture size limits.

## Multiplayer

Multiplayer is file-based at its core: a full `GameInfo` is serialized and exchanged rather than diffed/streamed. `logic/multiplayer/` handles this client-side (polling, friends, chat), talking to either a user-hosted `UncivServer` (the `server` module) or a hosted API (the newer `apiv2` protocol). Turn cloning (above) is part of what makes this tractable — a `GameInfo` snapshot is a self-contained, shippable unit.

## Build, test, and release pipeline

- Local development: [Building Locally](Building-Locally.md).
- CI builds/tests every push (GitHub Actions, replacing the historical Travis setup referenced in some older docs).
- Release packaging, store publishing (Google Play, F-Droid, itch.io), and the wiki-sync process are documented in [From code to deployment](From-code-to-deployment.md).

## Where to go next

- [Project structure and major classes](Project-structure-and-major-classes.md) — a narrative walkthrough of the game-state class tree.
- [Guiding Principles](../Guiding-Principles.md) — the design philosophy behind AI behavior, modding, and error handling.
- [Coding standards](Coding-standards.md) — style conventions for contributions.
- [UI development](UI-development.md), [Map rendering](Map-rendering.md), [Saved games and transients](Saved-games-and-transients.md) — deep dives referenced above.
- [Mods](../Modders/Mods.md) and [Uniques](Uniques.md) — the moddability system from a mod-author's perspective.
