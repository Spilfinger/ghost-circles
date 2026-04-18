# Ghost Circles — Project Brief

## What is Ghost Circles?

Ghost Circles is a location-based Progressive Web App (PWA) for iPhone. Players walk around in the real world and encounter invisible "ghost circles" — geofenced areas anchored to GPS coordinates. When a player enters a circle, the app triggers a sensory response: a sound (called a whisper), a haptic vibration, and a visual colour change on the map. The concept is an immersive, ambient experience somewhere between a walking audio tour and a ghost hunt.

## Current State (v2.5t)

Engine with full proximity detection, audio playback, lock/unlock and re-lock progression, priority system, whisper repeat control, and player inventory. Robust iPhone behaviour: screen wake lock, aggressive GPS recovery, audio context recovery. Debug overlay available via URL parameter.

### What works
- Full-screen Leaflet.js map (OpenStreetMap tiles) with the player's live GPS position shown as a blue dot
- Blue dot turns grey if no GPS fix received for more than 10 seconds (staleness indicator)
- `visible` property per circle: if false, the circle is not drawn on the map. All logic (state machine, audio, priority) runs regardless. Circles hidden at start can appear later — unlocking a locked circle automatically makes it visible
- Lock/unlock state machine: `conditions` array (ALL must be true) controls locked → passive. Immediate activation on unlock if the player is already inside
- **Re-locking**: `lockConditions` array (ANY one being true triggers re-lock). When met, a passive or active circle immediately reverts to locked (audio fades out if playing). When the condition clears and unlock conditions are still met, the circle restores to passive automatically
- Priority system: when a player is inside multiple unlocked circles simultaneously, only the highest-priority circle(s) activate. On a tie, all tied circles activate simultaneously. Lower-priority circles stay passive
- Each circle has a `visited` counter that increments each time the circle transitions passive → active
- `repeat` property per circle: `1` = play whisper once on entry (default), `0` = loop continuously until exit, `N` = play N times in sequence
- All audio routes through a GainNode so exit fades work correctly for both looping and sequential playback
- On exit (active → passive): whisper audio fades out over 1 second, then stops. No-op if audio already finished naturally
- **Player inventory**: `PLAYER_ITEMS` array defines items (name, icon emoji, quantity, startPresent, audio). Each item has an integer quantity. The icon is rendered once per unit of quantity (3 hearts = ❤️❤️❤️). An item is considered present when quantity > 0. `playerInventory` is a plain object mapping item name → current quantity
- `inventoryActions` on circles: optional array of actions that fire on passive → active. `{ type: "add", item }` increments quantity by 1; `{ type: "remove", item }` decrements by 1 (minimum 0)
- Condition types (work in both `conditions` and `lockConditions`): `visited`, `hasItem` (qty > 0), `isActive`, `itemQuantity` (supports `min` and/or `max` against current quantity)
- "Tap to Start" full-screen overlay — required to unlock iOS audio and GPS simultaneously
- Web Audio API for audio playback; all whisper and item audio files pre-loaded at startup in parallel
- Screen Wake Lock: requested after tap-to-start, reacquired automatically on return from background
- GPS recovery: on any non-permission GPS error, the watcher is cleared and restarted after 3 seconds. On `visibilitychange` (screen back on), the GPS watcher is always stopped and restarted immediately
- Audio context recovery: on `visibilitychange`, `audioCtx.resume()` is called explicitly to lift iOS's automatic suspension
- PWA manifest and service worker for offline capability and home screen installation
- Auto-update: service worker detects new deploys and reloads the page automatically
- Version pill in the top-right corner — bump on every deploy to confirm which version is live
- Debug overlay: hidden in production, shown at `?debug=true` — displays AudioContext state and a timestamped log of audio init, buffer loads, whisper plays, visibility changes, and GPS restarts

## File Structure

