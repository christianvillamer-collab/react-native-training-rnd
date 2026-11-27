# M01 Write-Up — Toolchain and Hybrid Project Setup

> Fill this in as you go. Write it so a trainee starting this stage next cohort could follow your
> path on their own.

## What I built

`PersonalProfileApp`'s skeleton — the toolchain setup, the chosen starter path, and the README that
gets a stranger to a running app.

## Why it's built this way (key decisions)

- Starter path: used `npx create-expo-app PersonalProfileApp --template bare-minimum` (confirmed by
  me). What I checked: after creation, `grep '"expo"' package.json` → `"expo": "~58.1.0"`, so the
  template did install Expo SDK 58 on this machine. Xcode is confirmed as 27.2 (27B52), replacing
  the earlier 26.4 assumption.
- **Fault 1 — wrong Node version**
  - What was planted: Node was on an older v22 instead of the pinned Node 24 LTS.
  - Exact symptom / error output: no visible error at the time — I installed `eas-cli`, logged in to
    Expo and created the project without a failure message. The mismatch was caught by checking the
    version, not by a crash.
  - Diagnostic path: `node -v` → `v22.x` versus the pinned Node 24; `npx expo-doctor` reported
    `20/20 checks passed` after the fix (no pre-fix warning was noted).
  - Fix: `nvm install 24` then `nvm use 24` to switch to the pinned Node 24.
  - Verification after fix: `node -v` → `v24.22.3`; `npx expo-doctor` → `20/20 checks passed.
    No issues detected!`.
- **Fault 2 — missing Pod install / CocoaPods not on `PATH`**
  - What was planted: a missing Pod install in `ios/`. Before I could run it, verifying the tool
    showed the same CocoaPods setup problem: `pod --version` reported that `pod` was not a command in
    the terminal. The root cause was that macOS system Ruby 2.6.10 was too old for CocoaPods and the
    gem executable directory was not on `PATH`; the full diagnosis is under "What tripped me up".
  - Exact symptom / error output: no iOS build error before the fix. The visible symptom was from
    `pod --version` → the terminal said `pod` was not a command.
  - Diagnostic path: `pod --version` → command not found; `ruby -v` → system Ruby 2.6.10;
    `gem environment` → gem executable directory; after adding that directory to `PATH`,
    `cd ios && pod install` could run.
  - Fix: installed Homebrew `ruby@3.4`, added `/opt/homebrew/opt/ruby@3.4/bin` and the gem exec dir
    `/opt/homebrew/lib/ruby/gems/3.4.0/bin` to `PATH`, installed `bundler` and `cocoapods`, then ran
    `cd ios && pod install`. Expo ran codegen, integrated dependencies, and printed a deprecation
    notice that direct `pod install` is being replaced by `npx expo run:ios`, plus a non-fatal
    `CoreModule` access-level warning.
  - Verification: `pod --version` → `1.17.0`; `Pod installation complete! There are 99 dependencies
    from the Podfile and 98 total pods installed.` (14 s on the first run, 32 s on the clean rebuild
    from the new path). Re-ran `npx expo run:ios` after the clean rebuild → `Build Succeeded`,
    `0 error(s), and 1 warning(s)`, installed and opened on iPhone 18 Pro.
- **Fault 3 — bad `ANDROID_HOME`**
  - What was planted: a bad `ANDROID_HOME` value. What I actually hit: Android Studio and the SDK
    were installed, but `adb` was not available in the terminal.
  - Exact symptom / error output: `adb: command not found` after installing Android Studio and the
    SDK.
  - Diagnostic path: `adb --version` → command not found; `echo $ANDROID_HOME` → initially not
    pointing at `/Users/ve250412/Library/Android/sdk`; real SDK path is
    `/Users/ve250412/Library/Android/sdk`.
  - Fix: set `ANDROID_HOME="$HOME/Library/Android/sdk"`, add `$ANDROID_HOME/platform-tools` to
    `PATH`, then `source ~/.zshrc`.
  - Verification after fix: `echo $ANDROID_HOME` → `/Users/ve250412/Library/Android/sdk`;
    `adb --version` → `1.0.41 (37.0.3-16887417)` at
    `/Users/ve250412/Library/Android/sdk/platform-tools/adb`; `adb devices` started the daemon.
    `npx expo run:android` had already built successfully with this setup.

