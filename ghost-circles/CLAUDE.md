# Ghost Circles — Project Brief

## What is Ghost Circles?

Ghost Circles is a location-based Progressive Web App (PWA) for iPhone. Players walk around in the real world and encounter invisible "ghost circles" — geofenced areas anchored to GPS coordinates. When a player enters a circle, the app triggers a sensory response: a sound (called a whisper), a haptic vibration, and a visual colour change on the map. The concept is an immersive, ambient experience somewhere between a walking audio tour and a ghost hunt.

## Current State (v3.51) — Save/Resume Release

The engine and editor are feature-complete for single-haunt, GPS-driven experiences with a full items system, time/date-aware conditions, and player progress persistence. Play state is automatically saved to localStorage after every circle visit and inventory change, keyed by haunt id. Players resume exactly where they left off on reload. A ↺ button in the haunt title bar lets players start over with a confirmation prompt. v3.51 is the stable authoring platform on which experiences are built.

---

## Engine Capabilities

### Circle state machine
Each circle is always in one of three states:

| State | Colour | Meaning |
|---|---|---|
| Locked | Grey `#9e9e9e` | Conditions not met — invisible to proximity detection |
| Passive | Red `#e53935` | Unlocked, player is outside |
| Active | Green `#2e7d32` | Player is inside and circle holds highest priority |

### Conditions (locked → passive)
`conditions` array on a circle — ALL must be true to unlock. Empty array means passive from start.

### Lock conditions (passive/active → locked)
`lockConditions` array on a circle — ANY one being true immediately re-locks the circle (fading audio if playing). When the lock condition clears and unlock conditions are still met, the circle automatically restores to passive. Evaluated in Pass 4 on every GPS tick.

### Condition types
All types work in `conditions`, `lockConditions`, `visibleConditions`, and `hideConditions`:

| Type | Syntax | True when |
|---|---|---|
| `visited` | `{ type: "visited", circle: <name>, minTimes: <n> }` | Named circle has been visited ≥ n times |
| `hasItem` | `{ type: "hasItem", item: <name> }` | Item quantity > 0 |
| `isActive` | `{ type: "isActive", circle: <name> }` | Named circle is currently active |
| `isPassive` | `{ type: "isPassive", circle: <name> }` | Named circle is currently passive |
| `isLocked` | `{ type: "isLocked", circle: <name> }` | Named circle is currently locked |
| `isVisible` | `{ type: "isVisible", circle: <name> }` | Named circle is currently on the map |
| `isHidden` | `{ type: "isHidden", circle: <name> }` | Named circle is currently off the map |
| `itemQuantity` | `{ type: "itemQuantity", item: <name>, min: <n> }` | Item quantity ≥ n |
| `itemQuantity` | `{ type: "itemQuantity", item: <name>, max: <n> }` | Item quantity ≤ n |
| `timeOfDay` | `{ type: "timeOfDay", from: "20:00", to: "06:00" }` | Current local time is within the range (cross-midnight supported) |
| `dayOfWeek` | `{ type: "dayOfWeek", days: ["saturday", "sunday"] }` | Current local day is in the array (lowercase English names) |
| `date` | `{ type: "date", from: "2026-10-31", to: "2026-11-01" }` | Current local date falls within the range (YYYY-MM-DD; same date for single day) |

`min` and `max` can be combined in one `itemQuantity` condition. All three time/date types use the device's local time via JavaScript's `Date` object.

### Priority system
When the player is inside multiple unlocked circles simultaneously, only the circle(s) with the highest `priority` value become active. Lower-priority circles remain passive — no whisper, no vibration. On a tie, all tied circles activate simultaneously. Priority demotion (a higher-priority circle entering range) does not trigger an exit vibration and does not fire exitActions.

### Visibility
Circle map visibility is dynamic and condition-driven:

