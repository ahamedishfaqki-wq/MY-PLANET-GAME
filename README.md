# My Planet — Full Game Design & Technical Spec

A life-sim → city-builder → space-colony idle game with social multiplayer.
This doc is written to be handed directly to an agentic coding assistant (e.g. GPT-5.1-Codex-Max) as a build spec, phase by phase.

---

## 1. Core Concept

- **Genre:** Idle/incremental economy + city builder + light social multiplayer
- **Loop:** Earn coins → spend coins on buildings/upgrades → unlock next tier → eventually unlock Space/Mars content
- **No XP/levels.** Progression gates are purely coin-threshold based ("no levels — only coins").
- **Platforms (target order):** Web (fastest to test) → Desktop export → Mobile export

---

## 2. Core Game Loop (MVP)

1. Player creates a character (gender, hair, outfit, accessories)
2. Player spawns in a starter home with 500 coins
3. Player performs an action to earn coins (tap a job / wait an idle timer / complete a task)
4. Player opens Build Menu, spends coins on a building (Small Stall → Restaurant → Shop → Hotel → Bus Stand → Railway → Metro → City Expansion)
5. Buildings generate passive coin income over time
6. At a coin threshold, Space/Mars tier unlocks (SpaceX Facility → Mars Colony)
7. (Phase 2) Multiplayer: friends list, chat, visiting other players' cities

---

## 3. Systems Breakdown

### 3.1 Character System
- 3 base presets: Boy, Girl, "Cool Style" (all should really just be starting presets, not hard gender locks — let players mix any hair/outfit/accessory with any base body)
- Customization categories: Hair (4+ options), Outfit (4+ options), Accessories (glasses/cap/backpack)
- Store as a flat data object, not baked sprites, so any combination can render:
```json
{
  "characterId": "player_001",
  "body": "base_a",
  "hair": "hair_02",
  "outfit": "outfit_03",
  "accessory": "glasses_01"
}
```
- Rendering approach: layered sprites (body → outfit → hair → accessory), OR swappable mesh/texture slots if 3D. Layered 2D sprites is far faster for a solo dev.

### 3.2 Economy System (the heart of the game)
- Single source of truth: `PlayerState.coins` (integer or long, not float — avoid floating point drift)
- Income sources:
  - **Active:** tap-to-earn job actions (small, frequent, capped per hour to avoid pure tapping abuse)
  - **Passive:** each owned building has a `coinsPerTick` and `tickIntervalSeconds`
- Central `EconomyManager` handles all coin mutations — nothing else should touch `coins` directly. This makes saving, syncing, and eventually anti-cheat validation much easier.

### 3.3 Building / City System
- Buildings are entirely data-driven (see schema below) — never hardcode a building's cost/behavior in code. This lets you add new buildings without touching game logic.
- Each building has: id, name, cost, unlockRequirement (coins earned so far, or previous building owned), incomeRate, tier, visual asset reference
- Placement: start simple — a fixed set of building "slots" in the home city scene rather than free-form placement. Free placement (drag anywhere, collision checking, grid snapping) is a whole extra system — save it for Phase 2 polish.

```json
{
  "buildingId": "restaurant",
  "displayName": "Restaurant",
  "cost": 5000,
  "unlockRequiresBuildingId": "small_stall",
  "incomePerTick": 25,
  "tickSeconds": 60,
  "tier": "city",
  "nextBuildingId": "shop_store"
}
```

### 3.4 Progression Gate (Space & Mars)
- Same data-driven pattern as buildings, just a separate tier list with much higher costs:
  - SpaceX Facility: 1,000,000,000 coins
  - Mars Colony: 5,000,000,000+ coins
- Treat this literally as "buildings with big numbers" in your system — don't build a separate engine for it. Reuse the Building System.