## How to build it (teach it to the next trainee)

Setup guide — install in this order and verify each tool before moving on:

Shell profile setup — add these exports to `~/.zshrc` once, then run `source ~/.zshrc` in a new
terminal before verifying tools:

```bash
# Homebrew Ruby for CocoaPods
export PATH="/opt/homebrew/opt/ruby@3.4/bin:$PATH"
export PATH="/opt/homebrew/lib/ruby/gems/3.4.0/bin:$PATH"

# Android SDK
export ANDROID_HOME="$HOME/Library/Android/sdk"
export PATH="$ANDROID_HOME/platform-tools:$PATH"
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"

# JDK bundled with Android Studio
export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
```

Verify in a new terminal:

```bash
node -v
watchman --version
ruby -v && bundle -v && pod --version
java -version
adb --version
xcodebuild -version
npx expo-doctor
```

1. Xcode — installed; verified with `xcodebuild -version` → `Xcode 27.2 (27B52)`. If `xcodebuild`
   points at Command Line Tools, switch with
   `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` and accept the license with
   `sudo xcodebuild -license accept`. User confirmed 27.2 for this cohort; it replaces the earlier
   26.4 assumption.
2. Android Studio + SDK — installed Android Studio, then opened SDK Manager and installed SDK
   Platform 37, Android SDK Build-Tools, Android SDK Platform-Tools and Android Emulator. Set
   `ANDROID_HOME="$HOME/Library/Android/sdk"` and added `$ANDROID_HOME/platform-tools` to `PATH`.
   Verified `adb --version` → `Android Debug Bridge 1.0.41 (37.0.3-16887417)` at
   `/Users/ve250412/Library/Android/sdk/platform-tools/adb`; `adb devices` started the daemon and
   listed no devices yet.

   Emulator setup for the run: opened Device Manager → Create Virtual Device → Phone → Medium
   Phone → API 37 system image → Finish, then started `Medium_Phone_API_37.0`. `adb devices`
   confirmed the emulator was visible before `npx expo run:android`.

   Optional physical device: enable Developer Options and USB debugging on the Android phone,
   connect it by USB, approve the computer/debug prompt on the phone, then confirm it appears in
   `adb devices`. A physical device is extra for M01; the checkpoint still needs the emulator.
3. JDK — using Android Studio's bundled JBR at
   `/Applications/Android Studio.app/Contents/jbr/Contents/Home`; set `JAVA_HOME` and added
   `$JAVA_HOME/bin` to `PATH`. Verified `java -version` → `OpenJDK 25.0.4.1`. The brief pins no JDK
   version — flagged for trainer confirmation. Gradle 9.4.1 started under this JDK without a Java
   version error.
4. Node.js — installed at the pinned major version; verified with `node -v` → `v24.22.3`.
5. Watchman — installed; verified with `watchman --version` → `2026.11.23.00`.
6. Ruby / Bundler / CocoaPods — the system Ruby was 2.6.10, too old for CocoaPods. Installed
   Homebrew `ruby@3.4` (3.4.13), added `/opt/homebrew/opt/ruby@3.4/bin` and the gem executable dir
   `/opt/homebrew/lib/ruby/gems/3.4.0/bin` to `PATH`, then installed gems without `sudo`:
   `gem install bundler` → 4.1.1, `gem install cocoapods` → 1.17.0. Verified with `ruby -v` →
   3.4.13, `bundle -v` → 4.1.1, `pod --version` → 1.17.0.
