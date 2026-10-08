# ci-workflows

Reusable GitHub Actions workflows shared across all app repos.
Fix or upgrade a workflow once here, and every app that calls it gets the change.

| Workflow | Purpose |
|---|---|
| [`android-build.yml`](.github/workflows/android-build.yml) | Builds a signed Android APK/AAB for **Flutter** or **native Android** (auto-detected) |

---

## Table of contents

1. [How it works](#how-it-works)
2. [Quick start (add CI to a new app)](#quick-start--add-ci-to-a-new-app)
3. [Caller file: `build.yml`](#caller-file-buildyml)
4. [Inputs reference](#inputs-reference)
5. [Signing setup](#signing-setup)
6. [Common recipes](#common-recipes)
7. [Outputs: where to find the APK](#outputs--where-to-find-the-apk)
8. [Updating & versioning this repo](#updating--versioning-this-repo)
9. [Troubleshooting](#troubleshooting)
10. [How the workflow works internally](#how-the-workflow-works-internally)

---

## How it works

```
App repo                                      ci-workflows repo
.github/workflows/build.yml   ──calls──▶   .github/workflows/android-build.yml
  (8 lines: when + which options)            (all the build logic)
```

- **`android-build.yml`** (this repo) contains all the logic. It has only an `on: workflow_call`
  trigger, so it never runs in this repo by itself. An empty Actions tab here is expected.
- **`build.yml`** (each app repo) is a thin caller. It decides *when* to build (push, tag, manual)
  and *which options* to use, then delegates to this repo.

Detection:

| File found in `working-directory` | Treated as | Gradle root |
|---|---|---|
| `pubspec.yaml` | Flutter | `android/` |
| `gradlew` | Native Android | `.` |
| neither | ❌ fails with a clear error | — |

---

## Quick start: add CI to a new app

### 1. Add the caller file

Create `.github/workflows/build.yml` in the app repo (next to `lib/`, `android/`, `pubspec.yaml`)
and paste the [caller template](#caller-file-buildyml).

Using the GitHub web UI: **Add file → Create new file**, then type `.github/workflows/build.yml`
as the name. Typing `/` creates the folders.

### 2. Check two things

- `uses:` points to `JAVIYARAJ/ci-workflows/.github/workflows/android-build.yml@v1`
- `branches: [main]` matches the repo's real default branch (`main` / `master` / `develop`)

### 3. Push

```bash
git add .github/workflows/build.yml
git commit -m "Add Android CI build"
git push
```

### 4. Verify

App repo → **Actions** tab → **Build APK** run → bottom of the page → **Artifacts** → download the zip.

> The first run without secrets gives a **debug-signed** APK (Flutter) and a yellow warning.
> That's expected and confirms the pipeline works. Then do the [signing setup](#signing-setup).

---

## Caller file: `build.yml`

Copy this into every app at `.github/workflows/build.yml`:

```yaml
name: Build APK

on:
  push:
    branches: [main]
    tags: ["v*"]          # push a v1.2.0 tag -> APK attached to a GitHub Release
  workflow_dispatch:       # manual "Run workflow" button in the Actions tab

permissions:
  contents: write          # needed only for create-release

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true # a newer push cancels the older running build

jobs:
  android:
    uses: JAVIYARAJ/ci-workflows/.github/workflows/android-build.yml@v1
    with:
      build-type: release
      create-release: true
      # flavor: prod
      # build-aab: true
      # split-per-abi: true
      # flutter-version: 3.35.0
      # build-number: ${{ github.run_number }}
      # working-directory: apps/mobile
    secrets: inherit       # passes the app repo's signing secrets to the workflow
```

### What each part does

| Section | Meaning | When to change |
|---|---|---|
| `on.push.branches` | Branches that trigger a build | Repo uses `master`/`develop`, or you want builds on more branches |
| `on.push.tags` | Tag pattern that triggers a release build | Different tagging convention |
| `workflow_dispatch` | Adds a manual **Run workflow** button | Keep it |
| `permissions` | Lets the job create GitHub Releases | Remove only if `create-release: false` |
| `concurrency` | Cancels outdated in-progress builds | Remove if every commit must build |
| `uses: …@v1` | Which workflow and version to run | Bump to `@v2` when a breaking version exists |
| `with:` | Build options | See [Inputs reference](#inputs-reference) |
| `secrets: inherit` | Forwards repo/org secrets | Keep it |

---

## Inputs reference

All inputs are optional.

| Input | Type | Default | Applies to | Description |
|---|---|---|---|---|
| `build-type` | string | `release` | both | `release` or `debug` |
| `flavor` | string | `""` | both | Product flavor name, e.g. `prod`, `staging` |
| `working-directory` | string | `.` | both | Path to the app inside the repo (monorepos) |
| `java-version` | string | `17` | both | JDK version. Use `21` if your AGP requires it |
| `flutter-version` | string | `""` (latest stable) | Flutter | Pin an exact version, e.g. `3.35.0` |
| `build-number` | string | `""` | Flutter | Overrides `versionCode`. Empty means use pubspec |
| `split-per-abi` | boolean | `false` | Flutter | Separate, smaller APKs per CPU architecture |
| `build-aab` | boolean | `false` | both | Also build an `.aab` for the Play Store |
| `create-release` | boolean | `false` | both | On tag pushes, attach outputs to a GitHub Release |
| `retention-days` | number | `14` | both | How long Artifacts are kept (1–90) |

### Secrets (all optional)

| Secret | Value |
|---|---|
| `KEYSTORE_BASE64` | The `.jks` keystore file, base64-encoded on a single line |
| `KEYSTORE_PASSWORD` | Keystore (store) password |
| `KEY_ALIAS` | Key alias, e.g. `upload` |
| `KEY_PASSWORD` | Key password |

If `KEYSTORE_BASE64` is missing:
- **Flutter release** builds fall back to debug keys. They install for testing but **can't be uploaded to Play**.
- **Native release** builds produce `app-release-unsigned.apk`, which **won't install**.

---

## Signing setup

Do this once per app, or once per org if all apps share a keystore.

### 1. Create a keystore (skip if you already have one)

```bash
keytool -genkey -v -keystore upload-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

> Back up the keystore and passwords somewhere safe, such as a password manager.
> **Losing them means you can't update the app on Play** (unless Play App Signing is enabled).

### 2. Base64-encode it

```bash
# Linux / Git Bash
base64 -w0 upload-keystore.jks > keystore.txt

# macOS
base64 -i upload-keystore.jks | tr -d '\n' > keystore.txt

# Windows PowerShell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("upload-keystore.jks")) > keystore.txt
```

### 3. Add the secrets

App repo → **Settings → Secrets and variables → Actions → New repository secret**.
Add all four: `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`.

Shared keystore across apps: add them at **org level** (Org → Settings → Secrets), so new repos need nothing.

### 4. Clean up

```bash
rm keystore.txt
```

Never commit `.jks`, `keystore.txt`, or `key.properties`. Add them to `.gitignore`.

> ⚠️ Avoid backslashes (`\`) in keystore passwords. The `.properties` format treats them as escape characters.

---

## Common recipes

### Debug build on every branch, release on main

```yaml
on:
  push:
    branches: ["**"]
jobs:
  android:
    uses: JAVIYARAJ/ci-workflows/.github/workflows/android-build.yml@v1
    with:
      build-type: ${{ github.ref == 'refs/heads/main' && 'release' || 'debug' }}
    secrets: inherit
```

### Play Store build (AAB + auto-increment versionCode)

```yaml
    with:
      build-type: release
      build-aab: true
      build-number: ${{ github.run_number }}
```

> ⚠️ `github.run_number` starts at 1. If the app already has a higher `versionCode` on Play,
> use an offset instead: `build-number: ${{ github.run_number + 1000 }}`.

### Flavors (staging + prod in parallel)

```yaml
jobs:
  staging:
    uses: JAVIYARAJ/ci-workflows/.github/workflows/android-build.yml@v1
    with: { flavor: staging }
    secrets: inherit
  prod:
    uses: JAVIYARAJ/ci-workflows/.github/workflows/android-build.yml@v1
    with: { flavor: prod, build-aab: true }
    secrets: inherit
```

### Monorepo (app in a subfolder)

```yaml
    with:
      working-directory: apps/mobile
```

### Pin the Flutter version (recommended for client projects)

```yaml
    with:
      flutter-version: 3.35.0
```

This prevents surprise breakages when a new stable Flutter release comes out.

### Smaller APKs for sharing with testers

```yaml
    with:
      split-per-abi: true
```

Gives `app-arm64-v8a-release.apk` and others. Most modern phones need `arm64-v8a`.

### Build only on tags (no build on every push)

```yaml
on:
  push:
    tags: ["v*"]
  workflow_dispatch:
```

---

## Outputs: where to find the APK

| Trigger | Where the APK goes |
|---|---|
| Push to a branch | Run page → **Artifacts** → `<repo>-<build-type>-<run#>.zip` (kept `retention-days`) |
| Push a `v*` tag with `create-release: true` | **Releases** page of the app repo, a permanent link you can share with clients |
| Manual run | Same as a branch push |

Tag-based release:

```bash
git tag v1.0.0
git push origin v1.0.0
```

---

## Updating & versioning this repo

App repos pin `@v1`, so a change here **doesn't reach any app until the tag moves**.

### Non-breaking change (bug fix, new optional input)

```bash
git add .
git commit -m "Fix: ..."
git push
git tag -f v1
git push -f origin v1     # all apps on @v1 pick it up on their next run
```

### Breaking change (renamed or removed input, changed default behavior)

```bash
git tag v2
git push origin v2
```

Then update app repos one by one: `@v1` → `@v2`.

### Testing a change before moving the tag

In one app's `build.yml`, temporarily point at the branch:

```yaml
uses: JAVIYARAJ/ci-workflows/.github/workflows/android-build.yml@my-feature-branch
```

Push, verify, then move the tag and revert the app to `@v1`.

### Private repo access (one-time)

If this repo is private: **Settings → Actions → General → Access** →
*"Accessible from repositories owned by the user/organization"*.
Without this, app repos fail with *"workflow was not found"*.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Push does nothing | `branches:` doesn't match the actual branch | Fix the branch name in `build.yml` |
| `workflow was not found` | Private repo access not enabled, or the `v1` tag isn't pushed | See [Private repo access](#private-repo-access-one-time); run `git push origin v1` |
| `No pubspec.yaml or gradlew found` | App is in a subfolder | Set `working-directory` |
| `Resource not accessible by integration` | Missing release permission | Add `permissions: contents: write` to `build.yml` |
| Java / AGP version error | AGP needs a newer JDK | `java-version: '21'` |
| `Keystore was tampered with, or password was incorrect` | Wrong password or corrupt base64 | Re-encode with `-w0` (single line); re-check secrets |
| `Cannot recover key` | Wrong `KEY_PASSWORD` or `KEY_ALIAS` | Check them with `keytool -list -v -keystore upload-keystore.jks` |
| APK named `*-unsigned.apk` (native) | Signing secrets missing | Add the 4 secrets |
| Play rejects: "signed in debug mode" | Signing secrets missing (Flutter fallback) | Add the 4 secrets |
| Play rejects: "version code already used" | versionCode not incremented | Bump pubspec version, or use `build-number` with an offset |
| `Task 'assembleXxxRelease' not found` | Flavor name typo | Must match `productFlavors` in `build.gradle` exactly |
| `No APK/AAB produced` | Build succeeded but outputs are in an unusual path | Check the build log; adjust the `find` in *Collect outputs* |
| First build is slow (~8 min) | Cold cache | Normal; later builds are ~3–4 min |

To debug, open the failed run, expand the red step, and read the **first** error, not the last line.

---

## How the workflow works internally

Step by step in `android-build.yml`:

1. **Checkout** the app repo.
2. **Detect project type**: `pubspec.yaml` means Flutter, `gradlew` means native. Sets the Gradle root.
3. **Set up Java** (Temurin) with the **Gradle cache**.
4. **Set up Flutter** (Flutter only) with the **pub/SDK cache**.
5. **Configure signing** (release + secrets present):
   - Decodes the keystore into `$RUNNER_TEMP`, **outside the repo**, so it can never be committed or uploaded.
   - Appends `android.injected.signing.*` properties to `gradle.properties`.
     These are the Android Gradle Plugin's built-in hooks (the same ones Android Studio's
     *Generate Signed APK* uses), so **no `build.gradle` or `key.properties` changes are needed** in any app.
6. **Build**:
   - Flutter: `flutter pub get` → `flutter build apk` (+ `appbundle` if enabled)
   - Native: `./gradlew assemble<Flavor><BuildType>` (+ `bundle…` if enabled)
7. **Collect outputs**: finds every `.apk` / `.aab` under `build/outputs` into one folder; fails if none are found.
8. **Upload artifact**: named `<repo>-<build-type>-<run#>`.
9. **GitHub Release**: only on `refs/tags/*` with `create-release: true`, with auto-generated release notes.

### Design decisions

| Decision | Why |
|---|---|
| Reusable workflow instead of copy-paste | One place to fix and upgrade for every app |
| Callers pin `@v1` | A bad change here can't break every client build at once |
| Injected signing instead of `key.properties` | Works on any project with zero code changes |
| Keystore in `$RUNNER_TEMP` | It can't leak into artifacts or commits |
| `build-number` not forced by default | Avoids a versionCode lower than what's already on Play |
| `concurrency` in the caller | Saves CI minutes; only the latest push builds |

### Adding a new input later

1. Add it under `on.workflow_call.inputs` with a **default**, so existing callers don't break.
2. Use it as `${{ inputs.<name> }}` in a step.
3. Document it in the [Inputs reference](#inputs-reference).
4. Move the `v1` tag. Removing or renaming an input is a breaking change and means `v2`.