### 3.5 Multiplayer / Social (Phase 2 — do not build first)
- Needs a backend: Firebase, Supabase, or custom Node + WebSocket server + database
- Minimum feature set:
  - Friends list (add/accept)
  - Text chat (simple message list per friend or per group)
  - Visit another player's city (read-only view of their PlayerState + city layout)
- **Design your PlayerState as a clean serializable object from day one** (Phase 1), even in single-player, so Phase 2 is "sync this object to a server" rather than a rewrite.

---

## 4. Data Model (the whole game state, in one object)

```json
{
  "playerId": "uuid",
  "character": { "body": "base_a", "hair": "hair_02", "outfit": "outfit_03", "accessory": "glasses_01" },
  "coins": 12450,
  "ownedBuildings": ["small_stall", "restaurant"],
  "buildingLevels": { "small_stall": 1, "restaurant": 1 },
  "lastSavedAt": "2026-09-12T10:00:00Z"
}
```

Keep this ONE object as the save file (local JSON for MVP). Everything — UI, economy calculations, save/load — reads and writes through this shape.

---

## 5. Tech Stack Recommendation

| Layer | Recommendation | Why |
|---|---|---|
| Engine | **Godot 4 (GDScript)** | Free, fast 2D iteration, easy web export, gentle learning curve |
| Alternative | Phaser 3 (TypeScript) | If you want browser-first + easier future web backend integration |
| Save system (MVP) | Local JSON file / Godot's `ConfigFile` | No backend needed yet |
| Backend (Phase 2) | Supabase (Postgres + Auth + Realtime) | Fastest way to get accounts, sync, and realtime chat without building your own server |
| Art (placeholder) | Kenney.nl free asset packs, or simple geometric placeholders | Don't commission art until the loop is proven fun |

---

## 6. Folder Structure (Godot example)

```
my-planet/
  project.godot
  scenes/
    Main.tscn
    CharacterCreator.tscn
    HomeCity.tscn
    BuildMenu.tscn
  scripts/
    autoload/
      PlayerState.gd        # singleton holding the data model
      EconomyManager.gd     # all coin math lives here
      SaveManager.gd        # load/save JSON
    ui/
      BuildMenuController.gd
      CharacterCreatorController.gd
    buildings/
      Building.gd           # generic building node, reads from data
  data/
    buildings.json           # all building definitions (data-driven)
    character_options.json   # hair/outfit/accessory catalog
  assets/
    sprites/
    ui/
```

---

## 7. Build Order / Roadmap

**Phase 1 — MVP (single player, local save)**
1. Project scaffold + PlayerState singleton + JSON save/load
2. Character creator scene (layered sprite swap, writes to PlayerState.character)
3. Home city scene with fixed building slots
4. EconomyManager: tap-to-earn action + passive income ticking
5. Build menu UI reading from `buildings.json`, spending coins, unlocking next tier
6. Basic UI: coin counter, building slot states (locked/unlocked/owned)

**Phase 2 — Depth & polish**
7. Free-form building placement (grid snapping)
8. Space/Mars tier using the same Building system
9. Juice: animations, sound, particle effects on coin gain
10. Mobile/export packaging

**Phase 3 — Multiplayer**
11. Supabase auth + player accounts
12. Sync PlayerState to cloud
13. Friends list + chat
14. Visit-a-friend's-city read view

---

## 8. First Prompt to Give an Agentic Coding Assistant

Copy this as your literal opening prompt to GPT-5.1-Codex-Max (or similar) to scaffold Phase 1, Step 1–2:

> Create a new Godot 4 project called "my-planet" with the folder structure below [paste Section 6]. Implement a `PlayerState` autoload singleton matching this data model [paste Section 4 JSON]. Implement a `SaveManager` autoload that saves/loads PlayerState to/from a local JSON file. Then build a simple `CharacterCreator.tscn` scene with three dropdown/button groups (Hair, Outfit, Accessory) that update PlayerState.character live, using placeholder colored rectangles as sprite layers for now. Do not implement economy or building systems yet — stop after character creation and save/load are verified with a debug print of PlayerState.

