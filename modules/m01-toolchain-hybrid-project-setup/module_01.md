# M01 — Toolchain and Hybrid Project Setup

This module sets up a machine that can build a bare React Native project on Expo tooling for both
iOS and Android. The outcome is a `PersonalProfileApp` skeleton, a README that takes a fresh clone to
a running app on both platforms, and an explanation of what happens between starting Metro and
seeing pixels on a device.

The setup is intentionally reproducible: pin the tool versions, keep the native folders in Git, and
use Expo's tooling to build. Expo Go is enough for the JS-only screens in M01; later modules move to
dev builds once native modules are linked, while EAS Build remains the build service for the rest of
the course.

## 1. Verify the toolchain before creating the app

Installing a tool is not the same as making it usable. A tool can be installed at the wrong major
version, or its executable can be missing from `PATH`. Verify every tool with a version command
before moving on.

| Tool | What to verify | Notes |
| --- | --- | --- |
| Xcode | `xcodebuild -version` | This cohort confirmed Xcode 27.2 (27B52), which satisfies the 26.4+ requirement. If the active developer directory is Command Line Tools, switch with `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` and accept the license with `sudo xcodebuild -license accept`. |
| Android Studio + SDK | SDK Platform 37, Platform-Tools, Build-Tools, Emulator; `adb --version` | Set `ANDROID_HOME="$HOME/Library/Android/sdk"` and add `$ANDROID_HOME/platform-tools` to `PATH`. |
| JDK | `java -version` | Android Studio's bundled JBR can be used. The exact JDK version is still open; confirm it with the trainer. |
| Node.js | `node -v` | Use the pinned Node 24 LTS line. If the shell is on an older major, switch with `nvm install 24` and `nvm use 24`. |
| Watchman | `watchman --version` | Needed by Metro's file watching. |
| Ruby / Bundler / CocoaPods | `ruby -v`, `bundle -v`, `pod --version` | macOS system Ruby is too old for CocoaPods. Use Homebrew `ruby@3.4`, add its `bin` directory and the gem executable directory reported by `gem environment` to `PATH`, then install Bundler and CocoaPods without `sudo`. |
| Git | `git --version` | The project needs to be cloned, committed, and pushed. |
| Expo/EAS CLI | `eas whoami` | Install with `npm install -g eas-cli`, then log in with a free Expo account. |

### Shell profile setup

Many of these tools are found through `PATH`. Add these exports to `~/.zshrc` once, then open a new
terminal (or run `source ~/.zshrc`) before verifying tools:

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

The last command, `npx expo-doctor`, is the overall health check. `20/20 checks passed. No issues
detected!` is the clean result. When adding SDK-aligned libraries later, install them with
`npx expo install` so versions match Expo SDK 58 instead of guessing.

## 2. Create the project

Use the bare-minimum starter on Expo tooling:

```bash
npx create-expo-app PersonalProfileApp --template bare-minimum
```

Confirm the SDK that was actually installed:

```bash
grep '"expo"' package.json
```

For this cohort the result is `"expo": "~58.1.0"`, which matches Expo SDK 58. A bare project keeps
`ios/` and `android/` in the repository, so the native folders are part of the project from the
start.

## 3. Learn the project layout

| File / folder | What it owns | Why it matters |
| --- | --- | --- |
| `ios/` | Xcode project and workspace, `Podfile`, `AppDelegate.swift`, `Info.plist`, assets, privacy manifests | Xcode and CocoaPods use it to build and install the iOS `.app`. It is committed so a fresh clone and EAS Build can produce the iOS binary. |
| `android/` | Gradle scripts, `AndroidManifest.xml`, `MainActivity.kt`, `MainApplication.kt`, resources | Gradle uses it to build `app-debug.apk`. It is committed for the same fresh-clone/EAS reason. |
| `app.json` | Static Expo app config: `name`, `slug`, `version` | Expo CLI and EAS use it as app identity. It contains no native code. |
| `metro.config.js` | Metro resolution and bundling behavior | This project uses Expo's default config unchanged: `getDefaultConfig(__dirname)` is exported. Customization is only needed for custom resolvers, transformers, or asset extensions. |
| `babel.config.js` | Babel presets and plugins | Expo SDK 58 configures Babel automatically with `babel-preset-expo`, so this file is not required in the starter. If customization is needed, create it with `npx expo customize babel.config.js`. |

The default Babel config generated by Expo is:

```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
  };
};
```

Babel compiles modern JS/JSX/TS syntax into code the JavaScript engine can run. Metro then bundles
and serves the compiled modules. After changing Babel config, restart Metro with a clean cache:

```bash
npx expo start --clear
```

The original absence of `babel.config.js` is expected, not a missing project file.

## 4. Understand the path from Metro to pixels

`npx expo start` does not start Xcode or Gradle. It starts Metro, a JavaScript bundler and dev
server on the Mac. Metro packs the JavaScript/TypeScript into a bundle and keeps serving it at
`localhost:8081`. The QR code and `exp://` link are another way for a device to reach that same
bundle.

The pixels do not come from Metro. They come from the native app hosting the bundle. With Expo Go,
the phone borrows Expo's already-built native shell, which knows how to create iOS/Android views and
exposes the native modules included in the SDK. The phone downloads the JS bundle over the same
Wi-Fi; Hermes runs that bundle inside the shell; React describes the UI; and the native side turns
that description into actual views.

That is why saving a JS file updates instantly, but changing `MainActivity.kt` or an iOS native file
requires a rebuild. The build tools create the host; Metro feeds it; Hermes drives it.

## 5. Run the app

### Expo account and EAS CLI

Install EAS CLI globally:

```bash
npm install -g eas-cli
```