- `visibleConditions` array — circle starts off the map, appears when ALL conditions are met
- `hideConditions` array — circle starts on the map, disappears when ALL conditions are met
- Both can be combined: visibleConditions controls show; hideConditions can re-hide after showing
- If neither is defined, the static `visible` property applies (`true` by default; `false` permanently hides)

Evaluated in **Pass 5** on every GPS tick. All engine logic (state machine, audio, proximity) runs regardless of whether a circle is currently on the map.

### Whisper repeat
`repeat` property per circle: `1` = play once on entry (default), `0` = loop continuously until exit, `N` = play N times in sequence. All audio routes through a GainNode. On exit, audio fades out over 1 second.

### Play to end
`playToEnd` boolean (default `false`). When true, if the player exits while a whisper is playing, the audio is not faded — it continues to its natural end. Incompatible with loop (`repeat: 0`); the editor greys it out in that case.

A global lock (`playToEndLock`) is held for the duration. While the lock is held, no other whisper (including re-entry of the same circle) can start. The lock is released when the audio ends naturally.

### Lock on end
`lockOnEnd` boolean (default `false`). When true, the circle locks itself automatically when its whisper finishes — whether the player stayed inside and the audio completed naturally, or whether `playToEnd` let the audio run after exit. Sets `rt.lockedByEnd = true` on the runtime object.

Requires `playToEnd: true` to be practically useful. The editor greys out `lockOnEnd` unless `playToEnd` is also on.

### lockedByEnd flag
`rt.lockedByEnd` is a runtime-only flag (not saved in the haunt JSON). When set, `checkUnlocks` and Pass 4 skip the circle unconditionally — it cannot be re-promoted regardless of its `conditions` array. Also set by exit-action `lock` so that externally locked circles are equally protected from auto-promotion.

### Volume
`volume` per circle, range 0–2.0, default 1.0 (100%). Applied to the GainNode at playback start.

### Inventory actions (when visited)
`inventoryActions` on circles: optional array of actions that fire on passive → active (player entry):
- `{ type: "addItem", item: <name> }` — increments item quantity by 1
- `{ type: "removeItem", item: <name> }` — decrements item quantity by 1 (minimum 0)

Circle-to-circle unlock/lock/show/hide actions are also authored in the "When visited" section of the editor but are stored as `visited` conditions on the target circle, not in `inventoryActions`.

### Exit actions (when exited)
`exitActions` on circles: optional array of actions that fire when the player **physically exits** the circle (not on priority demotion):

| Type | Effect |
|---|---|
| `lock` | Locks the target circle (sets state → locked, lockedByEnd → true, stops audio with fade) |
| `unlock` | Unlocks the target circle (sets state → passive, clears lockedByEnd) |
| `show` | Adds the target circle to the map (sets onMap → true) |
| `hide` | Removes the target circle from the map (sets onMap → false) |
| `addItem` | Increments the named item's quantity by 1 |
| `removeItem` | Decrements the named item's quantity by 1 (minimum 0) |

**Timing with `playToEnd`:** If `playToEnd` is true and audio is still playing when the player exits, exit actions are deferred — `rt.pendingExitActions` is set to `true` and the actions fire in the `onended` callback alongside `lockOnEnd`, once the audio finishes naturally.

**Timing with `lockOnEnd`:** When the player stays inside through the full audio and `lockOnEnd` fires, exit actions also fire at that moment (natural completion = the effective exit).

`fireExitActions(rt)` is the shared function that handles all six action types and resets `rt.pendingExitActions`.

### Immediate activation on unlock
If a player is already standing inside a circle when it unlocks, it activates immediately — no need to exit and re-enter. `checkProximity` re-runs recursively on unlock.

### playToEnd lock release
When the `playToEndLock` releases (audio ends naturally in either the stayed-inside or walked-out path), `onPlayToEndLockReleased()` runs:
1. Scans all `circleRuntime` entries for circles that are `active + inRange + activeGain === null` — circles that were promoted while the lock was held but whose whispers were blocked. Calls `startWhisperPlayback` on each immediately.
2. Calls `checkProximity` with `lastKnownLatLng` to handle any passive circles the player is currently inside.

