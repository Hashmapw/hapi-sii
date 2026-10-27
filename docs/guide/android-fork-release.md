# Android APK releases in hapi-sii

The fork's **Android APK Release (fork)** workflow builds a signed, optimized
APK from an explicit source commit and publishes it as a GitHub prerelease.
It does not require Firebase configuration. Such builds support normal hub
connections but have no Firebase background push notifications.

## Build locally

Install JDK 17 or newer (CI uses 21) and the Android SDK, then set
`ANDROID_HOME`. The Gradle wrapper supplies Gradle; compile/target SDK is 36.
From the repository root:

```sh
cd android
./gradlew :core:protocol:test :core:data:testDebugUnitTest :app:testDebugUnitTest
./gradlew :app:assembleDebug :app:lintDebug
```

The debug APK is `android/app/build/outputs/apk/debug/app-debug.apk` relative
to the repository root. For an optimized release APK, configure a persistent
signing key as described in the [Android README](https://github.com/Hashmapw/hapi-sii/blob/main/android/README.md#release-signing)
and run `./gradlew :app:assembleRelease :app:lintRelease` from `android/`.
Without signing configuration, a release APK is unsigned and cannot be installed.

## Publish a main snapshot

The workflow is restricted to `Hashmapw/hapi-sii`, dispatched from `main`.
Configure these Actions repository secrets once:

| Secret | Meaning |
|---|---|
| `HAPI_SII_ANDROID_KEYSTORE_BASE64` | Base64-encoded keystore containing alias `hapi-sii` |
| `HAPI_SII_ANDROID_KEYSTORE_PASSWORD` | Password used for both keystore and key |

Back up the keystore and password privately. For APKs distributed directly,
this key is the app's actual update identity; Google Play cannot recover it.
Do not commit the key or password. A new CI runner must reuse the same key.

Open **Actions → Android APK Release (fork) → Run workflow** and provide:

| Input | Example |
|---|---|
| `source_commit` | Full 40-character SHA of the main snapshot to build |
| `version_code` | `1` for the first build; increase on every subsequent release |
| `version_name` | `0.30.7-main.20261005.5153ff23` |
| `release_tag` | `android-main-20261005-5153ff23` (must be new) |

The version embedded in the source tree may still say `0.30.7` after new
commits have landed. The workflow overrides the APK's version and records
the full source SHA in `build-info.json`. The release tag points to that
source commit, while `workflow_commit` records the workflow revision.
Tags start with `android-` so they do not match the CLI/Hub `v*` release trigger.

Tests, release lint, APK signature verification and package metadata checks
must pass before publication. The Release includes the APK, SHA-256 checksums,
build provenance, APK metadata and the public signing certificate fingerprint.
Test installation and pairing on a real Android device separately.

The app requires Android 8.0 or newer and an HTTPS hub. Its package ID remains
`run.hapi.companion`: an existing installation signed by a different key must
be uninstalled first, which removes its local data. Versions signed by this
fork's persistent key can update each other when `versionCode` increases.