```
ghost-circles/
  index.html          — the entire app (map, GPS, proximity logic, audio, UI)
  manifest.json       — PWA manifest (name, icons, display mode)
  sw.js               — service worker (caching, auto-update on deploy)
  _headers            — Netlify headers (no-cache for sw.js + index.html, audio/mp4 MIME type)
  CLAUDE.md           — this file
  audio/
    trigger.m4a       — whisper for lindevej-a
    youfoundit.m4a    — whisper for lindevej-b
    near.m4a          — whisper for near
    goal.m4a          — whisper for goal
    ohno.m4a          — whisper for ohno
```

## Naming Conventions

| Term | Meaning |
|---|---|
| **Ghost circle** | A geofenced circle on the map tied to GPS coordinates. Has a name, radius, priority, whisper, and conditions. |
| **Whisper** | An audio file triggered when a player enters a ghost circle. Stored in `audio/`, named descriptively. |
| **Player marker** | The blue (fresh) or grey (stale) dot showing the user's current GPS position on the map. |
| **Locked** | A circle whose conditions are not yet met. Shown in grey. Ignored by proximity detection. |
| **Passive** | An unlocked circle the player is outside of. Shown in red. |
| **Active** | A circle the player is inside and which holds the highest priority. Shown in green. |
| **Visited** | The number of times a circle has transitioned from passive to active. |

### Circle colours
| State | Colour | Hex |
|---|---|---|
| Locked | Grey | `#9e9e9e` |
| Passive | Red | `#e53935` |
| Active | Green | `#2e7d32` |

### Code conventions
- `PLAYER_ITEMS` — array of item definition objects (name, icon, quantity, startPresent, audio)
- `playerInventory` — plain object mapping item name → current quantity; item is present when quantity > 0
- `renderInventory()` — re-renders the inventory row (icon rendered once per quantity unit); called at startup and after any inventory change
- `playItemAudio(item)` — plays an item's description audio once via Web Audio API
- `CIRCLE_DEFS` — array of circle definition objects (name, lat, lng, radius, priority, visible, repeat, whisper, conditions, lockConditions?, inventoryActions?)
- `circleRuntime` — object keyed by name: `{ def, leafletCircle, state, inRange, visited, onMap, activeGain, activeSource, playsRemaining }`
- `evaluateCondition(cond)` — evaluates a single condition object; supports `visited`, `hasItem`, `isActive`, `itemQuantity`
- `evaluateConditions(def)` — returns true if ALL unlock conditions are satisfied (uses `evaluateCondition`)
- `evaluateLockConditions(def)` — returns true if ANY lock condition is met (uses `evaluateCondition`)
- `checkUnlocks(userLatLng)` — promotes any locked circle whose conditions are now met to passive; pre-sets `inRange`; returns true if anything was promoted
- `checkProximity(lat, lng)` — four-pass algorithm: (1) update inRange → (2) find maxPriority → (3) resolve transitions, fire `inventoryActions` → recurse if unlocks → (4) apply re-lock/re-unlock rules
- `startWhisperPlayback(rt)` — starts audio for a circle according to its `repeat` setting, routed through a GainNode
- `playNextInSequence(rt, gain)` — chains sequential plays via `onended`; bails if session was cancelled
- `stopWhisperPlayback(rt)` — fades out active audio over 1 second and stops; no-op if already finished
- `initAudio()` — creates AudioContext and pre-loads all whisper and item audio files into `whisperBuffers`
- `startTracking()` — starts (or restarts) the GPS watcher; always clears existing watcher first
- `DEBUG_MODE` — true when `?debug=true` is in the URL

## Adding a New Circle

