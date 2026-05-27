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

## Phase C — Verify the original fix (id reused unconditionally)

> Phase C documents the behavior of the initial #8185 fix, which reused the id across rebuilds **unconditionally** (regardless of source edits). The follow-up commits replace that model with content-driven invalidation — see **Phase D**.

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

## Phase D — Verify the content-driven follow-up (id stable while sources stable, regenerated on source change)

The follow-up commits on `feature/jrc--6770.Fix.release.build.cache.invalidation` — `e7499f3e3` (content-driven mapping file id), `dcfd1228b` (source-change test + harness fix), `0502746af` (changelog amend) — change `InjectMappingFileIdTask` from "reuse the id forever" to "let Gradle's input snapshotting decide". A new `@InputFiles obfuscatableSources` collection (path-sensitive, relative) is populated from `project.fileTree("src")` with `java/**` + `kotlin/**` patterns, tests excluded; `useBlankMappingFileId` is promoted to `@Input`. The on-disk reuse path and the custom `outputs.upToDateWhen { }` predicate are removed.

The check we care about is no longer "does the id stay identical across rebuilds?" but "does the id stay identical *while sources are unchanged*, and does it regenerate when sources change?"

### D.0 — Republish the plugin from the follow-up branch

```bash
# in /Users/jrodiz/AndroidStudioProjects/firebase-android-sdk on the fix branch
./gradlew :firebase-crashlytics-gradle:publishToMavenLocal
```

Verified the published jar at `~/.m2/repository/com/google/firebase/firebase-crashlytics-gradle/3.0.7/firebase-crashlytics-gradle-3.0.7.jar` contains the new task inputs (`useBlankMappingFileId`, `obfuscatableSources`, `mappingFileIdFile`).

### D.1 — Stable id + UP-TO-DATE cascade while sources are unchanged

Repro app's release build temporarily flipped to `mappingFileUploadEnabled = true` (the upload-enabled path is where the content-driven invalidation actually engages — see D.4 for the blank-id mode).

```bash
cd /Users/jrodiz/AndroidStudioProjects/repro6770
./gradlew clean :app:assembleRelease       # build #1 — fresh id
cat app/build/crashlytics/release/mappingFileId.txt
./gradlew :app:assembleRelease             # build #2 — no source edits
cat app/build/crashlytics/release/mappingFileId.txt
```

| | Build #1 (post-clean) | Build #2 (no source edits) |
|---|---|---|
| `mappingFileId.txt` | `f70a7386acaa4a3081280ae61442d575` | `f70a7386acaa4a3081280ae61442d575` (**identical**) |
| `:app:injectCrashlyticsMappingFileIdRelease` | executed | **UP-TO-DATE** |
| `:app:mergeReleaseResources` | executed | **UP-TO-DATE** |
| `:app:processReleaseResources` | executed | **UP-TO-DATE** |
| `:app:compileReleaseKotlin` | executed | **UP-TO-DATE** |
| `:app:minifyReleaseWithR8` | executed | **UP-TO-DATE** |
| `:app:packageRelease` | executed | **UP-TO-DATE** |
| `:app:assembleRelease` (lifecycle) | executed | **UP-TO-DATE** |
| Wall time | 17 s | **4 s** |

The two non-cacheable tasks (`:app:lintVitalRelease` and `:app:uploadCrashlyticsMappingFileRelease`) still execute on build #2 by design. Every cache-relevant task is `UP-TO-DATE`.

### D.2 — Source edit regenerates the id and re-runs the downstream cascade

Edited `app/src/main/java/com/jrodiz/app/repro6770/ui/theme/Color.kt` (added one line: `val ReproProbe = Color(0xFF000001)`) and rebuilt without `clean`:

```bash
./gradlew :app:assembleRelease
cat app/build/crashlytics/release/mappingFileId.txt
```