7. Git — installed; verified with `git --version` → `git version 2.54.0 (Apple Git-157)`.
8. Project — created `PersonalProfileApp` with `npx create-expo-app --template bare-minimum`. Verified
   the installed SDK in `package.json`: `"expo": "~58.1.0"`. Ran `npx expo-doctor` (which installed
   `expo-doctor@1.21.9` on first use) → `20/20 checks passed. No issues detected!`.
9. Expo/EAS CLI — installed globally with `npm install -g eas-cli`. If there is no Expo account
   yet, created one at `expo.dev/signup`, verified the email, and accepted any trainer invitation
   or workspace access. Then logged in with:

   ```bash
   eas login
   eas whoami
   ```

   Verified `eas whoami` → the active Expo username. The free tier is enough for M01. Personal
   account details are not included in the teaching material.
10. `npx expo start` — starts Metro, the JavaScript bundler and dev server. The terminal prints a
    local URL (`http://localhost:8081`) and an `exp://` link with a QR code. On a physical phone,
    install **Expo Go** (free, from the App Store or Google Play), then scan the QR — with the iPhone
    Camera app or the scanner inside Expo Go on Android. Expo Go fetches the JS bundle from Metro
    over the same Wi-Fi and renders it inside Expo Go's own native shell; editing and saving JS/TS
    hot-reloads the app. This is a JS-only preview: Expo Go already contains the native modules that
    ship with SDK 58, so it runs the M01 skeleton, but it cannot load custom native code. From M02,
    once a community native module is linked, the path becomes a dev build (`expo-dev-client`, built
    with `expo run:*` or EAS Build), while EAS Build remains the build tool for the course.
11. Runs — `npx expo run:ios` → `Build Succeeded`, `0 error(s), and 1 warning(s)`, installed and
    opened on iPhone 18 Pro simulator (`org.name.PersonalProfileApp`). `npx expo run:android` →
    `BUILD SUCCESSFUL in 6m 26s` (176 actionable tasks: 156 executed, 20 from cache), installed
    `app-debug.apk` and opened `com.personalprofileapp/.MainActivity` on
    `Medium_Phone_API_37.0`; Gradle 9.4.1 emitted a Kotlin deprecation warning about
    `DefaultReactActivityDelegate` and a Gradle 10 compatibility warning. A connected physical device
    can be targeted instead of a simulator/emulator for day-to-day testing, but the M01 checkpoint
    still calls for simulator **and** emulator, with a real device as the extra.
12. Clean rebuild after moving the project outside the curriculum repo — stopped Metro on port 8081,
    removed `ios/Pods`, `ios/build`, `android/.gradle`, `android/app/build`, `android/build`, `.expo`
    and `node_modules`, then followed the README. `npm ci` → `added 446 packages` (3 moderate
    vulnerabilities reported); `cd ios && pod install` → 98 pods in 32 s; `npx expo-doctor` → 20/20;
    `npx expo run:ios` → `Build Succeeded`, `0 error(s), and 1 warning(s)` on iPhone 18 Pro;
    `npx expo run:android` → `BUILD SUCCESSFUL in 34s` (176 tasks: 90 executed, 86 from cache),
    installed `app-debug.apk` and opened `Medium_Phone_API_37.0`. Gradle also printed stale
    "Failed to open manifest file" lines pointing at the old `training/` path, but the build still
    succeeded.
13. Fresh clone test — cloned the GitHub repo into
    `/Users/ve250412/Desktop/React-native-training/PersonalProfileApp-fresh` and followed the README:
    `npm ci` → 446 packages in 17 s (3 moderate vulnerabilities); `cd ios && pod install` → 98 pods
    in 82 s; `npx expo start` → Metro QR on `exp://172.16.211.232:8081`; `npx expo run:ios` →
    `Build Succeeded`, `0 error(s), and 1 warning(s)`, opened on iPhone 18 Pro; `npx expo run:android`
    → `BUILD SUCCESSFUL in 2m 31s`, opened `com.personalprofileapp/.MainActivity` on
    `Medium_Phone_API_37.0`. `npx expo-doctor` in this clone gave the same `20/20 checks passed.
    No issues detected!` result. Android reported port 8081 already in use, the prompt answered "no"
    to 8082, and Expo skipped the dev server but still installed and opened the app. The stale
    "Failed to open manifest file" lines pointing at the old `training/` path appeared again but were
    non-fatal.

