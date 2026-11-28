# M02 Write-Up — Community Modules and Native Linking

> Fill this in as you go. Write it so a trainee starting this stage next cohort could follow your
> path on their own.

## What I built

The `PersonalProfileApp` skeleton from M01 now carries its first real native dependencies:
`@react-native-async-storage/async-storage` (2.2.0), `react-native-safe-area-context` (~5.11.1) and
`react-native-vector-icons` (^10.3.0), all installed with `npx expo install`. Autolinking wired the
three native modules; two things still had to be done by hand — registering `Ionicons.ttf` in
`ios/PersonalProfileApp/Info.plist` under `UIAppFonts`, and adding the vector-icons `fonts.gradle`
script to `android/app/build.gradle`. After `pod install`, a proof screen in `App.js` rendered an
Ionicons checkmark, wrote and read a value through AsyncStorage, and displayed live safe-area inset
values. Both platforms built and opened with all three working. Alongside the app is a one-page
lock-in audit of every dependency added so far (all eight rows), and `npx expo-doctor` is clean at
20/20.

An EAS login on the free tier was confirmed with `eas whoami`; account details are deliberately not
recorded here or in the teaching material.

## Why it's built this way (key decisions)

- **The hybrid rule.** The three additions are community modules because they do core jobs that
  need native code — persistent key-value storage, real device insets, and an icon font. `expo-status-bar`
  stays an Expo utility because it is a thin wrapper that can be swapped for React Native's own
  `StatusBar` in one line. That is the rule in practice: community modules own core features; small
  Expo utilities are used only where they are easy to replace later.
- **Why the committed native folders still matter.** `eas build` builds both platforms from the
  committed `ios/` and `android/` folders, and `eas submit` uploads the result to App Store
  Connect/TestFlight and the Play Console. Running `expo prebuild` would regenerate those folders
  from config and would overwrite the hand-written font registration this stage depends on. This
  course keeps and hand-manages the committed folders, so prebuild is not used here.
- **AsyncStorage — confident.** Last release 2026-05-29, Directory score 90 with a New Architecture
  badge and 15 open issues, and codegen generated `rnasyncstorage`. `npx expo install` pinned 2.2.0
  even though npm offers 3.1.1; that gap is intentional SDK-58 alignment, not a stale install, so it
  was not hand-upgraded.
- **safe-area-context — confident.** Last release 2026-11-03, Directory score 95, Expo Go yes, 78
  open issues, maintained by AppAndFlow (Janic Duplessis) — `th3rdwave` is only the Android Java
  namespace. The Directory page shows no New Architecture badge, but local `pod install` generated
  `safeareacontext` while the build was configured with the New Architecture, which is the stronger
  evidence here.