| | Build #2 (before edit) | Build #3 (after one-line `.kt` edit) |
|---|---|---|
| `mappingFileId.txt` | `f70a7386acaa4a3081280ae61442d575` | `f341bb62a188462fb26a43629de742f1` (**new**) |
| `:app:injectCrashlyticsMappingFileIdRelease` | UP-TO-DATE | **executed** (input snapshot tripped on the `.kt` change) |
| `:app:mergeReleaseResources` | UP-TO-DATE | **executed** (consumes the inject task's output) |
| `:app:processReleaseResources` | UP-TO-DATE | **executed** |
| `:app:compileReleaseKotlin` | UP-TO-DATE | **executed** |
| `:app:minifyReleaseWithR8` | UP-TO-DATE | **executed** |
| `:app:packageRelease` | UP-TO-DATE | **executed** |
| `:app:uploadCrashlyticsMappingFileRelease` | executed | **executed** (new mapping uploaded under the new id) |
| Wall time | 4 s | 17 s |

This closes the correctness gap that the original #8185 fix left open: prior to this follow-up, a code edit between two builds in the same workspace produced a different `mapping.txt` under the **same** id — the second upload would overwrite the first on the backend and crashes from the first APK would symbolicate against the wrong mapping. With the follow-up, every distinct `mapping.txt` ships with a distinct id.

### D.3 — `clean` continues to mint a fresh id

Implicit in D.1: build #1 was a `clean :app:assembleRelease` and produced an id different from any prior run.

### D.4 — Blank-id path (`mappingFileUploadEnabled = false`) remains stable across rebuilds

Reverted the repro app to its original `mappingFileUploadEnabled = false` (its committed state). A clean release build produced:

| | Blank-id build |
|---|---|
| `mappingFileId.txt` | `00000000000000000000000000000000` (`BLANK_MAPPING_FILE_ID`) |
| `:app:uploadCrashlyticsMappingFileRelease` | not registered (uploads off) |
| `obfuscatableSources` wiring on the inject task | **intentionally not wired** in this mode (only `useBlankMappingFileId=true` as an input) |

The blank-id mode is unchanged by the follow-up — same behavior as Phase C.4: stable id `0…0` across rebuilds, inject task UP-TO-DATE on the second run, and the mode flip itself (the `useBlankMappingFileId` `@Input`) invalidates the task natively without any custom predicate.

### D.5 — Working tree restored

After D.1–D.3, the temporary edits in repro6770 (`mappingFileUploadEnabled = true`, `val ReproProbe = …`) were reverted; `git status` in `/Users/jrodiz/AndroidStudioProjects/repro6770` reports only the untracked `.idea/` directory. No commits were made in either repo as part of this verification.

### Known gap (documented, not closed here)

Dependency-only changes — e.g. a `firebase-bom` or library version bump with no host source edit — do **not** regenerate the id under this implementation. `variant.compileClasspath` would have closed the gap but cycled through AGP's project-dep graph and was dropped. Real-world impact is narrow because release CI typically runs from clean checkouts (which mint a fresh id naturally per D.3). A future fix would either bump the AGP `compileOnly` to ≥8.6 (to access `variant.sources.*.static`) or hash R8's output `mapping.txt` in a post-R8 finalizer.

---

## Phase E — Jetpack Compose stress test (Compose Compiler in the loop)

The repro app already pulls in the Kotlin 2.0 Compose Compiler plugin (`alias(libs.plugins.kotlin.compose)`), `buildFeatures.compose = true`, and a Compose Material3 / preview-tooling stack. The Compose Compiler runs **inside** `compileReleaseKotlin` and emits everything to `build/`, never to `src/`, so conceptually our `fileTree("src") { include("**/*.kt") }` input snapshot shouldn't see anything Compose-generated. Phase E verifies that conceptual claim end-to-end by editing different *kinds* of Compose constructs and confirming the cascade reacts exactly the same as for plain Kotlin.

All six builds run against the same `mappingFileUploadEnabled = true` setup as Phase D. No KSP / KAPT / annotation processors are involved.

| # | Scenario | `inject…MappingFileIdRelease` | `compileReleaseKotlin` | `minifyReleaseWithR8` | `packageRelease` | `assembleRelease` | `mappingFileId.txt` | Wall time |
|---|----------|-------------------------------|------------------------|------------------------|------------------|--------------------|---------------------|-----------|
| E.1 | Clean release on Compose app | **SUCCESS** | SUCCESS | SUCCESS | SUCCESS | SUCCESS | `a8ebe255e57949f4963a00d778dfe6e0` | 15 s |
| E.2 | No-op rebuild | **UP-TO-DATE** | **UP-TO-DATE** | **UP-TO-DATE** | **UP-TO-DATE** | **UP-TO-DATE** | `a8ebe255e57949f4963a00d778dfe6e0` (same) | **3 s** |
| E.3 | Edit `@Composable` body (`Greeting`: `"Hello $name!"` → `"Hi $name!"`) | **SUCCESS** | SUCCESS | SUCCESS | SUCCESS | SUCCESS | `4c9a975837aa4184bfae6b913e8b295b` (new) | 15 s |
| E.4 | Add a new `@Composable` (`fun Farewell(name: String, ...)`) | **SUCCESS** | SUCCESS | SUCCESS | SUCCESS | SUCCESS | `f05ad5bfcb404328861d1d0dacc7e9cc` (new) | 16 s |
| E.5 | Edit `@Preview` composable body (`GreetingPreview`: `"Android"` → `"Compose"`) | **SUCCESS** | SUCCESS | SUCCESS | SUCCESS | SUCCESS | `d96af36d0c66467aa5509d78f22576da` (new) | 16 s |
| E.6a | `git checkout` MainActivity.kt (revert all E.3–E.5 edits) | **SUCCESS** | SUCCESS | SUCCESS | SUCCESS | SUCCESS | `fed451b85c744cfa8f5a1a319e8536e9` (new) | 16 s |
| E.6b | Immediate no-op rebuild after E.6a | **UP-TO-DATE** | **UP-TO-DATE** | **UP-TO-DATE** | **UP-TO-DATE** | **UP-TO-DATE** | `fed451b85c744cfa8f5a1a319e8536e9` (same) | **3 s** |

### Observations

- **No Compose-induced cache churn.** E.2 — the most sensitive test, since it exercises a no-op rebuild on a Compose project — shows `compileReleaseKotlin` (the task hosting the Compose Compiler) and every downstream task UP-TO-DATE, with the id byte-identical to E.1. The Compose Compiler does not write into `src/`, so it doesn't bleed into our input snapshot.
- **Every category of Compose source edit invalidates the id.** Body edits (E.3), function additions (E.4), and `@Preview`-only edits (E.5) all flip the id and re-run the full R8 / packaging cascade — same behavior as a plain `.kt` edit.
- **A→B→A still re-fires once, then settles.** E.6a fired (input snapshot differs from E.5's), E.6b is UP-TO-DATE again — Gradle's task cache works as expected. The id minted at E.6a is not equal to E.1's because the inject task always mints a fresh UUID when it runs (by design — only Gradle's input snapshotting decides whether it runs, the task body itself doesn't reuse anything).
- **`@Preview` functions count.** They live in `src/main/**/*.kt`, so editing one trips the input snapshot. This is the expected and desired behavior: changing a `@Preview` won't typically change `mapping.txt` (since the preview is also minified and R8's output is determined by all minified code), but the conservative behavior of "any user source change → new id" is correct and matches the spec from #6770.

### Conclusion

The content-driven fix behaves identically on the Compose app as on plain-Kotlin code. The Compose Compiler plugin does not interfere with Gradle's input snapshotting of our `obfuscatableSources` collection, and no spurious cache invalidation surfaces across any of the Compose edit patterns tested.

### Working tree restored

`git status` in `/Users/jrodiz/AndroidStudioProjects/repro6770` reports only the untracked `.idea/` directory after Phase E. `mappingFileUploadEnabled` reverted to `false` (its committed state); `MainActivity.kt` restored via `git checkout`. No commits made in either repo.

---

## Summary

| Scenario | `:app:injectCrashlyticsMappingFileIdRelease` | `mappingFileId.txt` | Downstream tasks |
|---|---|---|---|
| **Pre-fix `3.0.3`** — rebuild | re-executed every time | **changes every build** | re-executed (cache busted) |
| **Initial fix (#8185)** — rebuild | **UP-TO-DATE** | **identical** across runs | **UP-TO-DATE** |
| **Initial fix (#8185)** — rebuild after source edit | **UP-TO-DATE** | identical (correctness gap) | **UP-TO-DATE** |
| **Follow-up (content-driven)** — rebuild, no source edits | **UP-TO-DATE** | **identical** across runs | **UP-TO-DATE** |
| **Follow-up (content-driven)** — rebuild after `.kt` edit | **executed** | **new value** | **re-executed** |
| **Follow-up (content-driven)** — after `clean` | re-executed | new value | re-executed (cold) |
| **Follow-up (content-driven)** — blank-id mode rebuild | **UP-TO-DATE** | stays blank `0…0` | **UP-TO-DATE** |
| **Follow-up (content-driven)** — toggle `mappingFileUploadEnabled` | re-executed | flips on mode change | re-executed |

The follow-up delivers what #6770 asked for (release builds stop being invalidated on every run) **and** closes the correctness gap the initial fix left open: a source edit between two release builds now produces a new id, so distinct `mapping.txt` artifacts never share an id on the Crashlytics backend. The blank-id, `clean`, and mode-toggle invalidation paths continue to behave correctly. Phase E confirms the same behavior holds on a Jetpack Compose app — the Compose Compiler does not interfere with the input snapshot, and every category of Compose source edit (`@Composable` body, new composable, `@Preview` body) propagates correctly. No SDK runtime change required.
