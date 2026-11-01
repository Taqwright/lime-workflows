# Taqwright LIME workflows

Reusable GitHub Actions workflows for [Taqwright LIME](https://www.taqwright.ai). Your repository
calls them from short files **you own**. LIME improves the workflows here, and you get every fix
by staying on `@v1`, with no pull request and no rewrite of your files.

| Workflow | Runs on | What it does |
|---|---|---|
| [`pr-impact.yml`](.github/workflows/pr-impact.yml) | every pull request | The Taqwright GitHub App posts one comment: impacted screens and tests, tests to update, what is not automated yet, and **suggested tests** |
| [`automate.yml`](.github/workflows/automate.yml) | a `/automate …` comment | Builds the app from the PR head with **your build action**, records the test on a cloud device, and the Taqwright App opens it as a pull request in your **test** repository |

LIME's **Integrations → GitHub → Automation setup** writes these files for you. This page is for
reading them and changing them.

## Your files

### `.github/workflows/lime-pr.yml`

```yaml
name: LIME PR impact
on: pull_request
permissions:
  contents: read
jobs:
  lime:
    uses: Taqwright/lime-workflows/.github/workflows/pr-impact.yml@v1
    with:
      project: My App                                   # an existing LIME project
      runner: ${{ vars.LIME_RUNNER || '["ubuntu-latest"]' }}
    secrets: inherit
```

### `.github/workflows/automate.yml`

```yaml
name: automate
on:
  issue_comment:
    types: [created]
permissions:
  contents: read
  pull-requests: read
jobs:
  automate:
    uses: Taqwright/lime-workflows/.github/workflows/automate.yml@v1
    with:
      project: My App
      platforms: android                                # android, ios, or android,ios
      cloud-provider: lambdatest                        # lambdatest (TestMu AI) or browserstack
      runner-android: ${{ vars.LIME_RUNNER || '["ubuntu-latest"]' }}
      device-android: Pixel 8
      os-android: '14'
      artifact-android: app/build/outputs/apk/debug/app-debug.apk
    secrets: inherit
```

`/automate 2` records suggestion 2 from the impact comment, and `/automate <goal>` records your own
goal. With `platforms: android,ios`, append the platform (`/automate 2 ios`); the first one listed
is the default.

### Your build action

GitHub does not let `uses:` take an input, so the build lives at a **fixed path**, checked out from
the PR head:

- `.github/actions/lime-build-android/action.yml`, `.github/actions/lime-build-ios/action.yml`, or
- `.github/actions/lime-build/action.yml`, used when there is no per-platform one.

It is a normal [composite action](https://docs.github.com/actions/sharing-automations/creating-actions/creating-a-composite-action).
It can contain **any steps**: set-up actions, signing, caching, extra checks. Its one job is to leave
an installable file at the `artifact-<platform>` path. Starting points:

<details><summary><b>Native Android (Gradle)</b></summary>

```yaml
name: lime-build-android
runs:
  using: composite
  steps:
    - uses: actions/setup-java@v4
      with: { distribution: temurin, java-version: '17' }
    - run: ./gradlew assembleDebug
      shell: bash
# artifact-android: app/build/outputs/apk/debug/app-debug.apk
```
</details>

<details><summary><b>Flutter (Android)</b></summary>

```yaml
name: lime-build-android
runs:
  using: composite
  steps:
    - uses: actions/setup-java@v4
      with: { distribution: temurin, java-version: '17' }
    - uses: subosito/flutter-action@v2
      with: { channel: stable, cache: true }      # pin flutter-version: to match your app
    - run: |
        flutter pub get
        flutter build apk --debug
      shell: bash
# artifact-android: build/app/outputs/flutter-apk/app-debug.apk
```
</details>

<details><summary><b>React Native (Android)</b></summary>

```yaml
name: lime-build-android
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with: { node-version: '20' }
    - uses: actions/setup-java@v4
      with: { distribution: temurin, java-version: '17' }
    - run: |
        npm ci
        cd android && ./gradlew assembleDebug
      shell: bash
# artifact-android: android/app/build/outputs/apk/debug/app-debug.apk
```
</details>

<details><summary><b>Native iOS (Xcode), which needs a Mac and signing</b></summary>

Real cloud iPhones install only a **signed** `.ipa`. Add your distribution certificate and
provisioning profile as repository secrets and pass them to the steps that import them, or use
Fastlane (`match`). The runner must be a Mac: set `runner-ios`.

```yaml
name: lime-build-ios
runs:
  using: composite
  steps:
    # EDIT ME: import your signing certificate and provisioning profile here
    # (for example apple-actions/import-codesign-certs, or `fastlane match`).
    - run: |
        pod install --project-directory=ios || true
        xcodebuild -workspace ios/App.xcworkspace -scheme App -configuration Release \
          -archivePath build/App.xcarchive archive
        xcodebuild -exportArchive -archivePath build/App.xcarchive \
          -exportOptionsPlist ios/ExportOptions.plist -exportPath build/ipa
      shell: bash
# artifact-ios: build/ipa/App.ipa
```
</details>

<details><summary><b>Flutter (iOS), which needs a Mac and signing</b></summary>

```yaml
name: lime-build-ios
runs:
  using: composite
  steps:
    - uses: subosito/flutter-action@v2
      with: { channel: stable, cache: true }
    # EDIT ME: import signing (certificate + provisioning profile) before the build.
    - run: |
        flutter pub get
        flutter build ipa --export-options-plist=ios/ExportOptions.plist
      shell: bash
# artifact-ios: build/ios/ipa/<YourApp>.ipa
```
</details>

## Inputs

### `automate.yml`

| Input | Default | |
|---|---|---|
| `project` | (required) | An existing LIME project. Its bound **test** repository receives the test |
| `platforms` | `android` | `android`, `ios`, or `android,ios`. The first is the default for `/automate` |
| `cloud-provider` | `lambdatest` | `lambdatest` (TestMu AI) or `browserstack` |
| `runner-android` / `runner-ios` | `["ubuntu-latest"]` / `["macos-latest"]` | JSON list of runner labels: your own machine, e.g. `["self-hosted","macOS","ARM64"]` |
| `device-android` / `device-ios` | `Pixel 8` / `iPhone 15` | Exact provider device name |
| `os-android` / `os-ios` | `14` / `17` | Exact provider OS version |
| `artifact-android` / `artifact-ios` | Gradle's debug APK / (none) | Where your build action leaves the installable file |
| `lime-cli-version` | `latest` | Pin `@taqwright/lime-cli` if you need to |
| `timeout-minutes` | `45` | For the build-and-record job |

### `pr-impact.yml`

| Input | Default | |
|---|---|---|
| `project` | (required) | An existing LIME project |
| `runner` | `["ubuntu-latest"]` | JSON list of runner labels |
| `app-dir` | `.` | Where the app's source lives |
| `lime-cli-version` | `latest` | |

## Secrets

Pass them with `secrets: inherit`, or name them one by one.

| Secret | Needed by |
|---|---|
| `LIME_CI_TOKEN` | both, a CI token from LIME (Settings → Access) |
| `LT_USERNAME`, `LT_ACCESS_KEY` | `automate.yml` with TestMu AI |
| `BROWSERSTACK_USERNAME`, `BROWSERSTACK_ACCESS_KEY` | `automate.yml` with BrowserStack |

Your app repository needs **no credential for the test repository**: LIME hands the recording to
the test repository's own workflow, and the Taqwright GitHub App opens the pull request there.

## Self-hosted runners

Install once on the machine: **git**, **Node.js 20**, **gh** and **jq**, plus your build's own
toolchain (Android SDK with `ANDROID_HOME`, or Xcode, CocoaPods and signing for iOS). A runner's
PATH is captured when it is configured, so after installing anything run
`echo "$PATH" > .path && ./svc.sh stop && ./svc.sh start` in the runner's folder.

## Versions

`@v1` always points at the latest `v1.x.y`. Fixes and improvements arrive there with no change to
your files. A breaking change becomes `@v2`, and LIME's Automation setup offers it as a one-line
pull request. Pin `@v1.2.3` if you want to choose when you move.
