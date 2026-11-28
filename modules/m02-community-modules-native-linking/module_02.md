# M02 — Community Modules and Native Linking

This module adds the first real dependencies to the `PersonalProfileApp` skeleton: local storage,
device-safe layout, and icons. The work is not just installing packages. It is deciding what kind of
package to take on, checking whether it is safe to depend on, and doing the native wiring that
autolinking cannot do for you.

The app ends the stage with all three modules visibly working on an iOS simulator and an Android
emulator, plus a one-page lock-in audit of every dependency added so far. The native folders stay
committed and hand-managed: `eas build` builds both platforms from them, and `eas submit` uploads to
App Store Connect/TestFlight and the Play Console. EAS Update is out of scope.

## 1. Decide what kind of module to take on

The hybrid rule is the starting point. Use community modules for core features — jobs that need
real native behaviour and would be expensive to reimplement. Keep small Expo utilities only where
they are easy to replace later.

| Need in this stage | Choice | Reason |
| --- | --- | --- |
| Persist a value across app launches | `@react-native-async-storage/async-storage` | Core feature; needs a native storage implementation. |
| Keep content clear of notches and the status bar | `react-native-safe-area-context` | Core feature; needs real device insets. |
| Render an icon | `react-native-vector-icons` | Core UI need; supplied as a bundled font. |
| Status-bar styling | `expo-status-bar` | Small Expo utility, swappable for React Native's own `StatusBar` in one line. |

The committed native folders are part of this decision. Running `expo prebuild` regenerates `ios/`
and `android/` from configuration, which would overwrite the hand-written font registration added
in this stage. This course keeps and hand-manages those folders instead, so `npx expo run:ios` and
`npx expo run:android` build from what is committed. An EAS account on the free tier is enough for
the builds from here on.

## 2. Judge a dependency before taking it on

Four checks give an objective picture. Record the answers before installing, not after.

| Check | Where to get it | What good evidence looks like |
| --- | --- | --- |
| Last release | `npm view <package> time.modified` | A recent date relative to the pinned SDK. |
| New Architecture support | React Native Directory badge and local `pod install` output | A badge on the page, plus codegen lines such as `rnasyncstorage` and `Configuring the target with the New Architecture`. |
| Fit with React Native 0.88 | `npx expo install` plus a build | An SDK-aligned version and a successful build with RN 0.88.1, `compileSdk` 37 and `targetSdk` 36. |
| Directory standing | `https://reactnative.directory/package/<name>` | A listed package with a score, a manageable issue count, and not archived. |

Two more signals caught issues the four checks alone would have missed:

- **Deprecation notices.** npm marks `react-native-vector-icons@10.3.0` deprecated and points to
  per-icon-family packages. Its last release, 2025-07-23, is the oldest of the three.
- **Version range.** `npx expo install` pinned AsyncStorage to 2.2.0 even though npm offers 3.1.1 —
  an intentional SDK-58 alignment, not a stale install. Vector-icons resolved to a caret range,
  `^10.3.0`, rather than an Expo-pinned one, which needs rechecking before an SDK upgrade.

Directory data for this stage: AsyncStorage scores 90 with a New Architecture badge and 15 open
issues; safe-area-context scores 95 with Expo Go support and 78 open issues, but no New Architecture
badge on the page — its New Architecture proof comes from local codegen. Neither is archived.

The audit is not a reason to reject every imperfect package. Ask "what breaks if this disappears
tomorrow?", name the replacement and the migration steps, and write it down. A deprecated package
explicitly named in the brief can still be used, but the risk belongs in the open items, not in a
silent decision to change the implementation.

## 3. Install SDK-aligned versions

Let Expo resolve versions instead of typing them by hand:

```bash
npx expo install @react-native-async-storage/async-storage react-native-safe-area-context react-native-vector-icons
```

Confirm what landed in `package.json`: `2.2.0`, `~5.11.1` and `^10.3.0`. If a version looks older
than npm's latest, that is expected. Do not hand-upgrade a package to a version Expo did not select
for the SDK.

## 4. Link by hand where autolinking stops

Autolinking is the build tooling's discovery step. On iOS, CocoaPods reads the podspecs that
packages ship and links them into the Xcode project; on Android, `autolinkLibrariesWithApp()` does
the equivalent. Native modules that declare codegen specs also get native interface code generated
at this point. In this stage, `pod install` reported autolinking of `RNCAsyncStorage`,
`RNVectorIcons` and `react-native-safe-area-context`, and codegen produced files for
`rnasyncstorage`, `safeareacontext` and `RNVectorIconsSpec`.

What autolinking cannot describe is a font **asset**. A `.ttf` is a resource, not a native module
with a spec. The app's own native project must register it, on both platforms.