If there is no Expo account yet, create one at `expo.dev/signup`, verify the email, and accept any
trainer invitation or workspace access. Then log in and confirm the active account:

```bash
eas login
eas whoami
```

The free tier is enough for M01.

Start Metro for a JS-only preview:

```bash
npx expo start
```

Install Expo Go on a physical phone, scan the QR code with the iPhone Camera app or Expo Go's
scanner, and keep the phone on the same Wi-Fi as the computer. Expo Go already contains the native
modules shipped with SDK 58, so it can run the M01 skeleton, but it cannot load custom native code.
From M02, once a community native module is linked, the path becomes a dev build using
`expo-dev-client`, built with `expo run:*` or EAS Build.

### Android emulator and device setup

In Android Studio, open SDK Manager and install SDK Platform 37, Android SDK Build-Tools, Android
SDK Platform-Tools, and Android Emulator. Then open Device Manager → Create Virtual Device → Phone →
Medium Phone → API 37 system image → Finish, and start `Medium_Phone_API_37.0`. Confirm the emulator
is visible before building:

```bash
adb devices
```

For optional physical-device testing, enable Developer Options and USB debugging on the Android
phone, connect it by USB, approve the computer/debug prompt on the phone, then confirm it appears in
`adb devices`. A physical device is extra for M01; the checkpoint still needs the emulator.

Build and open the native apps:

```bash
npx expo run:ios
npx expo run:android
```

The iOS run builds with Xcode/CocoaPods and installs on an iOS simulator such as iPhone 18 Pro. The
Android run builds with Gradle and installs `app-debug.apk` on an emulator such as
`Medium_Phone_API_37.0`. A connected physical device can be used for day-to-day testing, but the M01
check still asks for both simulator and emulator, with a real device as the extra.

A clean result looks like:

- `npx expo-doctor` — `20/20 checks passed. No issues detected!`
- `npx expo run:ios` — `Build Succeeded`, `0 error(s), and 1 warning(s)`, opened on the simulator.
- `npx expo run:android` — `BUILD SUCCESSFUL`, installed `app-debug.apk`, opened
  `com.personalprofileapp/.MainActivity` on the emulator.

Non-fatal Kotlin/Gradle deprecation warnings are expected in this toolchain. If port 8081 is busy,
stop the existing Metro server or accept another port. If a project folder has been moved, stale
absolute paths can remain in generated build folders; remove generated folders and `node_modules`,
then reinstall before rebuilding.

## 6. Prove the README works from a fresh clone

The deliverable is not just “it runs on my machine.” Clone the repo into a new folder and follow
only `README.md`:

```bash
git clone <your-repo-url> PersonalProfileApp
cd PersonalProfileApp
npm ci
cd ios && pod install && cd ..
npx expo-doctor
npx expo start
npx expo run:ios
npx expo run:android
```

For this project, the fresh-clone run produced 446 packages from `npm ci`, 98 pods from
`pod install`, a clean `expo-doctor` result, an iOS build that opened on iPhone 18 Pro, and an
Android build that opened `com.personalprofileapp/.MainActivity` on `Medium_Phone_API_37.0`.

## 7. Why the native folders stay committed

“Bare” means `ios/` and `android/` belong to the project, not hidden inside Expo. They hold app
identity, permissions, and native dependencies — the things a cloud build and a fresh clone both
need. Without them, `npx expo run:ios` could not produce the `.app`, and EAS Build would have nothing
to build from.

Expo-first tooling is the layer on top. Expo Go is enough while screens are JS-only; once a native
module is linked in M02, Expo Go will not contain it and a dev build is needed. EAS Build is the
build tool used through the capstone, including release candidates, so trainees do not have to
hand-sign in Xcode or Gradle. A manual Xcode/Gradle path is optional depth later, not the required
M01 path.

## 8. Constraints and open items

Keep these stack constraints in view:

- React Native 0.88, Expo SDK 58, React 19.3.0.
- Node 24 LTS and New Architecture always on.
- Xcode 26.4 or newer; this cohort confirmed 27.2 (27B52).
- Android `compileSdk` 37 and `targetSdk` 36.

Some items are explicitly open for the program owner or trainer: the exact JDK version, store
accounts, the real-device pool, exact point versions, and the course's TypeScript-from-M01
expectation versus the JS-only starter observed here. Do not silently settle them in module docs;
flag them for confirmation.

## Comprehension questions

1. What shell-profile `PATH` and version checks would you run before building to know that Node, Ruby/CocoaPods, and the Android SDK are actually usable, and why are those checks different from installing the tools?
2. How would you decide which project file owns a change involving app identity, Metro resolution, JavaScript transformation, iOS native setup, or Android native setup?
3. Trace the path from `npx expo start` to pixels on a screen, describing the roles of Metro, Hermes, React, and the native shell.
4. Why does a bare Expo project keep `ios/` and `android/` in Git while still using Expo Go, dev builds, and EAS Build for building?
5. How would you verify that a stranger can clone the repo and reach a running app on both platforms using only the README, and what would you do if stale generated paths or a busy port interfered?

## Understanding Checklist

- [ ] I can explain why shell-profile `PATH` setup and version checks differ from installation, name the commands I would use for Node, CocoaPods, and `expo-doctor`, and confirm an Android emulator or device with `adb devices`.
- [ ] I can assign a change to `app.json`, `metro.config.js`, `babel.config.js`, `ios/`, or `android/` and explain what each owns.
- [ ] I can describe what Metro, Hermes, React, and the native shell each contribute between the QR code and visible UI.
- [ ] I can explain why native folders are committed and where Expo Go, dev builds, and EAS Build fit in the course.
- [ ] I can run a fresh-clone check from the README and respond to stale paths or a busy Metro port without inventing extra instructions.
