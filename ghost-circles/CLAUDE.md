# Ghost Circles — Project Brief

## What is Ghost Circles?

Ghost Circles is a location-based Progressive Web App (PWA) for iPhone. Players walk around in the real world and encounter invisible "ghost circles" — geofenced areas anchored to GPS coordinates. When a player enters a circle, the app triggers a sensory response: a sound (called a whisper), a haptic vibration, and a visual colour change on the map. The concept is an immersive, ambient experience somewhere between a walking audio tour and a ghost hunt.

## Current State (v3.0) — Foundation Release

The core engine is complete and proven. All major systems have been designed, implemented, and tested in the field. v3.0 is the stable foundation on which experiences are built.

One proof-of-concept circle (`near`) is included. All engine capabilities are live and available for use via `CIRCLE_DEFS` and `PLAYER_ITEMS`.

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
`lockConditions` array on a circle — ANY one being true immediately re-locks the circle (fading audio if playing). When the lock condition clears and unlock conditions are still met, the circle automatically restores to passive. Evaluated on every GPS tick after the main proximity passes.

### Condition types
All types work in both `conditions` and `lockConditions`:

| Type | Syntax | True when |
|---|---|---|
| `visited` | `{ type: "visited", circle: <name>, minTimes: <n> }` | Named circle has been visited ≥ n times |
| `hasItem` | `{ type: "hasItem", item: <name> }` | Item quantity > 0 |
| `isActive` | `{ type: "isActive", circle: <name> }` | Named circle is currently active |
| `itemQuantity` | `{ type: "itemQuantity", item: <name>, min: <n> }` | Item quantity ≥ n |
| `itemQuantity` | `{ type: "itemQuantity", item: <name>, max: <n> }` | Item quantity ≤ n |

`min` and `max` can be combined in one condition.

### Priority system
When the player is inside multiple unlocked circles simultaneously, only the circle(s) with the highest `priority` value become active. Lower-priority circles remain passive — no whisper, no vibration. On a tie, all tied circles activate simultaneously. Priority demotion (a higher-priority circle entering range) does not trigger an exit vibration.

### Visibility
`visible: false` on a circle definition hides it from the map at start. All engine logic (state machine, audio, proximity) runs regardless. Unlocking a hidden circle automatically adds it to the map. `visible: true` circles appear immediately, including locked ones (shown grey).

### Whisper repeat
`repeat` property per circle: `1` = play once on entry (default), `0` = loop continuously until exit, `N` = play N times in sequence. All audio routes through a GainNode. On exit, audio fades out over 1 second.

### Player inventory
`PLAYER_ITEMS` array defines items. Each item has: `name` (key), `icon` (emoji), `quantity` (integer), `startPresent` (bool), `audio` (file path or null). The icon is rendered once per unit of quantity — 3 hearts = ❤️❤️❤️. An item is considered present when quantity > 0. `PLAYER_ITEMS` is empty in the base release.

`inventoryActions` on circles: optional array of actions that fire on passive → active:
- `{ type: "add", item: <name> }` — increments quantity by 1
- `{ type: "remove", item: <name> }` — decrements quantity by 1 (minimum 0)

### Immediate activation on unlock
If a player is already standing inside a circle when it unlocks, it activates immediately — no need to exit and re-enter. The proximity algorithm re-runs recursively on unlock.

### iPhone reliability
- **Screen Wake Lock**: requested after Tap to Start, reacquired automatically on return from background
- **GPS recovery**: on any non-permission GPS error, the watcher restarts after 3 seconds. On `visibilitychange` (screen back on), the GPS watcher is always stopped and restarted immediately
- **GPS staleness indicator**: the player dot turns grey if no fix received for more than 10 seconds
- **AudioContext recovery**: on `visibilitychange`, `audioCtx.resume()` is called explicitly to lift iOS's automatic suspension
- **Tap to Start overlay**: required to unlock iOS audio and GPS simultaneously. Creates AudioContext inside the tap gesture

## File Structure

```
ghost-circles/
  index.html          — the entire app (map, GPS, proximity logic, audio, UI)
  manifest.json       — PWA manifest (name, icons, display mode)
  sw.js               — service worker (caching, auto-update on deploy)
  _headers            — Netlify headers (no-cache for sw.js + index.html, audio/mp4 MIME type)
  CLAUDE.md           — this file
  audio/
    near.m4a          — whisper for near (proof-of-concept circle)
    trigger.m4a       — whisper for lindevej-a (archived)
    youfoundit.m4a    — whisper for lindevej-b (archived)
```