On iOS, open `ios/PersonalProfileApp/Info.plist` and add:

```xml
<key>UIAppFonts</key>
<array>
  <string>Ionicons.ttf</string>
</array>
```

On Android, add this line in `android/app/build.gradle`, after the plugin declarations and before
the `react { ... }` block:

```groovy
apply from: "../../node_modules/react-native-vector-icons/fonts.gradle"
```

The Android script copies the bundled fonts into the asset path at build time. Without either
registration, the module links but the glyphs render as missing characters.

Refresh the native dependencies after the change:

```bash
cd ios && pod install && cd ..
```

Expect `Pod installation complete!` with 101 pods in this run. Non-fatal toolchain noise is normal:
React Native prints a notice that direct `pod install` is deprecated in favour of
`npx expo run:ios`, and Expo prints an internal `CoreModule` access-level warning. Do not edit
`node_modules` to silence either. Gradle later emits its own deprecation warnings and manifest
`package=` notices from dependencies; none of these fail the build.

The practical rule: if a change belongs to app identity, assets or native project configuration,
expect to make it by hand and rebuild. If it belongs to a package's own native project, let
autolinking find it — but read the generated output instead of assuming it worked.

## 5. Prove it on both platforms

Add a screen that exercises all three modules at once. Wrap the tree in `SafeAreaProvider`, read
`useSafeAreaInsets()` inside a `SafeAreaView`, write then read a key with AsyncStorage, and render an
Ionicons `checkmark-circle`:

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

Each line is a proof: the icon renders only if the font registration worked, the `setItem`/`getItem`
pair proves AsyncStorage's native module is linked, and a non-zero inset proves safe-area-context is
live.

Because native files changed, a rebuild is required — fast refresh cannot apply them:

```bash
npx expo run:ios
npx expo run:android
```

This run produced `Build Succeeded` with `0 error(s), and 1 warning(s)` on iPhone 18 Pro, and
`BUILD SUCCESSFUL in 2m 11s` with `app-debug.apk` opened on `Medium_Phone_API_37.0`. If port 8081 is
busy, answering "no" to the alternate-port prompt makes Expo skip the dev server, but the native
build, install and launch still succeed.

Check the screen, not just the build log. Both platforms showed the green Ionicons checkmark,
`Icons: Ionicons rendered`, `AsyncStorage: AsyncStorage works`, and a safe-area top inset — 62 on the
iOS simulator and 24 on the Android emulator. The exact numbers vary by device; a zero or absent
value is the thing to investigate.

Finish with the health check:

```bash
npx expo-doctor
```

A clean result is `20/20 checks passed. No issues detected!`.

## 6. Constraints and open items

Keep the program constraints in view:

- React Native 0.88, Expo SDK 58, React 19.3.0, Node 24 LTS, TypeScript from M01 in strict style.
- New Architecture always on; Xcode 26.4 or newer; Android `compileSdk` 37 and `targetSdk` 36.
- `ios/` and `android/` stay committed; Expo-first builds with EAS Build throughout; EAS Update is
  out of scope.
- Install SDK-aligned libraries with `npx expo install`; keep client state, theme/session and
  server data in their own homes as the program defines them.

Some items are explicitly open and must not be silently settled in teaching material: the missing
trainer example audit row (the format here was created by the trainee), the instructor review
checkbox in the brief, the vector-icons deprecation as a program-owner scope decision, and the
cohort items listed in the README such as store accounts, the real-device pool and exact point
versions, and the course's TypeScript-from-M01 expectation versus the JS-only starter observed in
the app.

## Comprehension questions

1. Walk through how you would choose between a community module, a small Expo utility and custom native code for a new feature, and explain what that choice means for the committed `ios/` and `android/` folders.
2. What evidence would you gather before accepting a community dependency in this project, and how would you decide what to do when one of those signals is worrying?
3. How would you work out whether a newly installed native module needs hand-written wiring, and what would you inspect to confirm that autolinking and codegen did their job?
4. If an icon font renders as missing glyphs after a successful build, what would you check on each platform, and how would you explain why those steps were necessary?
5. After changing a native configuration file, what would you do to confirm the change is actually live on both platforms, and why would a JavaScript hot reload be insufficient?

## Understanding Checklist

- [ ] I can state the hybrid rule with an example from this app and explain why `expo prebuild` is not used here.
- [ ] I can name the four dependency-safety checks, say where each comes from, and describe how I would record a worrying signal and an exit plan.
- [ ] I can explain what autolinking does on iOS and Android and identify the kind of change it cannot handle.
- [ ] I can name the iOS and Android locations where a font asset must be registered and explain why autolinking does not do it.
- [ ] I can describe the rebuild and on-screen verification steps for a native change on both platforms, including the role of `expo-doctor`.
