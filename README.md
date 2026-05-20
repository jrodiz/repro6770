# repro6770

Minimal Android app that reproduces [firebase/firebase-android-sdk#6770](https://github.com/firebase/firebase-android-sdk/issues/6770) and verifies the fix proposed in [PR #8185](https://github.com/firebase/firebase-android-sdk/pull/8185).

## The bug

`com.google.firebase.crashlytics`'s `injectCrashlyticsMappingFileId<Variant>` task is not up-to-date-aware: it generates a fresh random UUID on every release build and writes it to `app/build/crashlytics/release/mappingFileId.txt`. That cascades through `mergeReleaseResources` → `processReleaseResources` → `minifyReleaseWithR8` → `packageRelease` and busts the build cache for every minified release build, even with no source changes.

## Environment

| | |
|---|---|
| AGP | `9.1.0-alpha05` |
| Gradle | `9.2.1` |
| Kotlin | `2.0.21` |
| `compileSdk` / `minSdk` | `36` / `24` |
| `isMinifyEnabled` (release) | `true` |
| `firebase-bom` | `34.13.0` |
| App package | `com.jrodiz.app.repro6770` |

`google-services.json` is wired to a real Firebase project so the plugin task graph matches a real consumer setup.

## How to reproduce the bug (pre-fix plugin `3.0.3`)

In `build.gradle.kts`, set:

```kotlin
id("com.google.firebase.crashlytics") version "3.0.3" apply false
```

In `app/build.gradle.kts`, comment out the `mappingFileUploadEnabled = false` block under `buildTypes.release` (the toggle is left on in this repo to also demonstrate the blank-id path — see [`TEST_RESULTS_ISSUE_6770.md`](TEST_RESULTS_ISSUE_6770.md) phase C.3).

Then:

```bash
./gradlew :app:assembleRelease
cat app/build/crashlytics/release/mappingFileId.txt
./gradlew :app:assembleRelease           # no source changes
cat app/build/crashlytics/release/mappingFileId.txt   # different value — the bug
```

## How to verify the fix (PR #8185)

1. Check out the fix branch in a local clone of `firebase-android-sdk` and publish to mavenLocal:

   ```bash
   ./gradlew :firebase-crashlytics-gradle:publishToMavenLocal \
             :firebase-crashlytics-buildtools:publishToMavenLocal
   ```

2. In this project's `build.gradle.kts`, point at the local version:

   ```kotlin
   id("com.google.firebase.crashlytics") version "3.0.7" apply false
   ```

   `settings.gradle.kts` already declares `mavenLocal()` first in `pluginManagement.repositories`, so nothing else needs to change.

3. Rebuild:

   ```bash
   ./gradlew clean :app:assembleRelease
   ./gradlew :app:assembleRelease           # should stay UP-TO-DATE
   ```

   `mappingFileId.txt` should be byte-identical between runs and the downstream task cascade (`:mergeReleaseResources`, `:processReleaseResources`, `:minifyReleaseWithR8`, `:packageRelease`, `:assembleRelease`) should report `UP-TO-DATE`.

## Detailed results

Full transcript of pre-fix vs. post-fix runs, including `mappingFileId.txt` values, per-task `UP-TO-DATE` status, and wall-clock times, is in [`TEST_RESULTS_ISSUE_6770.md`](TEST_RESULTS_ISSUE_6770.md).