Project layout research — I checked the project root and native folders:

- `ios/` — the native iOS project: `PersonalProfileApp.xcodeproj`,
  `PersonalProfileApp.xcworkspace`, `Podfile`, `AppDelegate.swift`, `Info.plist`, assets and
  privacy manifests. Xcode/CocoaPods use it to build and install the `.app`; it is committed so a
  fresh clone and EAS Build can produce the iOS binary.
- `android/` — the native Android project: Gradle scripts, `AndroidManifest.xml`,
  `MainActivity.kt`, `MainApplication.kt` and resources. Gradle uses it to build `app-debug.apk`;
  it is committed for the same fresh-clone/EAS reason.
- `app.json` — Expo's static app config. In this project it only contains app `name`, `slug` and
  `version`, which Expo CLI/EAS use as app identity; it contains no native code.
- `metro.config.js` — currently calls `getDefaultConfig(__dirname)` and exports it unchanged. Metro
  uses it to resolve JS/TS modules and assets and serve the bundle; customization would be needed
  only for custom resolvers, transformers or asset extensions.
- `babel.config.js` — not present in the starter initially. Expo SDK 58 configures Babel
  automatically with `babel-preset-expo`; a file is only needed for custom presets/plugins. I created
  it for practice with `npx expo customize babel.config.js`, and it worked. It generated the default:
  `module.exports = function (api) { api.cache(true); return { presets: ['babel-preset-expo'] }; };`.
  After changing it, restart Metro with `npx expo start --clear`. Babel compiles modern JS/JSX/TS
  syntax; Metro bundles and serves the compiled modules. The original absence of the file was
  expected, not a missing project file.

## Concepts worth explaining

**1. The path from `npx expo start` to what I saw on screen**

When I ran `npx expo start`, I wasn't starting Xcode or Gradle — I was starting Metro, a bundler
running on my Mac. Its job is to take my JavaScript/TS, pack it into one bundle, and keep serving it
at `localhost:8081`. The QR code and `exp://` link are just another way for a device to reach that
same bundle.

The screen pixels do not come from Metro. They come from the native app hosting the bundle. With
Expo Go, I was borrowing Expo's already-built native shell; it already knows how to make iOS/Android
views and exposes the SDK's native modules. On my phone it downloaded the bundle over the same Wi-Fi,
then **Hermes** ran that bundle inside the shell and React described the UI, which the native side
turned into actual views.

That's why saving a JS file updates instantly, but changing something like `MainActivity.kt` needs a
rebuild — `expo run:ios` gave me an app on the iPhone 18 Pro simulator, and `expo run:android` built
with Gradle and opened `Medium_Phone_API_37.0`. The build tools create the host; Metro feeds it;
Hermes drives it.

**2. Why `ios/` and `android/` are committed even though Expo builds for us**

"Bare" means those folders are ours, not hidden inside Expo. They hold app identity, permissions and
native dependencies — the things a cloud build and a fresh clone both need. Without them,
`npx expo run:ios` could not have produced my `.app`, and EAS Build would have nothing to build from.

Expo-first tooling is the layer on top: Expo Go was enough for M01 because my screens were JS-only,
but from M02, once a native module is linked, Expo Go won't contain it and I'll need a dev build.
EAS Build is the build tool I'll keep using through the capstone, so I don't have to hand-sign in
Xcode/Gradle. Committing the native folders is what makes the README's "fresh clone → running app"
check possible.

## What tripped me up

