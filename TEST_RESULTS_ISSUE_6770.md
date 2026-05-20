# Test Results — Issue #6770

End-to-end reproduction of the bug and verification of the fix in a real Android app outside this repo.

## Environment

| Item | Value |
|---|---|
| Repro app path | `/Users/jrodiz/AndroidStudioProjects/repro6770` |
| App package | `com.jrodiz.app.repro6770` |
| AGP | `9.1.0-alpha05` |
| Gradle wrapper | `9.2.1` |
| Kotlin | `2.0.21` (Compose) |
| `compileSdk` / `minSdk` | `36` / `24` |
| `isMinifyEnabled` (release) | `true` |
| `firebase-bom` | `34.13.0` |
| `google-services` plugin | `4.4.4` |
| **Pre-fix plugin (repro)** | `com.google.firebase.crashlytics 3.0.3` (published) |
| **Fixed plugin (verify)** | `com.google.firebase.crashlytics 3.0.7` from mavenLocal, branch `feature/jrc--6770.Fix.release.build.cache.invalidation`, commits `b2ecb0daf` + `b02cde8d5` |
| `google-services.json` | real (Firebase project `repro6770`) |

Repro app artifacts inspected each run:
| File | Purpose |
|---|---|
| `app/build/crashlytics/release/mappingFileId.txt` | The id the plugin minted/read this run |
| `app/build/generated/res/injectCrashlyticsMappingFileIdRelease/values/com_google_firebase_crashlytics_mappingfileid.xml` | The XML resource the SDK reads at runtime |

---

## Phase A — Reproduce the bug (pre-fix plugin `3.0.3`)

### Wiring used

Root `build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.compose) apply false
    id("com.google.gms.google-services") version "4.4.4" apply false
    id("com.google.firebase.crashlytics") version "3.0.3" apply false
}
```

`app/build.gradle.kts` plugins block:
```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}
```

### Commands

```bash
cd /Users/jrodiz/AndroidStudioProjects/repro6770
./gradlew :app:assembleRelease    # build #1 (cold)
cat app/build/crashlytics/release/mappingFileId.txt
./gradlew :app:assembleRelease    # build #2 (no source changes — the smoking gun)
cat app/build/crashlytics/release/mappingFileId.txt
```

### Observed result

| | Build #1 | Build #2 (no source changes) |
|---|---|---|
| `mappingFileId.txt` | `f5f03101bb964fc392f7af9cd39f9c21` | `6514b43e4b5b4e96ab19a8deb5869aec` (changed) |
| `:app:injectCrashlyticsMappingFileIdRelease` | executed | **executed again** (NOT UP-TO-DATE) |
| `:app:processReleaseResources` | executed | **re-executed** |
| `:app:uploadCrashlyticsMappingFileRelease` | executed | **re-executed** |
| `:app:packageRelease` | executed | **re-executed** |
| `:app:assembleRelease` (lifecycle) | executed | **re-executed** |
| `:app:minifyReleaseWithR8` | executed | UP-TO-DATE (R8's input snapshot didn't trip on the resource change) |

**Conclusion:** nothing changed in sources, but the id flipped and a chunk of downstream work was re-done. Matches the report in #6770 and the duplicate #7185.

---

## Phase B — Build the fixed plugin locally

### Commands

```bash
# in /Users/jrodiz/AndroidStudioProjects/firebase-android-sdk on the fix branch
rm -rf ~/.m2/repository/com/google/firebase/firebase-crashlytics-gradle \
       ~/.m2/repository/com/google/firebase/firebase-crashlytics-buildtools
./gradlew :firebase-crashlytics-gradle:publishToMavenLocal \
          :firebase-crashlytics-buildtools:publishToMavenLocal
```

> Note: `-PprojectsToPublish="firebase-crashlytics-gradle,firebase-crashlytics-buildtools" publishReleasingLibrariesToMavenLocal` was tried first but only deployed the buildtools jar; calling `:publishToMavenLocal` on both modules directly is what actually works.

### Verified artifacts

| Artifact | Path | Version |
|---|---|---|
| Gradle plugin | `~/.m2/repository/com/google/firebase/firebase-crashlytics-gradle/3.0.7/` | `3.0.7` |
| Buildtools jar | `~/.m2/repository/com/google/firebase/firebase-crashlytics-buildtools/3.0.7/` | `3.0.7` |