- **vector-icons — usable but flagged.** Codegen generated `RNVectorIconsSpec` and both platforms
  built, so it works. However, npm marks `react-native-vector-icons@10.3.0` **deprecated** ("moved to
  a new model of per-icon-family packages"), and its last release is 2025-07-23 — the oldest signal
  of the three. It also resolved with a caret range rather than an Expo-pinned one. The brief names
  this exact package, so migrating it is a program-owner scope decision, not a trainee-only fix; the
  audit records the finding and a concrete exit plan instead of changing the implementation.
- **Where autolinking needed help.** All three modules were discovered and linked automatically —
  `pod install` reported `RNCAsyncStorage`, `RNVectorIcons` and `react-native-safe-area-context`, and
  codegen ran for all three. What autolinking does *not* do is register a font **asset**: a `.ttf` is
  a resource, not a native module. That is why `UIAppFonts` and the Gradle font script had to be
  written by hand. AsyncStorage and safe-area-context needed no manual edits because they ship
  podspecs/Gradle projects and codegen specs that autolinking can consume directly.

## How to build it (teach it to the next trainee)

Worked example: linking `react-native-vector-icons` by hand on both platforms.

1. **Install the SDK-aligned versions.** Do not type versions yourself; let Expo resolve them:

   ```bash
   npx expo install @react-native-async-storage/async-storage react-native-safe-area-context react-native-vector-icons
   ```

   Confirm what landed in `package.json`: `2.2.0`, `~5.11.1` and `^10.3.0` respectively. If a
   version looks older than npm's latest (AsyncStorage 2.2.0 versus 3.1.1), that is expected — Expo
   pins versions that match SDK 58.

2. **Register the icon font on iOS.** Open `ios/PersonalProfileApp/Info.plist` and add:

   ```xml
   <key>UIAppFonts</key>
   <array>
     <string>Ionicons.ttf</string>
   </array>
   ```

   Without this entry the native module links fine but the glyphs render as missing characters.

3. **Register the icon font on Android.** In `android/app/build.gradle`, add this line after the
   plugin declarations and before the `react { ... }` block:

   ```groovy
   apply from: "../../node_modules/react-native-vector-icons/fonts.gradle"
   ```

   The script copies the bundled fonts into the Android asset path at build time.

4. **Refresh native dependencies.**

   ```bash
   cd ios && pod install && cd ..
   ```

   Expect `Auto-linking React Native modules ... RNCAsyncStorage, RNVectorIcons, and
   react-native-safe-area-context`, codegen output for `rnasyncstorage`, `safeareacontext` and
   `RNVectorIconsSpec`, and `Pod installation complete!` (101 pods in this run). Non-fatal noise is
   normal here: RN prints a notice that direct `pod install` is deprecated in favour of
   `npx expo run:ios`, and Expo prints an internal `CoreModule` access-level warning. Do not edit
   `node_modules` to silence either.

5. **Add a proof screen that exercises all three modules.** Wrap the tree in `SafeAreaProvider`,
   read `useSafeAreaInsets()` inside a `SafeAreaView`, write then read a key with AsyncStorage, and
   render an Ionicons `checkmark-circle`:

   ```jsx
   import { useEffect, useState } from 'react';
   import { StyleSheet, Text } from 'react-native';
   import AsyncStorage from '@react-native-async-storage/async-storage';
   import {
     SafeAreaProvider,
     SafeAreaView,
     useSafeAreaInsets,
   } from 'react-native-safe-area-context';
   import Icon from 'react-native-vector-icons/Ionicons';
   import { StatusBar } from 'expo-status-bar';

   function ProfileScreen() {
     const insets = useSafeAreaInsets();
     const [storedValue, setStoredValue] = useState('loading...');

     useEffect(() => {
       async function checkStorage() {
         await AsyncStorage.setItem('m02_check', 'AsyncStorage works');
         const value = await AsyncStorage.getItem('m02_check');
         setStoredValue(value ?? '(no value)');
       }
       checkStorage();
     }, []);

     return (
       <SafeAreaView style={styles.container}>
         <Icon name="checkmark-circle" size={48} color="#2e7d32" />
         <Text style={styles.text}>Icons: Ionicons rendered</Text>
         <Text style={styles.text}>AsyncStorage: {storedValue}</Text>
         <Text style={styles.text}>Safe-area top inset: {insets.top}</Text>
         <StatusBar style="auto" />
       </SafeAreaView>
     );
   }

   export default function App() {
     return (
       <SafeAreaProvider>
         <ProfileScreen />
       </SafeAreaProvider>
     );
   }

   const styles = StyleSheet.create({
     container: {
       flex: 1,
       backgroundColor: '#fff',
       alignItems: 'center',
       justifyContent: 'center',
       gap: 16,
     },
     text: { fontSize: 18 },
   });
   ```

6. **Rebuild both platforms — fast refresh is not enough.** Native files changed, so run:

   ```bash
   npx expo run:ios
   npx expo run:android
   ```

   If port 8081 is already in use, answering "no" to the alternate-port prompt makes Expo skip the
   dev server, but the native build, install and launch still succeed.

7. **Check the screen, not just the build log.** A green check icon proves the font registration,
   the `AsyncStorage: AsyncStorage works` line proves the native module is linked, and a non-zero
   top inset proves safe-area-context is live. This run showed top inset 62 on the iOS simulator and
   24 on the Android emulator; exact numbers vary by device.

8. **Run the health check last.**

   ```bash
   npx expo-doctor
   ```

   A clean result is `20/20 checks passed. No issues detected!`.

9. **Write the lock-in audit while the evidence is fresh.** One row per dependency (all eight here),
   with columns for exact version, purpose, maintainer, health signals, lock-in rating, failure mode
   and exit plan. Pull last-release dates with `npm view <package> time.modified`, and record
   Directory standing from `https://reactnative.directory/package/<name>`.

## Concepts worth explaining

**1. Autolinking versus manual linking**

Autolinking is the build tooling's discovery step. On iOS, CocoaPods reads the podspecs that
packages ship and `link_native_modules!` adds them to the Xcode project; on Android, Gradle's
`autolinkLibrariesWithApp()` does the equivalent. Native modules that declare codegen specs get
native interface code generated at this stage — in this run, `rnasyncstorage`, `safeareacontext`
and `RNVectorIconsSpec` each produced generated files under `ios/build/generated/ios/ReactCodegen`.
That is why adding a native module usually needs no hand-written bridge registration.

Manual linking is for what autolinking cannot describe. A font file is an asset, not a native
module with a spec, so nothing tells iOS to copy `Ionicons.ttf` or Android to include it. Those two
settings live in the app's own native project (`UIAppFonts` in `Info.plist`, a Gradle script in
`build.gradle`), outside any package. The practical rule: if a change belongs to app identity,
assets or native project configuration, expect to make it by hand and rebuild. If it belongs to a
package's own native project, let autolinking find it — but verify the generated output rather than
assuming. Because native files changed, a rebuild with `npx expo run:ios` / `npx expo run:android`
was required; editing JS alone would not have applied the font registration.

