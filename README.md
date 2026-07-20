# TouchFence

A personal touch-blocker for a cracked screen. Draw a freeform shape in the
working (bottom) half of the screen, the app converts it into at most 128
rectangles, and a D-pad nudges the mask up over the damaged area. A foreground
service then keeps one tiny overlay window per rectangle alive, eating every
touch inside the mask while everything outside behaves normally.

No dependencies, no ads, no network permission. Four Kotlin files.

## Build and install

1. Install Android Studio (free) and open this folder (File > Open).
2. Let it sync â€” it will download Gradle 8.4 and SDK 34 on first run.
3. On the phone: Settings > About phone > tap Build number 7 times, then
   Developer options > enable USB debugging.
4. Plug in the phone, accept the debugging prompt, press Run (green â–¶).

Command line alternative: `./gradlew installDebug` with the phone plugged in.

## Using it

1. Open TouchFence, allow notifications, tap "Grant overlay permission"
   (Settings opens â€” enable it for TouchFence, go back).
2. Tap "Draw or edit mask". Scribble or circle the shape of the dead zone
   anywhere below the dashed line. Thick scribbles fill; closed loops fill
   their interior. Undo/Clear as needed, then "Done".
3. The shape becomes red rectangles. Hold the D-pad arrows to slide the whole
   mask up onto the damaged area (Step toggles 20 px / 120 px). "Save mask".
4. Tap "Start blocking". The zones tint faint red and eat all touches.

The persistent notification has **Pause 60s** (lifts the mask briefly â€” your
escape hatch if a zone covers something you need) and **Stop**.

## Tweakables

All in one place at the top of each file:

`DrawMaskActivity.kt` â€” `GUARD_FRACTION` (how much of the screen accepts
drawing, default bottom 50%), `MAX_RECTS` (128), `BRUSH_PX` (scribble
thickness), `STEP_FINE` / `STEP_COARSE` (D-pad step sizes), `START_CELL_PX`
(rasterization resolution).

`BlockerService.kt` â€” `ZONE_COLOR` (tint of active zones; lower the leading
byte for more invisible), `PAUSE_MS` (pause length).

`MaskStore.kt` â€” `MANUAL_OVERRIDE`: hardcode exact rectangles in code and skip
the drawing UI entirely. Example inside the file.

## Known limits (Android, not fixable in an app)

- Overlays cannot cover the lock screen, the status bar, or the notification
  shade â€” use fingerprint unlock if the crack sits over the PIN pad.
- The mask is portrait-only; keep auto-rotate off while blocking.
- If you ever block yourself out completely: reboot into safe mode
  (hold power, then long-press "Power off"), which disables all overlays.