1. Add an entry to `CIRCLE_DEFS` in `index.html`:
   ```js
   {
     name:             'my-circle',
     lat:              55.9950,
     lng:              12.5510,
     radius:           40,
     priority:         50,             // higher = takes precedence in overlaps
     visible:          true,           // false = hidden on map until unlocked
     repeat:           1,              // 1 = once, 0 = loop, N = N times
     whisper:          'audio/mywhisper.m4a',
     conditions:       [],             // ALL must be true to unlock (locked → passive)
     // lockConditions: [],            // optional — ANY one being true re-locks (passive/active → locked)
     // inventoryActions: [],          // optional — [{ type: 'add'|'remove', item: <name> }]
   }
   // Condition types (work in both conditions and lockConditions):
   // { type: 'visited',      circle: <name>, minTimes: <n> }
   // { type: 'hasItem',      item: <name> }                    — quantity > 0
   // { type: 'isActive',     circle: <name> }
   // { type: 'itemQuantity', item: <name>, min: <n> }          — quantity ≥ n
   // { type: 'itemQuantity', item: <name>, max: <n> }          — quantity ≤ n
   ```
2. Place the `.m4a` file in `audio/`
3. No other code changes needed

## Adding a New Item

1. Add an entry to `PLAYER_ITEMS` in `index.html`:
   ```js
   { name: 'my-item', icon: '🔮', quantity: 1, startPresent: false, audio: null }
   ```
   `quantity` is the starting amount. The icon is rendered once per unit. Set `audio` to a file path if the item has description audio; otherwise `null`.
2. If the item has audio, place the `.m4a` file in `audio/`
3. No other code changes needed — `renderInventory()` picks it up automatically

## Priority System

When a player is inside multiple unlocked circles simultaneously:
- The circle(s) with the highest `priority` value become active (green) and play their whisper
- Lower-priority circles stay passive (red) even though the player is physically inside them — no whisper, no vibration
- If two circles share the same highest priority, both activate and both play simultaneously
- When the player exits the highest-priority circle, the algorithm re-evaluates and the next highest circle that the player is still inside becomes active
- Priority demotion (losing active status because a higher-priority circle was entered) does not trigger an exit vibration

## Deployment

Deployed to Netlify as a static site (no build step). The `_headers` file controls HTTP headers:
- `sw.js` and `index.html` served with `no-cache` so browsers always fetch them fresh
- `audio/*.m4a` served with `Content-Type: audio/mp4` — required for iOS Safari's Web Audio API

## Debug Mode

Append `?debug=true` to the URL to show the debug overlay (top-left corner). It displays:
- Current `AudioContext.state` (running / suspended / closed)
- Timestamped log: audio init, buffer loads, whisper plays, visibility changes, GPS restarts, resume calls and outcomes

All logging runs in production too — it just writes to a hidden element. No performance impact.

## Known iOS Safari Quirks Solved

- **Audio autoplay**: iOS blocks audio until a user gesture. Solved with the "Tap to Start" overlay, which creates and resumes the `AudioContext` inside the tap handler.
- **AudioContext suspension on screen lock**: iOS suspends the context when the screen turns off. Solved by calling `audioCtx.resume()` in the `visibilitychange` handler when the page becomes visible.
- **GPS silently dying after screen off**: `watchPosition` sometimes stops delivering updates after screen off/on. Solved by unconditionally stopping and restarting the watcher in `visibilitychange`.
- **GPS blocked by audio errors**: Fixed by running `startTracking()` and `initAudio()` in parallel — GPS can never be blocked by audio failure.
- **MIME type for .m4a**: iOS Safari's `decodeAudioData` rejects audio without `Content-Type: audio/mp4`. Solved via `_headers`.
- **Filename with leading space**: Original audio file had a leading space, causing 404 on Netlify. Fixed by moving to `audio/trigger.m4a`.

## Next Steps

- **Persistent inventory**: save `playerInventory` to `localStorage` so items survive page reloads and sessions
- **Audio fade-in**: mirror the exit fade with a fade-in on entry for smoother audio transitions
- **Ambient/background audio**: a persistent audio layer that plays regardless of circle state (e.g. atmospheric sound for the whole experience)
- **Visibility conditions**: extend the conditions system to support showing/hiding circles based on runtime state beyond just unlocking (e.g. time of day, visit count thresholds)