**2. Judging whether a dependency is safe to take on**

Four checks give an objective picture before you commit: how recently it was released
(`npm view <package> time.modified`), whether it supports the New Architecture (Directory badge
*and* local codegen output), whether its resolved version fits React Native 0.88 (`npx expo install`
plus a successful build), and its standing on React Native Directory (score, open issues, archived
or not). Two more signals caught things the four checks alone would miss: npm's deprecation notice
on `react-native-vector-icons`, and the caret range it resolved to instead of an Expo-pinned range.

The point of the audit is not to reject every imperfect package. It is to ask "what breaks if this
disappears tomorrow?" and write a specific exit plan — name the replacement and the steps to get
there. A deprecated package that is explicitly named in the brief can still be used; it just needs
the risk recorded and escalated as a scope decision rather than silently accepted.

## Checkpoint evidence

- **iOS:** `npx expo run:ios` → `Build Succeeded`, `0 error(s), and 1 warning(s)`; installed and
  opened on iPhone 18 Pro. Screen showed the green Ionicons checkmark, `Icons: Ionicons rendered`,
  `AsyncStorage: AsyncStorage works`, and `Safe-area top inset: 62`.
- **Android:** `npx expo run:android` → `BUILD SUCCESSFUL in 2m 11s` (239 actionable tasks: 84
  executed, 23 from cache, 132 up-to-date); installed `app-debug.apk` and opened
  `com.personalprofileapp/.MainActivity` on `Medium_Phone_API_37.0`. Screen showed the same three
  proofs with `Safe-area top inset: 24`. Gradle emitted deprecation warnings and node_modules
  manifest `package=` notices; none failed the build.
- **Linking:** `pod install` reported autolinking of `RNCAsyncStorage`, `RNVectorIcons` and
  `react-native-safe-area-context`, generated codegen for all three, and finished with 101 pods
  installed. Manual font registration is present in both `Info.plist` and `android/app/build.gradle`.
- **Audit:** `docs/lock-in-audit.md` has all eight dependency rows with version, purpose, maintainer,
  health signals, lock-in rating, failure mode and exit plan. Open items are stated explicitly:
  no trainer example row was available, so the format was created by the trainee, and there is no
  instructor to sign off the brief's review checkbox. The vector-icons deprecation is recorded as a
  finding with a migration path.
- **Health check:** `npx expo-doctor` → `20/20 checks passed. No issues detected!`.

## What I'd do differently

- Check Directory standing, npm deprecation notices and the resolved version range **before**
  installing, not during the audit afterwards. The vector-icons deprecation would have surfaced at
  the decision point instead of at the review point.
- Ask for the trainer's example audit row at the start of the stage, then match it before writing
  the first row. Without one, define the columns explicitly and flag the format as an open item.
- Treat non-fatal warnings as a named group in the notes (RN `pod install` deprecation, Expo
  `CoreModule` warning, Gradle/Kotlin deprecations, manifest `package=` notices). Sorting them once
  avoids re-investigating each one later.
- Re-check version pinning before any SDK upgrade, especially the caret-ranged vector-icons
  package, and use `npx expo install` for the upgrade rather than editing `package.json` by hand.
