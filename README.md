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
    secrets:
      LIME_CI_TOKEN: ${{ secrets.LIME_CI_TOKEN }}
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
    secrets:                                            # only what it needs, never `inherit`
      LIME_CI_TOKEN: ${{ secrets.LIME_CI_TOKEN }}
      LT_USERNAME: ${{ secrets.LT_USERNAME }}           # or BROWSERSTACK_USERNAME / _ACCESS_KEY
      LT_ACCESS_KEY: ${{ secrets.LT_ACCESS_KEY }}
```

`/automate 2` records suggestion 2 from the impact comment, and `/automate <goal>` records your own
goal. With `platforms: android,ios`, append the platform (`/automate 2 ios`); the first one listed
is the default.

**Several tests in one comment, up to 5.** Name several suggestions with commas, or put one goal
per line under a bare `/automate`. Numbers and goals can mix, and list markers are optional:

```
/automate 1,3
```
```
/automate
- log in with demo@example.com and verify Home shows "Welcome"
- add a product to the cart and verify the badge shows 1
2
```

Commas separate **numbers** only, because a typed goal keeps its own commas. LIME reads the
command and refuses one it will not run, before anything is built or a device is booked, with a
reply that says why: more than 5 tests (it suggests how to split them), a suggestion number that is
gone, one behind a feature flag, or a line that is not a test goal. The tests are recorded one after
another, each waiting for the test repository's workflow before the next, and they **all land in
one pull request** in your test repository (test-repo template v28 or later). Allow about 15
minutes per test. The job's timeout grows to fit, with `timeout-minutes` as the floor.

With test-repo template v30 or later, that pull request **opens once, after the last test**, and
its run starts on its own. Before v30 it opened after the first test, and each later test was a
push onto it with the test repository's `GITHUB_TOKEN`. GitHub holds a run raised that way until
someone with write access clicks *Approve workflows to run*. A second `/automate` later, on a pull
request whose test pull request is already open, still adds to it that way, so that run still
needs approval.

### Your build action

GitHub does not let `uses:` take an input, so the build lives at a **fixed path**. The copy that
runs is the one on your **default branch**, reviewed and merged, so a pull request cannot change
the build steps that run with your secrets. A pull request's own copy is used only when the
default branch has none (the pull request that first adds it):

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
| `allow-fork-prs` | `false` | Run on pull requests from forks. Leave it off; label one PR `lime-allow-fork` instead |
| `timeout-minutes` | `45` | For the build-and-record job. The floor: a command with several tests gets 30 + 15 per test when that is more |

### `pr-impact.yml`

| Input | Default | |
|---|---|---|
| `project` | (required) | An existing LIME project |
| `runner` | `["ubuntu-latest"]` | JSON list of runner labels |
| `app-dir` | `.` | Where the app's source lives |
| `lime-cli-version` | `latest` | |

## Secrets

**Name them one by one**, as in the callers above. `secrets: inherit` works, but it hands this
workflow **every** secret in your repository, when it needs only these:

| Secret | Needed by |
|---|---|
| `LIME_CI_TOKEN` | both, a CI token from LIME (Settings → Access) |
| `LT_USERNAME`, `LT_ACCESS_KEY` | `automate.yml` with TestMu AI |
| `BROWSERSTACK_USERNAME`, `BROWSERSTACK_ACCESS_KEY` | `automate.yml` with BrowserStack |

Your app repository needs **no credential for the test repository**: LIME hands the recording to
the test repository's own workflow, and the Taqwright GitHub App opens the pull request there.

## Security

- **Pin what you trust.** Your repository runs this workflow's code with the secrets you pass it.
  `@v1` moves when LIME ships a fix. If you need to choose exactly what runs, pin a full commit
  SHA (`…/automate.yml@<40-char sha>`) and move it when you have reviewed the change.
- **Pass only the secrets it needs**, as above, never `inherit`.
- **Pull requests from forks are refused by default.** `/automate` builds and runs the pull
  request's own code, build action included, with your secrets. From a fork, that is a stranger's
  code. Only owners, members and collaborators can trigger it at all. On a fork's pull request a
  maintainer must first add the label `lime-allow-fork` (after reading the change), or the caller
  must set `allow-fork-prs: true`.
- **Never attach a self-hosted runner to a public repository.** Anyone who can open a pull
  request there can run code on it. Your runner belongs to your own, private repositories; this
  repository has none.
- **Nothing here holds a secret**, and calling it gives this repository no access to yours: the
  workflow runs in your repository, on your runners.

## Self-hosted runners

GitHub's machines come with a clean, known toolset. Your own machine comes with whatever is
already on it, and that is where first runs fail: a tool that is missing, too new or too old, or
a setting meant for the person who uses the machine. Work through this list once per machine.

### Checklist

**Before you start**

- [ ] The Taqwright GitHub App is installed on the **app** repository and the **test** repository.
- [ ] LIME has a project with the test repository bound to it (LIME → Integrations → GitHub →
      Automation setup).
- [ ] You have a LIME CI token (Settings → Access), and your cloud provider's username and access key.

**The machine**

- [ ] The runner is registered on **private** repositories only (see [Security](#security)), and
      runs as a service: `./svc.sh install && ./svc.sh start`. A dedicated user is best: anything
      in that user's home directory is visible to the jobs.
- [ ] Installed: **git**, **Node.js 20**, **gh** and **jq**. Check with
      `git --version && node --version && gh --version && jq --version`.
- [ ] Android: the Android SDK, with `ANDROID_HOME` set. **Java comes from your build action**
      (`actions/setup-java`), not the machine: a machine's default JDK is often newer than your
      Gradle supports.
- [ ] iOS: Xcode and its command line tools, CocoaPods, and your signing certificate and
      provisioning profile, or `fastlane match`.
- [ ] **No global lime-cli.** Run `npm ls -g @taqwright/lime-cli`, and if it is there,
      `npm uninstall -g @taqwright/lime-cli`. The workflows run `@taqwright/lime-cli@latest`.
      An old global copy has answered in its place before.
- [ ] **Refresh the runner's PATH after installing anything.** The runner saves PATH when it is
      configured and never reads it again. In the runner's folder:
      `echo "$PATH" > .path && ./svc.sh stop && ./svc.sh start`.

**The repositories**

- [ ] The repository variable `LIME_RUNNER` is set to the runner's labels as a JSON list, for
      example `["self-hosted","macOS","ARM64"]`, on **both** repositories. For iOS on a different
      machine, also set `LIME_RUNNER_IOS` on the app repository.
- [ ] Secrets on the **app** repository: `LIME_CI_TOKEN`, plus `LT_USERNAME` and `LT_ACCESS_KEY`
      (or `BROWSERSTACK_USERNAME` and `BROWSERSTACK_ACCESS_KEY`).
- [ ] Secrets on the **test** repository: `LIME_CI_TOKEN` and `ANTHROPIC_API_KEY` (or your coding
      agent's key).
- [ ] Automation setup shows every row green: both app workflows say *Uses lime-workflows @v1*,
      the build action says *Yours*, and the test workflow is current. Merge LIME's pull requests
      first: the build action is taken from the **default branch**.
- [ ] Your build action builds **your** app: the Java version, the Flutter version, flavours,
      signing. `artifact-<platform>` in `automate.yml` must name the file it leaves.

**The first run**

- [ ] Open a small pull request on the app repository. The impact comment appears within a few
      minutes, posted by the Taqwright App.
- [ ] Comment `/automate 1` on it. The Taqwright App reacts with 👀 and says what it is
      automating, and in about ten minutes it replies with a link to a test pull request in the
      test repository.

### When a first run fails

| What you see | Why | Fix |
|---|---|---|
| The job waits for a runner and never starts | `LIME_RUNNER`'s labels do not match the runner's, or the runner is offline | Compare the labels with Settings → Actions → Runners |
| `gh: command not found` (or `jq`, `node`) right after installing it | The runner still has its old PATH | Refresh it (above) |
| `Unsupported class file major version` in the build | The machine's default JDK is newer than your Gradle supports | Set `java-version` with `actions/setup-java` in your build action |
| `--goal is required` when commenting `/automate 2` | An old global lime-cli answered instead of `@latest` | `npm uninstall -g @taqwright/lime-cli` |
| `No build action` | The build action is not on the default branch yet | Merge LIME's setup pull request |
| `… does not exist` after the build | `artifact-<platform>` does not name the file your build leaves | Fix the path in `automate.yml`, or the build |
| The test repository's job stalls on its coding-agent step | The agent picked up the machine user's own settings or login | Keep `taqwright-automate.yml` current: it runs the agent with a config folder of its own |
| `401` from LIME | A secret is missing or holds the wrong token | Re-create `LIME_CI_TOKEN` from Settings → Access |

## Versions

`@v1` always points at the latest `v1.x.y`. Fixes and improvements arrive there with no change to
your files. A breaking change becomes `@v2`, and LIME's Automation setup offers it as a one-line
pull request. Pin `@v1.2.3` if you want to choose when you move.