The repro app's `settings.gradle.kts` already has `mavenLocal()` at the top of `pluginManagement.repositories`, so no settings change was needed — just bump the version line in the root `build.gradle.kts`:

```kotlin
id("com.google.firebase.crashlytics") version "3.0.7" apply false
```

---

## Phase C — Verify the fix

### C.1 — Idempotent rebuild

```bash
./gradlew clean :app:assembleRelease    # build #1 — fresh id
cat app/build/crashlytics/release/mappingFileId.txt
./gradlew :app:assembleRelease           # build #2 — should stay UP-TO-DATE
cat app/build/crashlytics/release/mappingFileId.txt
```

| | Build #1 (post-clean) | Build #2 (no changes) |
|---|---|---|
| `mappingFileId.txt` | `2c6677f620ae41898c2832d18c591214` | `2c6677f620ae41898c2832d18c591214` (**identical**) |
| `:app:injectCrashlyticsMappingFileIdRelease` | executed | **UP-TO-DATE** |
| `:app:mergeReleaseResources` | executed | **UP-TO-DATE** |
| `:app:processReleaseResources` | executed | **UP-TO-DATE** |
| `:app:minifyReleaseWithR8` | executed | **UP-TO-DATE** |
| `:app:packageRelease` | executed | **UP-TO-DATE** |
| Wall time | 1 m 12 s (cold) | 3 s (`47 up-to-date / 2 executed`) |

The two tasks still "executed" on build #2 are `:app:lintVitalRelease` and `:app:uploadCrashlyticsMappingFileRelease` — both are lifecycle-style tasks that are not cacheable by design. Everything cache-relevant is `UP-TO-DATE`.

### C.2 — `clean` still forces a fresh id

```bash
./gradlew clean :app:assembleRelease
```

| | Before `clean` | After `clean` + build |
|---|---|---|
| `mappingFileId.txt` | `2c6677f620ae41898c2832d18c591214` | `fc57f791b33f4fd79840f5e74a2fb3b2` (**new**) |
| `:app:injectCrashlyticsMappingFileIdRelease` | (n/a) | **executed** (no on-disk id to reuse) |

### C.3 — Toggling `mappingFileUploadEnabled` invalidates the task

Added to `app/build.gradle.kts` inside `buildTypes.release { }`:

```kotlin
configure<com.google.firebase.crashlytics.buildtools.gradle.CrashlyticsExtension> {
    mappingFileUploadEnabled = false
}
```

| | Before toggle | After toggle |
|---|---|---|
| `mappingFileId.txt` | `fc57f791b33f4fd79840f5e74a2fb3b2` | `00000000000000000000000000000000` (blank id) |
| `:app:injectCrashlyticsMappingFileIdRelease` | UP-TO-DATE | **executed** (mode changed `false → true` for `useBlankMappingFileId`) |
| `:app:uploadCrashlyticsMappingFileRelease` | in graph | **removed** from graph (plugin doesn't register it when uploads are off) |

### C.4 — Idempotent rebuild also works in the blank-id mode

```bash
./gradlew :app:assembleRelease   # second run after toggling off — no source changes
```

| | Run after toggle | Next run (no changes) |
|---|---|---|
| `mappingFileId.txt` | `00000000000000000000000000000000` | `00000000000000000000000000000000` (identical) |
| `:app:injectCrashlyticsMappingFileIdRelease` | executed | **UP-TO-DATE** |
| `:app:assembleRelease` (lifecycle) | executed | **UP-TO-DATE** |
| Wall time | ~4 s | **1 s** |

---

## Summary

| Scenario | `:app:injectCrashlyticsMappingFileIdRelease` | `mappingFileId.txt` | Downstream tasks |
|---|---|---|---|
| **Pre-fix `3.0.3`** — rebuild | re-executed every time | **changes every build** | re-executed (cache busted) |
| **Fix** — rebuild | **UP-TO-DATE** | **identical** across runs | **UP-TO-DATE** |
| **Fix** — after `clean` | re-executed | new value | re-executed (cold) |
| **Fix** — toggle `mappingFileUploadEnabled = false` | re-executed | flips to blank `0…0` | re-executed |
| **Fix** — rebuild in blank-id mode | **UP-TO-DATE** | stays blank | **UP-TO-DATE** |

The fix delivers what #6770 asked for (release builds stop being invalidated on every run) and preserves the necessary regeneration paths (`clean`, mode toggle). No SDK runtime change required.
