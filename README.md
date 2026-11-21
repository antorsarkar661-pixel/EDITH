# EDITH Launcher (low-RAM build)

A futuristic, HUD-style home screen for the **Lenovo Tab M10 HD (2 GB RAM, Android 10)**.
Pure Kotlin + framework views, one library (RecyclerView), no network, no services.

## What you get

- **Home:** ring clock (hour / minute / second arcs), tick marks, corner brackets, battery / RAM / storage / date readouts, "EDITH ONLINE" greeting, optional scan-line + radar sweep (~15 fps).
- **Dock:** 5 outlined circular icons. Long-press = remove / app info.
- **App drawer:** swipe up (or tap "SWIPE UP"). Command-line search (`> search_`), A-Z jump rail on the right, Enter launches the first match.
- **Long-press an app:** add to dock, app info, uninstall, hide.
- **Config:** long-press an empty spot on Home, or type `/config` in the drawer search. Accent colour (cyan / red / green / amber), scan-line toggle, icon style (outlined / original), unhide apps, reset dock, jump to the default-home picker.
- Portrait and landscape.

## 1. Build the APK

1. Install **Android Studio** (current stable) on your PC. Open the `EdithLauncher` folder.
2. Let Gradle sync (it downloads Gradle 8.7, AGP 8.5.2, Kotlin 1.9.24; if Studio offers to upgrade them, accepting is fine).
3. **Build > Build Bundle(s) / APK(s) > Build APK(s)** for a debug APK, or in a terminal:
   ```
   gradlew assembleRelease        (Windows)
   ./gradlew assembleRelease      (Linux/macOS)
   ```
   If `gradlew` is missing, run `gradle wrapper --gradle-version 8.7` once, or just build from Studio.
4. Output: `app/build/outputs/apk/release/app-release.apk` (R8-shrunk, signed with the debug key so it installs without a keystore; fine for personal use).

## 2. Install and make it your home

On the tab: **Settings > About tablet**, tap *Build number* 7 times, then **Developer options > USB debugging** ON. Connect to the PC and accept the prompt.

```
adb install -r app/build/outputs/apk/release/app-release.apk
```

Then on the tab: press Home, pick **EDITH**, choose **Always**.
Or: **Settings > Apps > Default apps > Home app > EDITH**.
Or open EDITH > long-press Home > "SET AS DEFAULT HOME APP".

**Safety net:** do NOT uninstall or disable the stock Lenovo launcher. If anything ever misbehaves, switch the Home app back in Settings > Apps > Default apps. Do your debloat only after EDITH is installed and confirmed as default.

## 3. Memory and battery test checklist

Run each before/after and compare:

```
adb shell dumpsys meminfo com.edith.launcher        # look at TOTAL PSS (goal: well under ~60 MB idle)
adb shell dumpsys meminfo | findstr /i launcher     # compare with the stock launcher's PSS
adb shell dumpsys batterystats --reset              # then use the tab for an hour, with screen on at Home
adb shell dumpsys batterystats | findstr /i edith
adb shell top -n 1 | findstr /i edith               # CPU while sitting on Home
```

Checklist:
- [ ] Idle on Home with scan-line OFF: PSS noted, CPU about 0-2 %
- [ ] Idle on Home with scan-line ON: CPU noted (should stay low; turn it off if it bothers you)
- [ ] Open the drawer, scroll the whole list: PSS before/after (icon cache is capped at 4 MB)
- [ ] Rotate portrait/landscape a few times: no crash, grid re-flows
- [ ] Install and uninstall an app: drawer refreshes within about a second
- [ ] Screen off 5 minutes: `top` shows EDITH at 0 % (animations and receivers stop in onPause/onStop)
- [ ] `adb shell dumpsys activity services com.edith.launcher` shows no services

## 4. Five RAM-saving decisions

1. **One Canvas view for the whole home screen.** Clock, rings, brackets, text, scan-line are drawn directly. No Compose, WebView, Lottie, bitmaps, blur or shaders; no view tree to hold in memory. Paints, paths and tick arrays are built once, nothing is allocated in `onDraw`.
2. **Icons: 96 px bitmaps in a 4 MB LruCache, loaded on one thread.** A full list can never grow past 4 MB of icons, and only visible rows ever load (RecyclerView recycling, item animator off).
3. **Event-driven, not polled.** Time comes from `ACTION_TIME_TICK`, battery from the sticky battery broadcast, app list from package add/remove/change broadcasts (debounced). No services, no alarms, no background threads left running.
4. **Everything stops when it is not visible.** The HUD ticker stops in `onPause` and while the drawer covers it; receivers are unregistered in `onStop` and `onDestroy`; animators are cancelled. Scan-line is capped at ~15 fps, falls back to 1 redraw/second, and turns itself off at 20 % battery or less.
5. **Minimal dependencies and plain black UI.** Framework theme (no AppCompat), settings screen built in code, system monospace font (no font file), no `INTERNET` permission, R8 + resource shrinking on. Pure `#000000` background.

## 5. Notes and honest limits

- The greeting ("EDITH ONLINE") shows once per launcher process (first start after boot or after Android kills it), not on every unlock.
- Crash safety: if the process crashed in the last 15 s, EDITH starts in SAFE MODE (no animation loop). Android restarts the default home app after a crash. If something is badly broken, switch Home app back to the stock launcher.
- The font is the system monospace. To use Share Tech Mono / Orbitron, drop a `.ttf` in `res/font/` (keep it under 50 KB) and load it with `resources.getFont()` in `HudView`, the adapter and `SettingsActivity`.
- Disabled apps (`pm disable-user`) disappear from the drawer automatically.
- Hidden apps are hidden by package name; restore them in CONFIG.
- This project was written without an Android SDK available, so it has not been compiled here. If Studio reports a small error on first sync/build, send me the message and I will fix it.