## Naming Conventions

| Term | Meaning |
|---|---|
| **Ghost circle** | A geofenced circle on the map tied to GPS coordinates |
| **Whisper** | An audio file triggered when a player enters a ghost circle |
| **Player marker** | The blue (fresh) or grey (stale) dot showing GPS position |
| **Locked** | Circle whose conditions are not yet met — grey, ignored by proximity |
| **Passive** | Unlocked circle the player is outside — red |
| **Active** | Circle the player is inside and which holds highest priority — green |
| **Visited** | Number of times a circle has transitioned from passive to active |

## Code Conventions

- `PLAYER_ITEMS` — array of item definitions (name, icon, quantity, startPresent, audio)
- `playerInventory` — plain object mapping item name → current quantity
- `renderInventory()` — re-renders the inventory row (icon once per quantity unit)
- `playItemAudio(item)` — plays an item's description audio once
- `CIRCLE_DEFS` — array of circle definitions (name, lat, lng, radius, priority, visible, repeat, whisper, conditions, lockConditions?, inventoryActions?)
- `circleRuntime` — object keyed by name: `{ def, leafletCircle, state, inRange, visited, onMap, activeGain, activeSource, playsRemaining }`
- `evaluateCondition(cond)` — evaluates a single condition; supports all types
- `evaluateConditions(def)` — ALL unlock conditions satisfied
- `evaluateLockConditions(def)` — ANY lock condition met
- `checkUnlocks(userLatLng)` — promotes locked circles whose conditions are now met; returns true if anything promoted
- `checkProximity(lat, lng)` — four-pass algorithm: (1) update inRange → (2) find maxPriority → (3) resolve transitions + fire inventoryActions → recurse if unlocks → (4) apply re-lock/re-unlock rules
- `startWhisperPlayback(rt)` — starts audio per `repeat` setting, routed through GainNode
- `playNextInSequence(rt, gain)` — chains sequential plays via `onended`; cancellation-safe
- `stopWhisperPlayback(rt)` — fades audio over 1 second and stops
- `initAudio()` — creates AudioContext; pre-loads all whisper and item audio files
- `startTracking()` — starts (or restarts) the GPS watcher
- `DEBUG_MODE` — true when `?debug=true` is in the URL

## Adding a New Circle

```js
// In CIRCLE_DEFS:
{
  name:       'my-circle',
  lat:        55.9950,
  lng:        12.5510,
  radius:     40,                  // metres
  priority:   50,                  // higher = takes precedence in overlaps
  visible:    true,                // false = hidden on map until unlocked
  repeat:     1,                   // 1 = once, 0 = loop, N = N times
  whisper:    'audio/mywhisper.m4a',
  conditions: [],                  // ALL must be true to unlock
  // lockConditions:   [],         // optional — ANY triggers re-lock
  // inventoryActions: [],         // optional — [{ type: 'add'|'remove', item: <name> }]
}
```

Place the `.m4a` in `audio/`. No other code changes needed.

## Adding a New Item

```js
// In PLAYER_ITEMS:
{ name: 'my-item', icon: '🔮', quantity: 1, startPresent: false, audio: null }
```

`audio` can be a file path for a description sound, or `null`. No other code changes needed.

## Deployment

Deployed to Netlify as a static site (no build step). The `_headers` file controls HTTP headers:
- `sw.js` and `index.html` served with `no-cache` so browsers always fetch fresh
- `audio/*.m4a` served with `Content-Type: audio/mp4` — required for iOS Safari's Web Audio API

## Debug Mode

Append `?debug=true` to the URL to show the overlay (top-left corner):
- Current `AudioContext.state`
- Timestamped log: audio init, buffer loads, whisper plays, visibility changes, GPS restarts, resume calls

All logging runs in production too — it just writes to a hidden element.

## Known iOS Safari Quirks Solved

- **Audio autoplay**: `AudioContext` created and resumed inside the Tap to Start gesture
- **AudioContext suspension on screen lock**: `audioCtx.resume()` called on `visibilitychange`
- **GPS dying after screen off**: watcher unconditionally restarted on `visibilitychange`
- **GPS blocked by audio errors**: `startTracking()` and `initAudio()` run in parallel
- **MIME type for .m4a**: `Content-Type: audio/mp4` set via `_headers`

## Next Steps

The engine is complete. The next major piece is the **Designer/Editor tool** — a way to author experiences (place circles, assign whispers, define conditions and inventory) without editing code directly. This is the foundation that tool will be built on.
