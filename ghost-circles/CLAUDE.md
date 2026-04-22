# Ghost Circles — Project Brief

## What is Ghost Circles?

Ghost Circles is a location-based Progressive Web App (PWA) for iPhone. Players walk around in the real world and encounter invisible "ghost circles" — geofenced areas anchored to GPS coordinates. When a player enters a circle, the app triggers a sensory response: a sound (called a whisper), a haptic vibration, and a visual colour change on the map. The concept is an immersive, ambient experience somewhere between a walking audio tour and a ghost hunt.

## Current State (v3.3) — Editor Release

The core engine is complete and proven. The visual editor is complete: experiences (haunts) can be authored entirely in the browser without touching code, exported as compressed URLs, and shared with players. v3.3 is the stable authoring platform on which experiences are built.

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

`min` and `max` can be combined in one `itemQuantity` condition.

### Priority system
When the player is inside multiple unlocked circles simultaneously, only the circle(s) with the highest `priority` value become active. Lower-priority circles remain passive — no whisper, no vibration. On a tie, all tied circles activate simultaneously. Priority demotion (a higher-priority circle entering range) does not trigger an exit vibration.

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

Requires `playToEnd: true` to be useful (otherwise the circle returns to passive when the player exits, and locks only if the player stays inside until the audio ends). The editor greys out `lockOnEnd` unless `playToEnd` is also on.

### lockedByEnd flag
`rt.lockedByEnd` is a runtime-only flag (not saved in the haunt JSON). When set, `checkUnlocks` and Pass 4 skip the circle unconditionally — the circle cannot be re-promoted regardless of its `conditions` array. The only way to clear `lockedByEnd` is via a future explicit "unlock" action. This prevents empty-conditions circles from being auto-promoted on every GPS tick after `lockOnEnd` fires.

### Volume
`volume` per circle, range 0–2.0, default 1.0 (100%). Applied to the GainNode at playback start. Values above 1.0 boost the signal.

### Inventory actions
`inventoryActions` on circles: optional array of actions that fire on passive → active (i.e. on entry):
- `{ type: "add", item: <name> }` — increments quantity by 1
- `{ type: "remove", item: <name> }` — decrements quantity by 1 (minimum 0)

### Player inventory
`PLAYER_ITEMS` array defines items. Each item has: `name` (key), `icon` (emoji), `quantity` (integer), `startPresent` (bool), `audio` (file path or null). The icon is rendered once per unit of quantity — 3 hearts = ❤️❤️❤️. An item is considered present when quantity > 0.

### Immediate activation on unlock
If a player is already standing inside a circle when it unlocks, it activates immediately — no need to exit and re-enter. `checkProximity` re-runs recursively on unlock.

### iPhone reliability
- **Screen Wake Lock**: requested after Tap to Start, reacquired automatically on return from background
- **GPS recovery**: on any non-permission GPS error, the watcher restarts after 3 seconds. On `visibilitychange` (screen back on), the GPS watcher is always stopped and restarted immediately
- **GPS staleness indicator**: the player dot turns grey if no fix received for more than 10 seconds
- **AudioContext recovery**: on `visibilitychange`, `audioCtx.resume()` is called explicitly to lift iOS's automatic suspension
- **Tap to Start overlay**: required to unlock iOS audio and GPS simultaneously. Creates AudioContext inside the tap gesture

---

## Editor Capabilities

The editor is a full in-browser authoring tool for creating and editing haunts. It is activated by appending `?editor=true` to the URL (or via the Netlify redirect at `/editor`).

### Circle placement
Tap the + button to enter placement mode. Tap the map to drop a new circle at that location. A drag handle appears to reposition it before confirming. Tap Confirm to place, Cancel to abort.

### Circle properties panel
Tapping a circle on the map opens its property panel (slides up from the bottom). All properties are editable live and autosaved to the draft:

| Property | Type | Default | Notes |
|---|---|---|---|
| Name | text | `circle-N` | Unique identifier — used in conditions |
| Lat / Lng | number | placement point | Editable for fine positioning |
| Radius | number | 40 m | |
| Priority | number | 50 | Higher = takes precedence in overlaps |
| Starts locked | toggle | off | Circle begins locked regardless of conditions |
| Hide in editor | toggle | off | Session-only; resets on refresh |
| Hide from players | toggle | off | Sets `visible: false` — permanently hidden from map in play |
| Repeat | number | 1 | 0 = loop, N = play N times |
| Play to end on exit | toggle | off | Greyed out when repeat = 0 |
| Lock when done | toggle | off | Greyed out when repeat = 0 or play-to-end is off |
| Whisper | text | `""` | Bare filename without extension (e.g. `rain`) |
| Volume | slider | 1.0 (100%) | Range 0–2.0 |
| Delete | button | — | Removes circle from map and runtime |

### Conditions
Four condition sections appear below the properties:

- **Becomes unlocked if** — `conditions` array (ALL must be true)
- **Becomes locked if** — `lockConditions` array (ANY triggers re-lock)
- **Becomes visible if** — `visibleConditions` array (ALL must be true)
- **Becomes hidden if** — `hideConditions` array (ALL must be true)

Each section has an **Add condition** button that opens a 3-step picker: choose condition type → choose target circle or item → choose threshold. Existing conditions can be removed individually.

### When visited actions
`inventoryActions` array — actions that fire when the circle is entered. Each action has a type (`add` or `remove`) and an item name. Editable via the **When visited** section of the panel.

### Haunt metadata
The editor header shows the haunt name (editable inline). Metadata stored alongside circles:
- `id` — UUID generated on first save, preserved on import
- `title` — from the haunt name field
- `author` — (currently not editable in UI, preserved from import)
- `version` — integer, incremented on each export
- `created` — ISO timestamp, set on first save, preserved on import
- `modified` — ISO timestamp, updated on every save

### Autosave
The draft is autosaved to `localStorage["ghost-circles-draft"]` after every change (debounced). On reload, the draft is restored. The "Draft saved" flash confirms each save.

### Export (↑ button)
Opens the export modal. Generates a compressed haunt URL using LZString (`compressToEncodedURIComponent`). The modal shows:
- The play URL (tap to copy)
- An audio checklist: for each whisper used, a HEAD request checks whether the file exists in `audio/`. Green = found, red = missing.

### Import (↓ button)
Opens the import modal. Accepts a haunt URL (paste the full URL with `?haunt=` param) or raw JSON. On import: circles are rebuilt on the map, `id` and `created` from the current draft are preserved (haunt identity continuity), metadata fields from the imported haunt are applied, the draft is autosaved.

### Clear draft (🗑 button)
Resets the editor to a blank canvas: clears all circles, resets the haunt name to "New Haunt", generates a new `id`.

---

## Haunt Format

Haunts are serialised as JSON, then LZString-compressed into the `?haunt=` URL parameter.

```json
{
  "metadata": {
    "id":       "abc123",
    "title":    "My Haunt",
    "author":   "",
    "version":  3,
    "created":  "2026-04-01T10:00:00.000Z",
    "modified": "2026-04-22T14:30:00.000Z"
  },
  "circles": [
    {
      "name":       "circle-1",
      "lat":        55.9950,
      "lng":        12.5510,
      "radius":     40,
      "priority":   50,
      "visible":    true,
      "startsLocked": false,
      "repeat":     1,
      "playToEnd":  false,
      "lockOnEnd":  false,
      "volume":     1.0,
      "whisper":    "rain",
      "conditions": [],
      "lockConditions": [],
      "visibleConditions": [],
      "hideConditions": [],
      "inventoryActions": []
    }
  ],
  "items": []
}
```

`lockedByEnd` is runtime-only and is never serialised.

---

## Play Mode

When a haunt URL (`?haunt=`) is present, the engine:
1. Decompresses and parses the JSON
2. Loads circles into `CIRCLE_DEFS`
3. Fetches all whisper audio files listed in the circles
4. Shows a slim dark-green title bar at the top of the map with the haunt title
5. Runs the standard GPS/proximity engine

The play URL can be opened directly on any iPhone — no install required (PWA prompt appears separately).

---

## File Structure

