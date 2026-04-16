# Ghost Circles — Project Brief

## What is Ghost Circles?

Ghost Circles is a location-based Progressive Web App (PWA) for iPhone. Players walk around in the real world and encounter invisible "ghost circles" — geofenced areas anchored to GPS coordinates. When a player enters a circle, the app triggers a sensory response: a sound (called a whisper), a haptic vibration, and a visual colour change on the map. The concept is an immersive, ambient experience somewhere between a walking audio tour and a ghost hunt.

## Current State (v2.0)

Clean milestone build. Two live ghost circles with full proximity detection, audio playback, a lock/unlock progression system, and a priority system for overlapping circles. Robust iPhone behaviour: screen wake lock, aggressive GPS recovery, and audio context recovery after screen off/on. Debug overlay available via URL parameter.

### What works
- Full-screen Leaflet.js map (OpenStreetMap tiles) with the player's live GPS position shown as a blue dot
- Blue dot turns grey if no GPS fix received for more than 10 seconds (staleness indicator)
- Two ghost circles: lindevej-a (red, 50 m) and lindevej-b (grey/locked, 30 m)
- Lock/unlock state machine: lindevej-b unlocks after lindevej-a has been visited at least once
- Priority system: when a player is inside multiple overlapping circles, only the highest-priority circle(s) play their whisper and show as active (green). Lower-priority circles stay passive (red). On a tie, all tied circles activate simultaneously
- Each circle has a `visited` counter that increments each time the player enters it (passive → active)
- "Tap to Start" full-screen overlay — required to unlock iOS audio and GPS simultaneously
- Web Audio API for audio playback; all whisper files pre-loaded at startup in parallel
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
- `CIRCLE_DEFS` — array of circle definition objects (name, lat, lng, radius, priority, whisper, conditions)
- `circleRuntime` — object keyed by name: `{ def, leafletCircle, state, inRange, visited }`
- `evaluateConditions(def)` — returns true if all conditions for a circle are satisfied
- `checkUnlocks()` — promotes any locked circle whose conditions are now met to passive
- `checkProximity(lat, lng)` — three-pass algorithm: update inRange → find maxPriority → resolve state transitions
- `playWhisper(file)` — plays a pre-loaded AudioBuffer by file path
- `initAudio()` — creates AudioContext and pre-loads all whisper files
- `startTracking()` — starts (or restarts) the GPS watcher; always clears existing watcher first
- `DEBUG_MODE` — true when `?debug=true` is in the URL

## Adding a New Circle

1. Add an entry to `CIRCLE_DEFS` in `index.html`:
   ```js
   {
     name:       'my-circle',
     lat:        55.9950,
     lng:        12.5510,
     radius:     40,
     priority:   50,
     whisper:    'audio/mywhisper.m4a',
     conditions: [],             // or: [{ type: 'visited', circle: 'other-name', minTimes: 1 }]
   }
   ```
2. Place the `.m4a` file in `audio/`
3. No other code changes needed

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

- **Visibility state**: show/hide circles or change their appearance based on time of day, player history, or other runtime conditions
- **Audio playcount / repeat**: control how many times a whisper plays, or loop ambient audio while the player remains inside a circle
- **Player inventory**: track which circles the player has visited across sessions (e.g. using localStorage), and use that as a new condition type
