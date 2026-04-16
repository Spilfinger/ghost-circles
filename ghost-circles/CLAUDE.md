# Ghost Circles — Project Brief

## What is Ghost Circles?

Ghost Circles is a location-based Progressive Web App (PWA) for iPhone. Players walk around in the real world and encounter invisible "ghost circles" — geofenced areas anchored to GPS coordinates. When a player enters a circle, the app triggers a sensory response: a sound (called a whisper), a haptic vibration, and a visual colour change on the map. The concept is an immersive, ambient experience somewhere between a walking audio tour and a ghost hunt.

## Current State (v1.2)

The app has two live ghost circles with full proximity detection, audio playback, and a lock/unlock progression system. The UI is clean with no debug elements.

### What works
- Full-screen Leaflet.js map (OpenStreetMap tiles) with the player's live GPS position shown as a blue dot
- Two ghost circles on the map — lindevej-a (red, 50 m radius) and lindevej-b (grey/locked, 30 m radius)
- Lock/unlock state machine: lindevej-b is locked at start and only becomes active after the player enters lindevej-a
- Proximity detection on all unlocked circles: circle turns green on enter, red on exit, phone vibrates, whisper plays
- Each circle has its own named whisper file in `audio/`
- "Tap to Start" full-screen overlay on load — required to unlock iOS audio and GPS simultaneously
- Web Audio API for audio playback (more reliable on iOS Safari than the HTML Audio element)
- All whisper files are pre-loaded at startup in parallel, keyed by file path
- PWA manifest and service worker for offline capability and home screen installation
- Auto-update: when a new version is deployed to Netlify, the service worker detects the change and reloads automatically
- Version pill in the top-right corner — bump on every deploy to confirm which version is live on the device

## File Structure

```
ghost-circles/
  index.html          — the entire app (map, GPS, proximity logic, audio, UI)
  manifest.json       — PWA manifest (name, icons, display mode)
  sw.js               — service worker (caching, auto-update on deploy)
  _headers            — Netlify headers (cache control for sw.js + index.html, MIME type for audio)
  CLAUDE.md           — this file
  audio/
    trigger.m4a       — whisper for lindevej-a (plays on enter)
    youfoundit.m4a    — whisper for lindevej-b (plays on enter)
```

## Naming Conventions

| Term | Meaning |
|---|---|
| **Ghost circle** | A geofenced circle on the map tied to GPS coordinates. Has a name, radius, whisper, and lock state. |
| **Whisper** | An audio file triggered when a player enters a ghost circle. Stored in `audio/`, named descriptively (not numerically). |
| **Player marker** | The blue dot showing the user's current GPS position on the map. |
| **Locked** | A circle that is not yet active. Shown in grey. Ignored by proximity detection until unlocked. |
| **Active** | An unlocked circle waiting to be entered. Shown in red. |
| **Inside** | The player is currently within the circle's radius. Shown in green. |

### Circle colours
| State | Colour | Hex |
|---|---|---|
| Locked | Grey | `#9e9e9e` |
| Active (unlocked, outside) | Red | `#e53935` |
| Inside | Green | `#2e7d32` |

### Code conventions
- `CIRCLE_DEFS` — the array of circle definition objects (name, lat, lng, radius, whisper, unlockedBy)
- `circleRuntime` — object keyed by circle name, holding live state: `{ def, leafletCircle, inside, locked }`
- `unlockCircle(name)` — marks a circle active and turns it red
- `playWhisper(file)` — plays a pre-loaded audio buffer by file path
- `initAudio()` — creates the AudioContext and pre-loads all whisper files
- `checkProximity()` — called on every GPS update; iterates over all circles
- `startTracking()` — starts the GPS watcher

## Adding a New Circle

1. Add an entry to `CIRCLE_DEFS` in `index.html`:
   ```js
   {
     name:       'my-circle',
     lat:        55.9950,
     lng:        12.5510,
     radius:     40,
     whisper:    'audio/mywhisper.m4a',
     unlockedBy: 'lindevej-b',  // or null if active from the start
   }
   ```
2. Place the `.m4a` file in `audio/`
3. No other code changes needed — the circle is automatically rendered, tracked, and unlocked when its dependency is triggered

## Deployment

The app is deployed to Netlify as a static site (no build step — files are served as-is). The `_headers` file controls HTTP headers:
- `sw.js` and `index.html` are served with `no-cache` so the browser always fetches them fresh
- `audio/*.m4a` files are served with `Content-Type: audio/mp4` — required for iOS Safari's Web Audio API to decode them correctly

## Known iOS Safari Quirks Solved

- **Audio autoplay**: iOS blocks audio until a user gesture. Solved with the "Tap to Start" overlay, which creates and resumes the `AudioContext` inside the tap event handler.
- **AudioContext suspension**: iOS suspends `AudioContext` immediately after creation. Solved by calling `audioCtx.resume()` inside the tap gesture before fetching audio.
- **GPS blocked by audio errors**: In an earlier version, `startTracking()` was called with `await initAudio()`, meaning an audio failure would silently prevent GPS from starting. Fixed by running them in parallel — `startTracking()` fires immediately, `initAudio()` runs alongside it.
- **MIME type for .m4a**: iOS Safari's `decodeAudioData` rejects audio served without `Content-Type: audio/mp4`. Solved via `_headers`.
- **Filename with leading space**: The original audio file had a leading space in its name (` whisper-01.m4a`), causing a 404 on Netlify. Fixed by moving it to `audio/trigger.m4a`.

## Next Steps

- Add more circles and whisper files to extend the experience beyond lindevej-a and lindevej-b
- Consider externalising `CIRCLE_DEFS` into a `circles.json` file so the experience can be edited without touching code
- Add a visual indicator when a whisper is playing (e.g. a pulsing ring animation on the active circle)
- Explore a darker, more atmospheric map tile style (e.g. Stadia Alidade Smooth Dark) to match the ghost theme
- Add PWA icons (192×192 and 512×512) so the app installs cleanly to the iPhone home screen
- Consider whether the version pill should be hidden in production and only shown during testing