```
ghost-circles/
  index.html          — the entire app (engine + editor, ~3000 lines)
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
| **lockedByEnd** | Runtime flag: circle was self-locked by lockOnEnd; suppresses auto-unlock |

---

## Code Conventions

- `CIRCLE_DEFS` — array of circle definition objects (the haunt data)
- `PLAYER_ITEMS` — array of item definitions (name, icon, quantity, startPresent, audio)
- `playerInventory` — plain object mapping item name → current quantity
- `circleRuntime` — object keyed by name: `{ def, leafletCircle, state, inRange, visited, onMap, activeGain, activeSource, playsRemaining, lockedByEnd }`
- `playToEndLock` — module-level variable; holds the `rt` of any currently-playing playToEnd whisper, or `null`
- `evaluateCondition(cond)` — evaluates a single condition object; supports all types
- `evaluateConditions(def)` — returns true if ALL unlock conditions are satisfied
- `evaluateLockConditions(def)` — returns true if ANY lock condition is met
- `computeVisibility(def)` — returns whether a circle should currently be on the map
- `checkUnlocks(userLatLng)` — promotes locked circles whose conditions are met; skips `lockedByEnd` circles unconditionally; returns true if anything promoted
- `checkProximity(lat, lng)` — five-pass algorithm: (1) update inRange → (2) find maxPriority → (3) resolve transitions + fire inventoryActions + checkUnlocks → recurse if unlocks → (4) re-lock/re-unlock (skips lockedByEnd) → (5) update dynamic map visibility
- `startWhisperPlayback(rt)` — checks playToEndLock, claims it if playToEnd, starts audio per repeat setting
- `playNextInSequence(rt, gain)` — chains sequential plays via `onended`; fires lockOnEnd on natural completion
- `stopWhisperPlayback(rt)` — if playToEnd: lets audio run to end, replaces onended for cleanup; otherwise fades over 1s and stops
- `initAudio()` — creates AudioContext; pre-loads all whisper and item audio files; logs per-buffer success/failure with duration
- `startTracking()` — starts (or restarts) the GPS watcher
- `renderInventory()` — re-renders the inventory row
- `playItemAudio(item)` — plays an item's description audio once
- `saveDraft()` / `saveDraftDebounced()` — writes haunt state to localStorage
- `showFlash(msg, durationMs)` — generic flash message helper
- `generateId()` — returns a random hex string used as haunt `id`
- `EDITOR_MODE` — true when `?editor=true` is in the URL
- `DEBUG_MODE` — true when `?debug=true` is in the URL

---

## Audio Files

- All whispers must be **`.m4a`** format (required for Web Audio API on iOS Safari)
- The `whisper` field in circle definitions stores the **bare filename without extension** (e.g. `rain`, not `rain.m4a`)
- The engine always fetches from `audio/<name>.m4a` at runtime
- `whisperBuffers` is keyed by bare name (e.g. `whisperBuffers['rain']`)
- Audio files must be **uploaded manually** to the `audio/` folder in the GitHub repo — the editor has no file upload capability
- The export modal audio checklist (HEAD requests) shows which files are present/missing before sharing

---

## Deployment

Deployed to Netlify as a static site (no build step). The `_headers` file controls HTTP headers:
- `sw.js` and `index.html` served with `no-cache` so browsers always fetch fresh
- `audio/*.m4a` served with `Content-Type: audio/mp4` — required for iOS Safari's Web Audio API

---

## Debug Mode

Append `?debug=true` to the URL to show the debug overlay (top-left corner):
- Current `AudioContext.state`
- Timestamped rolling log (last 8 entries):
  - Audio context creation and resume
  - Per-buffer load: `buf ok: <name> (<duration>s)` or `buf ERR: <name> — <reason>`
  - Whisper play: `play: <name> ×<N> dur=<s> (ctx:<state>)`
  - Play blocked: `play BLOCKED: <name> — playToEnd lock held by <name>`
  - Lock claimed/released: `playToEnd lock claimed/released: <name>`
  - Lock on end: `lockedByEnd: <name>`
  - Unlock skipped: `unlock SKIP: <name> (lockedByEnd)`
  - Pass 4 skip: `pass4 SKIP: <name> (lockedByEnd)`
  - GPS restarts and visibility changes

All logging runs in production — it just writes to a hidden element.

---

## Known iOS Safari Quirks Solved

- **Audio autoplay**: `AudioContext` created and resumed inside the Tap to Start gesture
- **AudioContext suspension on screen lock**: `audioCtx.resume()` called on `visibilitychange`
- **GPS dying after screen off**: watcher unconditionally restarted on `visibilitychange`
- **GPS blocked by audio errors**: `startTracking()` and `initAudio()` run in parallel
- **MIME type for .m4a**: `Content-Type: audio/mp4` set via `_headers`

---

## Known Limitations

- **No file upload in editor**: audio `.m4a` files must be added to the `audio/` folder in the GitHub repo manually. The export checklist will flag missing files.
- **No items editor**: `PLAYER_ITEMS` and item-related conditions/actions exist in the engine but there is no UI to author them — requires direct code editing.
- **No circle list panel**: circles can only be selected by tapping them on the map. There is no list view or search.
- **No time/date conditions**: the condition system has no awareness of real-world time or date.
- **lockedByEnd is session-only**: the flag is not saved in the haunt JSON and resets on page reload. A locked-by-end circle will return to passive on the player's next session.
- **Single haunt per session**: only one haunt can be active at a time. There is no concept of haunt switching or saved progress.
- **Play-state not persisted**: visited counts, inventory, and circle states are lost on page reload.

---

## Next Steps

- **Items editor**: UI to define items (name, icon, audio) and wire them to circle actions and conditions
- **Time/date conditions**: `{ type: "after", time: "18:00" }`, `{ type: "before", time: "21:00" }`, `{ type: "onDate", date: "2026-06-21" }`
- **When exited actions**: `exitActions` array on circles, mirroring `inventoryActions` but firing on active → passive
- **Circle list panel**: scrollable list of all circles in the haunt with tap-to-select and reorder
- **Explicit unlock action**: `{ type: "unlock", circle: <name> }` inventory/exit action that clears `lockedByEnd` on another circle
- **Play-state persistence**: save visited counts, inventory, and circle states to `localStorage` so players can resume a session
- **Server storage**: eventually, haunts stored server-side with short share codes instead of long compressed URLs