- `xcodebuild -version` failed with "tool 'xcodebuild' requires Xcode, but active developer directory
  '/Library/Developer/CommandLineTools' is a command line tools instance". Diagnosis: `xcode-select -p`
  showed Command Line Tools, and `ls -d /Applications/Xcode*.app` found no Xcode — only CLT was
  installed. Fix: installed Xcode, ran
  `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`, accepted the license, then
  verified `xcodebuild -version` → `Xcode 27.2 (27B52)`.
- `gem install cocoapods` failed on macOS system Ruby 2.6.10 with "ffi requires Ruby version >= 3.0,
  < 4.1.dev"; pinning `ffi -v 1.17.4` failed the same way. Diagnosis: `ruby -v` confirmed 2.6.10.
  Fix: `brew install ruby@3.4`, put `/opt/homebrew/opt/ruby@3.4/bin` and the gem exec dir
  `/opt/homebrew/lib/ruby/gems/3.4.0/bin` (from `gem environment`) on `PATH`, then reinstalled
  `bundler` and `cocoapods` without `sudo`. `pod` was still missing until the gem exec dir was added.
  Verified `pod --version` → `1.17.0`.

## Checkpoint evidence

- `npx expo-doctor` → `20/20 checks passed. No issues detected!` (after installing
  `expo-doctor@1.21.9`).
- `npx expo run:ios` → `Build Succeeded`, `0 error(s), and 1 warning(s)`, installed and opened on
  iPhone 18 Pro simulator.
- `npx expo run:android` → `BUILD SUCCESSFUL in 6m 26s`, installed `app-debug.apk`, opened
  `com.personalprofileapp/.MainActivity` on `Medium_Phone_API_37.0`; Kotlin/Gradle deprecation
  warnings only.
- Clean rebuild from the new location — `/Users/ve250412/Desktop/React-native-training/PersonalProfileApp`
  (outside the curriculum repo):
  - Steps followed from `README.md`: stop Metro on 8081; remove generated build dirs and
    `node_modules`; `npm ci`; `cd ios && pod install`; `npx expo-doctor`; `npx expo run:ios`;
    `npx expo run:android`.
  - iOS result: `Build Succeeded`, `0 error(s), and 1 warning(s)`, opened on iPhone 18 Pro.
  - Android result: `BUILD SUCCESSFUL in 34s`, opened `com.personalprofileapp/.MainActivity` on
    `Medium_Phone_API_37.0`.
  - Confirmed: built and ran on both platforms from a clean state using only the README steps.
- True fresh clone test — repo URL:
  `https://github.com/christianvillamer-collab/PersonalProfileApp-react-native.git`; clone folder:
  `/Users/ve250412/Desktop/React-native-training/PersonalProfileApp-fresh`.
  - Steps followed from `README.md`: `npm ci`; `cd ios && pod install`; `npx expo-doctor` →
    `20/20 checks passed. No issues detected!`; `npx expo start`; `npx expo run:ios`;
    `npx expo run:android`.
  - iOS result: `Build Succeeded`, `0 error(s), and 1 warning(s)`, installed and opened on
    iPhone 18 Pro (`org.name.PersonalProfileApp`).
  - Android result: `BUILD SUCCESSFUL in 2m 31s` (176 tasks: 90 executed, 86 from cache), installed
    `app-debug.apk` and opened `com.personalprofileapp/.MainActivity` on `Medium_Phone_API_37.0`.
    Port 8081 was busy; answering "no" to port 8082 made Expo skip the dev server, but the native
    build/install still succeeded.
  - Confirmed: a fresh clone followed the README and ran on both simulator and emulator. The stale
    old-path manifest warnings were non-fatal.
- Three planted faults — see "Why it's built this way" for each symptom, diagnostic path, fix and
  verification.

## What I'd do differently

- Check and pin the Node version before installing anything else.
- Update macOS first, then install the full Xcode app — Command Line Tools alone is not enough.
- Install the needed libraries with commands, set up the PATH for each tool, `source` the shell file,
  and verify with the version command after each fix.