### Start inside a circle
After `initAudio()` resolves (Tap to Start), the `.then()` handler scans all `circleRuntime` entries for `active + inRange + activeGain === null` and calls `startWhisperPlayback` on each — handles circles that became active during GPS/audio initialisation before buffers were loaded.

Note: `decodeAudioData` uses the explicit callback form wrapped in `new Promise((res, rej) => audioCtx.decodeAudioData(arrayBuffer, res, rej))` to avoid a Safari timing issue where the promise-based API can resolve before the buffer is actually ready.

### Time condition re-evaluation
Time-based conditions (`timeOfDay`, `dayOfWeek`, `date`) are evaluated in two ways:

**On first GPS fix** (`firstGpsFix` flag): `onPosition` calls `checkUnlocks(L.latLng(...))` before the normal `checkProximity` on the very first position fix after page load. This runs typically within 8 seconds of Tap to Start — long before the first 60-second tick.

**Every 60 seconds** (`setInterval`): keeps time-based conditions live as time passes without requiring player movement. On each tick:
1. Calls `checkUnlocks(L.latLng(...))` directly — re-evaluates every locked circle (except `lockedByEnd`) against current conditions. This is necessary because `checkProximity`'s Pass 3 skips locked circles entirely; `checkUnlocks` is only called inside Pass 3 on passive→active transitions, so it would never run when the player is stationary.
2. Calls `checkProximity` — handles Pass 4 re-locking, activates any newly-passive circles the player is standing inside, and updates visibility (Pass 5).

Logs `time tick: checking N locked circles` each tick (visible in debug mode).

### Player progress persistence (play mode only)
Play state is automatically saved to `localStorage` after every `checkProximity` call and after every `fireExitActions` call. The save is keyed `ghost-circles-save-<hauntId>` so different haunts never collide. Saved fields:

- `visited` — per-circle visit count
- `states` — per-circle state (`locked` or `passive`; `active` is saved but restored as `passive` — the engine re-activates on the first GPS tick if the player is still inside, which avoids spurious exit-action firing if they've since moved away)
- `lockedByEnd` — per-circle flag so circles locked by `lockOnEnd` or an exit-action `lock` stay locked across sessions
- `inventory` — current quantity per item

On load, the save is applied after `circleRuntime` is built but before GPS starts (`_hauntSaveKey` is set inside the haunt URL loading block). `savePlayState()` guards with `if (!_hauntSaveKey) return` so it is a complete no-op in editor mode.

**Start Over button (↺)** — shown in the haunt title bar (right side, white, `pointer-events: auto` while the bar itself keeps `pointer-events: none`). Tap → `window.confirm` → on confirm: `localStorage.removeItem(saveKey)`, then `_hauntSaveKey = null` (prevents any in-flight GPS tick from re-saving before the reload completes), then `window.location.reload()`.

### iPhone reliability
- **Screen Wake Lock**: requested after Tap to Start, reacquired automatically on return from background
- **GPS recovery**: on any non-permission GPS error, the watcher restarts after 3 seconds. On `visibilitychange` (screen back on), the GPS watcher is always stopped and restarted immediately
- **GPS staleness indicator**: the player dot turns grey if no fix received for more than 10 seconds
- **AudioContext recovery**: on `visibilitychange`, `audioCtx.resume()` is called explicitly to lift iOS's automatic suspension
- **Tap to Start overlay**: required to unlock iOS audio and GPS simultaneously. Creates AudioContext inside the tap gesture

---

## Design Patterns

### Onion / bullseye
Concentric circles at the same location with decreasing radii and increasing priority. The outermost ring plays an ambient loop while the player approaches; stepping into the inner ring (higher priority) demotes the outer to passive (audio fades), activates the inner with its own whisper. Exit actions on the inner can re-lock it and unlock a third ring, creating a staged reveal.

```
outer  — radius 80 m, priority 10, repeat 0 (loop)
middle — radius 40 m, priority 20, repeat 1, playToEnd, lockOnEnd
         exitActions: [{ type: 'lock', target: 'outer' }]
inner  — radius 15 m, priority 30, startsLocked
         conditions: [{ type: 'visited', circle: 'middle', minTimes: 1 }]
```

### 2copiesKey (items as OR workaround)
The condition system is AND-only — a circle unlocks only when ALL conditions are true. To unlock a circle when the player has visited either circle A or circle B, use an item as a flag: both A and B have `addItem: key` in their When visited actions; the target circle has `hasItem: key` as its unlock condition. First visit to either A or B drops the key and unlocks the target.

### Mines! (item quantity as lives)
Give the player a `hearts` item starting at quantity 3. "Mine" circles have `removeItem: hearts` as a When visited action and a `lockOnEnd` whisper ("Ouch!"). A "dead" circle has `itemQuantity: hearts, max: 0` as its unlock condition and plays a game-over whisper. A "you win" circle unlocks after visiting all safe targets.

### InvisibleFrogs (item collection as win counter)
Scatter hidden circles across the map, each visible only via `visibleConditions`. Each circle adds a `frog` item on visit and locks itself (`lockOnEnd`). A "win" circle has `itemQuantity: frog, min: N` as its unlock condition and plays a completion whisper when the player has collected all N frogs.

---

## Editor Capabilities

The editor is a full in-browser authoring tool. Activated by `?editor=true` in the URL.

### Circle placement
Tap + to enter placement mode. Tap the map to drop a circle. Drag to reposition before confirming.

### Circle properties panel
All properties are live-editable and autosaved:

| Property | Type | Default | Notes |
|---|---|---|---|
| Name | text | `circle-N` | Unique identifier — used in conditions and actions |
| Lat / Lng | number | placement point | Editable for fine positioning |
| Radius | number | 40 m | |
| Priority | number | 50 | Higher = takes precedence in overlaps |
| Starts locked | toggle | off | Overrides conditions at game start |
| Hide in editor | toggle | off | Session-only; resets on refresh |
| Hide from players | toggle | off | Sets `visible: false` |
| Repeat | number | 1 | 0 = loop, N = play N times |
| Play to end on exit | toggle | off | Greyed out when repeat = 0 |
| Lock when done | toggle | off | Greyed out when repeat = 0 or play-to-end is off |
| Whisper | text | `""` | Bare filename without extension (e.g. `rain`) |
| Volume | slider | 1.0 (100%) | Range 0–2.0 |
| Delete | button | — | Removes circle |

### Conditions
Four condition sections, each with an **Add condition** button opening a multi-step picker:

- **Becomes unlocked if** — `conditions` array (ALL must be true)
- **Becomes locked if** — `lockConditions` array (ANY triggers re-lock)
- **Becomes visible if** — `visibleConditions` array (ALL must be true)
- **Becomes hidden if** — `hideConditions` array (ALL must be true)

Condition types available in the picker: another circle's state (active/passive/locked), another circle's visited count, another circle's visibility, player has item (`hasItem`), item quantity (`itemQuantity` with ≥ and/or ≤ thresholds), time of day, day of week, specific date range.

Time/date conditions show with a 🕐 prefix in the Active conditions display. The picker for **Time of day** shows two `HH:MM` inputs with a note that ranges can cross midnight. **Day of week** shows seven checkboxes (at least one must be checked). **Specific date range** shows two date pickers defaulting to today; set both to the same date for a single day.

If a circle has the same condition in both its `conditions` (unlock) and `lockConditions` (re-lock) arrays, a red **⚠️ Conflicting conditions** warning appears at the top of the Active conditions panel.

### When visited actions
Actions that fire on passive → active (player entry). Two categories in the picker:

- **Circle actions** (Lock / Unlock / Show on map / Hide from map) — stored as `visited` conditions on the target circle
- **Item actions** (Add item / Remove item) — stored in `inventoryActions` on the source circle

Displayed as e.g. "Unlock: circle-b" or "Add item: key".

### When exited actions
`exitActions` — actions on physical player exit. 2-step picker: choose action type → choose target circle or item. Six types: Lock / Unlock / Show / Hide (circle targets) and Add item / Remove item (item targets). Displayed as e.g. "On exit lock: circle-b" or "On exit add item: frog".

### Items editor (🎒)
Tap the backpack button in the editor header to open the items sheet. Items are the named quantities that circles can add/remove and conditions can test.

Each item has:
- **Icon** — any single emoji or character shown in the inventory row
- **Name** — unique identifier used in actions and conditions
- **Starting quantity** — quantity at game start (0 = not present)

Tap an item row to expand it for inline editing (icon input, name input, quantity stepper). Tap **Done** to collapse. Tap the trash icon to delete. Tap **+ Add item** to create a new item (opens immediately in edit mode).

Changes autosave. Items are included in the exported haunt JSON and load correctly in play mode and on import.

### Haunt metadata
`id`, `title`, `author`, `version`, `created`, `modified`. `id` and `created` are preserved on import. `version` is incremented on export.

### Autosave
Draft saved to `localStorage["ghost-circles-draft"]` after every change (debounced). "Draft saved" flash confirms.

### Export (↑)
Generates a compressed play URL using LZString. Modal shows the URL (tap to copy) and an audio checklist (HEAD requests per whisper file — green if found, red if missing).

### Import (↓)
Accepts a haunt URL or raw JSON. Rebuilds map, preserves `id`/`created`, autosaves.

### Clear draft (🗑)
Resets to blank canvas — new `id`, empty circles and items, "New Haunt" title.

---

## Haunt Format

Haunts are serialised as JSON and LZString-compressed into the `?haunt=` URL parameter.

```json
{
  "metadata": {
    "id":       "abc123",
    "title":    "My Haunt",
    "author":   "",
    "version":  3,
    "created":  "2026-04-01T10:00:00.000Z",
    "modified": "2026-05-05T14:30:00.000Z"
  },
  "circles": [
    {
      "name":        "circle-1",
      "lat":         55.9950,
      "lng":         12.5510,
      "radius":      40,
      "priority":    50,
      "visible":     true,
      "startsLocked": false,
      "repeat":      1,
      "playToEnd":   false,
      "lockOnEnd":   false,
      "volume":      1.0,
      "whisper":     "rain",
      "conditions":        [],
      "lockConditions":    [],
      "visibleConditions": [],
      "hideConditions":    [],
      "inventoryActions":  [],
      "exitActions":       []
    }
  ],
  "items": [
    {
      "name":         "key",
      "icon":         "🗝️",
      "quantity":     0,
      "startPresent": false,
      "audio":        ""
    }
  ]
}
```

Runtime-only fields (`lockedByEnd`, `pendingExitActions`) are never serialised.

---

## Play Mode

When a `?haunt=` URL parameter is present:
1. LZString-decompress and parse the JSON
2. Load circles into `CIRCLE_DEFS` and items into `PLAYER_ITEMS`
3. Initialise `playerInventory` from item `startPresent` / `quantity` values
4. Fetch all whisper audio files
5. Show a slim dark-green title bar with the haunt title
6. Run the standard GPS/proximity engine

---

## File Structure

```
ghost-circles/
  index.html          — the entire app (engine + editor, ~3500 lines)
  manifest.json       — PWA manifest (name, icons, display mode)
  sw.js               — service worker (caching, auto-update on deploy)
  _headers            — Netlify headers (no-cache for sw.js + index.html, audio/mp4 MIME type)
  CLAUDE.md           — this file
  audio/
    near.m4a          — proof-of-concept whisper
    trigger.m4a       — archived
    youfoundit.m4a    — archived
```

---

## Naming Conventions

| Term | Meaning |
|---|---|
| **Ghost circle** | A geofenced circle on the map tied to GPS coordinates |
| **Whisper** | An audio file triggered when a player enters a ghost circle |
| **Haunt** | A complete experience: a named collection of circles, items, and metadata |
| **Player marker** | The blue (fresh) or grey (stale) dot showing GPS position |
| **Locked** | Circle whose conditions are not yet met — grey, ignored by proximity |
| **Passive** | Unlocked circle the player is outside — red |
| **Active** | Circle the player is inside and which holds highest priority — green |
| **Visited** | Number of times a circle has transitioned from passive to active |
| **Item** | A named quantity in the player's inventory; used in actions and conditions |
| **lockedByEnd** | Runtime flag: circle was locked by lockOnEnd or an exit-action lock; suppresses auto-unlock |
| **pendingExitActions** | Runtime flag: player physically exited a playToEnd circle; exitActions deferred to audio end |

---

## Code Conventions

- `CIRCLE_DEFS` — array of circle definition objects (the haunt data)
- `PLAYER_ITEMS` — array of item definitions `{ name, icon, quantity, startPresent, audio }`
- `playerInventory` — plain object mapping item name → current quantity
- `circleRuntime` — object keyed by name: `{ def, leafletCircle, state, inRange, visited, onMap, activeGain, activeSource, playsRemaining, lockedByEnd, pendingExitActions }`
- `playToEndLock` — module-level variable; holds the `rt` of any currently-playing playToEnd whisper, or `null`
- `lastKnownLatLng` — module-level `{ lat, lng }` updated on every GPS fix; used by `onPlayToEndLockReleased`, the post-initAudio scan, and the 60-second time-condition timer
- `firstGpsFix` — boolean, true until the first `onPosition` call; triggers an immediate `checkUnlocks` pass for time-based conditions on first fix
- `_hauntSaveKey` — `ghost-circles-save-<hauntId>` in play mode, `null` in editor mode; guards `savePlayState()` and the Start Over handler
- `savePlayState()` — writes visited counts, circle states, lockedByEnd flags, and inventory to localStorage; no-op when `_hauntSaveKey` is null
- `evaluateCondition(cond)` — evaluates a single condition object; supports all types
- `evaluateConditions(def)` — returns true if ALL unlock conditions are satisfied
- `evaluateLockConditions(def)` — returns true if ANY lock condition is met
- `computeVisibility(def)` — returns whether a circle should currently be on the map
- `checkUnlocks(userLatLng)` — promotes locked circles whose conditions are met; skips `lockedByEnd` circles unconditionally; returns true if anything promoted
- `checkProximity(lat, lng)` — five-pass algorithm: (1) update inRange → (2) find maxPriority → (3) resolve transitions + inventoryActions + checkUnlocks + exitActions → recurse if unlocks → (4) re-lock/re-unlock (skips lockedByEnd) → (5) update dynamic map visibility
- `startWhisperPlayback(rt)` — checks playToEndLock, claims it if playToEnd, starts audio per repeat setting
- `playNextInSequence(rt, gain)` — chains sequential plays via `onended`; on natural completion fires lockOnEnd, `fireExitActions`, then `onPlayToEndLockReleased`
- `stopWhisperPlayback(rt)` — if playToEnd: lets audio run to end, sets `pendingExitActions` flag on exit, fires `fireExitActions` then `onPlayToEndLockReleased` in onended; otherwise fades over 1s and stops
- `fireExitActions(rt)` — executes all `exitActions` on the circle's def; handles lock/unlock/show/hide/addItem/removeItem; resets `rt.pendingExitActions`
- `onPlayToEndLockReleased()` — scans circleRuntime for active+inRange+silent circles and starts their whispers; then calls checkProximity for any remaining passive circles
- `initAudio()` — creates AudioContext; pre-loads all whisper and item audio files using callback-form `decodeAudioData` wrapped in a Promise; logs per-buffer success/failure with duration
- `startTracking()` — starts (or restarts) the GPS watcher
- `renderInventory()` — re-renders the inventory row
- `playItemAudio(item)` — plays an item's description audio once
- `saveDraft()` / `saveDraftDebounced()` — writes haunt state (circles + items) to localStorage
- `showFlash(msg, durationMs)` — generic flash message helper
- `generateId()` — returns a random hex string used as haunt `id`
- `EDITOR_MODE` — true when `?editor=true` is in the URL
- `DEBUG_MODE` — true when `?debug=true` is in the URL

---

## Audio Files

- All whispers must be **`.m4a`** format (required for Web Audio API on iOS Safari)
- The `whisper` field stores the **bare filename without extension** (e.g. `rain`, not `rain.m4a`)
- The engine always fetches from `audio/<name>.m4a` at runtime
- `whisperBuffers` is keyed by bare name for whispers and full path for item audio (e.g. `whisperBuffers['rain']`)
- Audio files must be **uploaded manually** to the `audio/` folder in the GitHub repo
- The export modal audio checklist (HEAD requests) shows which files are present/missing before sharing

---

## Deployment

Deployed to Netlify as a static site (no build step). The `_headers` file controls HTTP headers:
- `sw.js` and `index.html` served with `no-cache` so browsers always fetch fresh
- `audio/*.m4a` served with `Content-Type: audio/mp4` — required for iOS Safari's Web Audio API

---

## Debug Mode

Append `?debug=true` to the URL to show the debug overlay (top-left corner). Rolling log of last 8 entries:
- Buffer loads: `buf ok: <name> (<s>)` / `buf ERR: <name> — <reason>`
- Whisper play: `play: <name> ×<N> dur=<s> (ctx:<state>)`
- Play blocked: `play BLOCKED: <name> — playToEnd lock held by <name>`
- Play skipped: `play SKIP: <name> — no buffer (keys: <list>)`
- Lock claimed/released: `playToEnd lock claimed/released: <name>`
- Lock on end: `lockedByEnd: <name>`
- Unlock skipped: `unlock SKIP: <name> (lockedByEnd)`
- Pass 4 skip: `pass4 SKIP: <name> (lockedByEnd)`
- Exit actions: `exit action: lock/unlock/show/hide/addItem/removeItem → <target>`
- Time tick: `time tick: checking <N> locked circles`

---

## Known iOS Safari Quirks Solved

- **Audio autoplay**: `AudioContext` created and resumed inside the Tap to Start gesture
- **AudioContext suspension on screen lock**: `audioCtx.resume()` called on `visibilitychange`
- **GPS dying after screen off**: watcher unconditionally restarted on `visibilitychange`
- **GPS blocked by audio errors**: `startTracking()` and `initAudio()` run in parallel
- **MIME type for .m4a**: `Content-Type: audio/mp4` set via `_headers`
- **`decodeAudioData` resolving early**: Safari's promise-based `decodeAudioData` can resolve before the buffer is ready; fixed by using `new Promise((res, rej) => audioCtx.decodeAudioData(arrayBuffer, res, rej))` (explicit callback form)
- **Start-inside-circle whisper not playing**: GPS fix can arrive before `initAudio` resolves; fixed by scanning for active+inRange+silent circles in the `initAudio().then()` handler

---

## Known Limitations

- **No file upload in editor**: audio `.m4a` files must be added to the `audio/` folder in the GitHub repo manually. The export checklist will flag missing files.
- **No circle list panel**: circles can only be selected by tapping them on the map. No list view.
- **lockedByEnd is session-only in the haunt JSON**: the flag is not serialised, but it is included in the localStorage play-state save, so it persists correctly across reloads for players.


---

## Next Steps

- **Circle list panel**: scrollable list of all circles with tap-to-select and rename
- **Server storage**: haunts stored server-side with short share codes instead of long compressed URLs