Then iterate scene by scene, always giving it one bounded task at a time and reviewing before moving to the next (see Section 7 order).

---

## 9. Key Pitfalls to Avoid

- **Don't build free-form placement or multiplayer first.** Both are expensive and unnecessary to prove the core loop is fun.
- **Don't hardcode building costs/effects in code.** Always read from `buildings.json` — you'll change these numbers constantly while balancing.
- **Don't use floats for currency.** Use integers (or a BigInt-style library once numbers exceed ~2^53 for the Mars tier's billions).
- **Playtest the coin pacing early** — idle games live or die on whether the wait times feel rewarding, not tedious. Get a friend to click through Phase 1 before investing in art.# My Planet — Playable MVP Skeleton

This is a working Godot 4 skeleton implementing Phase 1 of the game spec:
character creation, coin economy, and a data-driven build menu, with local save/load.

## How to run

1. Install [Godot 4.2+](https://godotengine.org/download) (standard, not .NET version — this project uses GDScript).
2. Open Godot → "Import" → select the `project.godot` file in this folder.
3. Press the Play button (top right) or F5. It will run `scenes/Main.tscn`.

## What works right now

- **Character tab:** pick Hair/Outfit/Accessory from dropdowns, hit a preset (Boy/Girl/Cool Style),
  and set a **Username**. Selections are stored in `PlayerState` and shown as text (no art yet —
  see "Next steps").
- **Build tab:** shows every building from `data/buildings.json`. Buttons are enabled/disabled
  automatically based on whether you can afford it and have unlocked its prerequisite.
- **Map tab:** a clickable Kerala district map (14 districts, laid out roughly north-to-south —
  stylized for gameplay, not a geographically precise GIS map). Clicking a district sets
  `PlayerState.place`.
- **Chat tab:** a working chat UI (message list + input + send). Right now it's backed by a
  **mock** in `BackendManager.gd` — messages you send are echoed locally and pre-seeded with
  5 sample messages, so you can build/test the UI before wiring a real server.
- **Leaderboard tab:** ranks players by coins, highest first. Currently combines 5 **mock**
  players (in `BackendManager._mock_players`) with your real live coin count, so the list
  updates as you earn — this proves out the sort/format logic before a real backend exists.
- **Earn Coins button:** top-right, gives +10 coins per click (placeholder for a real job minigame).
- **Mute button:** top-right, toggles both music and sound effects.
- **Sound:** button clicks, coin gains, and successful purchases play real procedurally-generated
  beep tones (`SoundManager.gd`) — these need zero asset files, they synthesize sound at runtime.
- **Background music:** the code is wired up (`SoundManager._try_load_music()`), but no music
  file is bundled — we can't ship copyrighted music. Drop your own `bgm.ogg` into
  `assets/audio/` (see the .txt file there for details) and it'll loop automatically.
- **Real chat with other players:** the Chat tab now has Host/Join controls using Godot's
  built-in networking (`NetworkManager.gd`, ENet) — no external service or account needed.
  One player taps **Host**, the other enters the host's IP and taps **Join**. On the same
  WiFi/LAN this works immediately; over the open internet the host would need port forwarding
  (a router setting, not code). Until you Host/Join, chat still works locally via the mock —
  it upgrades to real player-to-player messages automatically once connected.
- **Owned buildings passively generate coins** on their own timer (see `EconomyManager._process`).
- **Autosave** every 30 seconds and on quit, to `user://save_game.json`
  (on Windows this resolves to `%APPDATA%/Godot/app_userdata/My Planet/`).

### Important: Leaderboard is still a single-player mock; Chat can now be real

The leaderboard still combines 5 mock players with your live coin count — it has no network
component yet. Chat, however, is genuinely real once you Host/Join (see above). Making the
leaderboard show real other players needs the same kind of step: either share it over
`NetworkManager` (broadcast your coin count like chat messages) for LAN play, or move to a real
cloud backend (Supabase) for players who aren't on the same network — see "Next steps" below
for exactly how.

## Project structure

```
my-planet/
  project.godot
  scenes/
    Main.tscn              # root scene: tabs + earn button
    CharacterCreator.tscn
    BuildMenu.tscn
  scripts/
    autoload/
      PlayerState.gd       # the single data model (singleton)
      SaveManager.gd       # JSON save/load
      EconomyManager.gd    # all coin math + building rules (singleton)
    ui/
      MainController.gd
      CharacterCreatorController.gd
      BuildMenuController.gd
    ui/
      KeralaMapController.gd
      ChatController.gd
      LeaderboardController.gd
  data/
    buildings.json          # every building's cost/income/unlock — edit freely
    character_options.json  # hair/outfit/accessory catalog + presets
    kerala_districts.json   # 14 districts + stylized x/y layout positions
  assets/
    audio/
      PUT_YOUR_MUSIC_HERE.txt   # drop your own bgm.ogg here
```

New autoloads:
- **BackendManager** — owns the mock chat/leaderboard data. Swap this for a real cloud backend
  (Supabase, etc.) later without touching the UI scripts — see step 5 below.
- **NetworkManager** — real ENet peer-to-peer networking for Host/Join chat. No account or
  external service required, just an IP address.
- **SoundManager** — click/coin sound effects (generated in code) and background music playback
  (loads your own file).

## How to test real multiplayer chat

1. Export the project or run it from two separate Godot editor windows (or two computers on the
   same WiFi).
2. On Player A's copy: go to the Chat tab, tap **Host**.
3. On Player B's copy: enter Player A's local IP address (find it with `ipconfig` on Windows or
   `ifconfig`/`ip addr` on Mac/Linux) in the IP field, tap **Join**.
4. Messages typed on either copy now appear on both — this is real networking, not a mock.

## Next steps (in priority order)

1. **Replace placeholder text/UI with real sprites.** Swap `PreviewLabel` in
   `CharacterCreator.tscn` for a `Sprite2D`/`AnimatedSprite2D` stack (body → outfit → hair →
   accessory layers), driven by the same `PlayerState.character` values.
2. **Replace the build menu list with visual building slots** in an actual city scene, instead
   of a scrolling button list — same underlying `EconomyManager` calls, just different UI.
3. **Add the Space/Mars tier visually** — no code changes needed, they're already in
   `buildings.json` (`spacex_facility`, `mars_colony`); just needs distinct art/UI treatment
   since they're a different "look" (space vs. city).
4. **Tune the numbers.** Everything is in `buildings.json` — no code touching required to
   rebalance costs, income rates, or tick speeds.
5. **Make Chat & Leaderboard real (the actual next step for what you asked for):**
   - Set up a Supabase project (free tier is enough): a `players` table (username, place, coins,
     updated_at) and a `messages` table (username, text, created_at).
   - In `BackendManager.gd`, replace `fetch_leaderboard()` with a Supabase `SELECT * FROM players
     ORDER BY coins DESC`, and replace `send_chat_message()`/`fetch_chat_history()` with inserts/
     selects on `messages`. Subscribe to Supabase Realtime on `messages` and call
     `message_received.emit(...)` when a row comes in from another player.
   - Call `SaveManager.save_game()` → also push `PlayerState.to_dict()` to the `players` table
     so your coins show up for others.
   - Because `ChatController` and `LeaderboardController` only talk to `BackendManager`'s public
     interface, none of their code needs to change.
6. **Add player accounts** (Supabase Auth) so usernames are unique and persistent instead of
   just a local text field.

## Design principle to preserve as you extend this

Everything gameplay-related is either:
- **Data** (`buildings.json`, `character_options.json`) — change freely, no code needed, or
- **Logic that reads that data** (`EconomyManager`, `PlayerState`) — the only place coins/buildings
  should ever be mutated.

Keep new features following that split and the game stays easy to balance and debug.
