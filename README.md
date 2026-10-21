# repo-config

Workflows, an action and Renovate presets shared by the repositories of yuyuyuyuyu-dev.

## Table of contents

- [What it provides](#what-it-provides)
- [Workflows](#workflows)
  - [Oxfmt](#oxfmt)
  - [Kotlin lint](#kotlin-lint)
  - [Deploy to GitHub Pages](#deploy-to-github-pages)
  - [Update the Wasm yarn.lock](#update-the-wasm-yarnlock)
- [Actions](#actions)
  - [Set up the JDK](#set-up-the-jdk)
- [Renovate presets](#renovate-presets)
- [License](#license)

## What it provides

| Name                                                   | For                            | What it does                                                                          |
| ------------------------------------------------------ | ------------------------------ | ------------------------------------------------------------------------------------- |
| [Oxfmt](#oxfmt)                                        | Every repository               | Checks the formatting with [Oxfmt](https://oxc.rs/docs/guide/usage/formatter).        |
| [Kotlin lint](#kotlin-lint)                            | Kotlin projects                | Runs Detekt and ktlint, and checks the formatting of the version catalog.             |
| [Deploy to GitHub Pages](#deploy-to-github-pages)      | Compose for Web projects       | Builds the app and deploys it to GitHub Pages.                                        |
| [Update the Wasm yarn.lock](#update-the-wasm-yarnlock) | Compose for Web projects       | Regenerates the Wasm yarn.lock, opens a pull request and enables auto-merge on it.    |
| [Set up the JDK](#set-up-the-jdk)                      | Kotlin projects                | Sets up Zulu 25 for a Kotlin project, or Zulu 21 for a Compose Multiplatform project. |
| [Renovate presets](#renovate-presets)                  | Repositories that use Renovate | Groups and auto-merges minor and patch updates.                                       |

## Workflows

Each workflow is a reusable workflow. Call it from a job in your own workflow, which decides when it runs. The workflows that run Gradle choose the JDK with [Set up the JDK](#set-up-the-jdk).

### Oxfmt

Checks that every file is formatted by Oxfmt. The version of Oxfmt is pinned in [package.json](package.json) of this repository, so the repository that calls it needs no package.json. Oxfmt reads `.oxfmtrc.json` of that repository, if it has one.

```yaml
jobs:
  oxfmt:
    name: Oxfmt
    permissions:
      contents: read
    uses: yuyuyuyuyu-dev/repo-config/.github/workflows/oxfmt.yml@main
```

### Kotlin lint

Runs three Gradle tasks as parallel jobs: `detekt` of the [Detekt Gradle plugin](https://detekt.dev/docs/gettingstarted/gradle), `ktlintCheck` of [ktlint-gradle](https://github.com/JLLeitschuh/ktlint-gradle), and `versionCatalogFormat` of [version-catalog-update](https://github.com/littlerobots/version-catalog-update-plugin), which fails when the version catalog is not formatted. The project has to apply these three plugins.

Gradle runs on the JDK that [Set up the JDK](#set-up-the-jdk) chooses from the inputs. A Kotlin project passes nothing and gets Zulu 25:

```yaml
jobs:
  kotlin-lint:
    name: Kotlin lint
    permissions:
      contents: read
    uses: yuyuyuyuyu-dev/repo-config/.github/workflows/kotlin-lint.yml@main
```

A Compose Multiplatform project says so, and gets Zulu 21:

```yaml
with:
  compose-multiplatform: true
```

| Input                   | Default | Description                                                    |
| ----------------------- | ------- | -------------------------------------------------------------- |
| `compose-multiplatform` | `false` | `true` for a Compose Multiplatform project.                    |
| `jdk-distribution`      |         | Passed to [Set up the JDK](#set-up-the-jdk) as `distribution`. |
| `jdk-version`           |         | Passed to [Set up the JDK](#set-up-the-jdk) as `version`.      |
| `working-directory`     | `.`     | The directory that has `gradlew`.                              |

### Deploy to GitHub Pages

Builds the Wasm app with `./gradlew :<app-module>:wasmJsBrowserDistribution` and deploys `<app-module>/build/dist/wasmJs/productionExecutable` to GitHub Pages. Anything the build needs, such as the library definitions of [AboutLibraries](https://github.com/mikepenz/AboutLibraries), has to be produced by that Gradle task itself.

When it is called on `workflow_run`, it builds the commit that the triggering run checked, not the latest one.

```yaml
name: Deploy

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]

permissions: {}

jobs:
  deploy:
    name: Deploy
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push'
    permissions:
      contents: read
      pages: write
      id-token: write
    uses: yuyuyuyuyu-dev/repo-config/.github/workflows/deploy-to-github-pages.yml@main
    with:
      app-module: webApp
```

| Input        | Default | Description                                   |
| ------------ | ------- | --------------------------------------------- |
| `app-module` |         | The name of the Gradle module of the web app. |

The repository has to set the source of GitHub Pages to GitHub Actions in its settings.

### Update the Wasm yarn.lock

Deletes `kotlin-js-store/wasm/yarn.lock` and regenerates it with `kotlinWasmUpgradeYarnLock`. The JS lockfile, `kotlin-js-store/yarn.lock`, is left as it is. Regenerating from scratch picks up npm releases within the version ranges, which upgrading the stored lockfile does not. When anything changed, it opens a pull request as a GitHub App, or updates the one already open, and enables auto-merge with squash on a pull request it opens. When nothing changed, it closes the open pull request.

Gradle runs in a job that has no write access. Only the job after it, which runs no build, gets the token of the app.

Neither Dependabot nor Renovate can update this lockfile, because the Kotlin Gradle Plugin generates it and there is no package.json for them to read.

```yaml
name: Update the Wasm yarn.lock

on:
  schedule:
    - cron: "41 21 * * *"
  workflow_dispatch:

permissions: {}

jobs:
  update-wasm-yarn-lock:
    name: Update the Wasm yarn.lock
    uses: yuyuyuyuyu-dev/repo-config/.github/workflows/update-wasm-yarn-lock.yml@main
    with:
      app-client-id: ${{ vars.PR_BOT_CLIENT_ID }}
    secrets:
      app-private-key: ${{ secrets.PR_BOT_PRIVATE_KEY }}
```

| Input           | Default | Description                                                  |
| --------------- | ------- | ------------------------------------------------------------ |
| `app-client-id` |         | The client ID of the GitHub App that opens the pull request. |

| Secret            | Description                                                    |
| ----------------- | -------------------------------------------------------------- |
| `app-private-key` | The private key of the GitHub App that opens the pull request. |

The app needs read and write access to the contents and the pull requests of the repository. A pull request opened by an app, unlike one opened by `GITHUB_TOKEN`, triggers the other workflows, so the required checks run on it. The repository has to allow auto-merge in its settings.

## Actions

### Set up the JDK

Sets up the JDK to run Gradle on. Use it in the jobs of your own workflows that run Gradle, with the same inputs as the workflows of this repository get, so that every job runs on the same JDK:

```yaml
steps:
  - name: Set up the JDK
    uses: yuyuyuyuyu-dev/repo-config/.github/actions/setup-jdk@main
    with:
      compose-multiplatform: true
```

| `compose-multiplatform` | `distribution` and `version` | JDK                                                                                                 |
| ----------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| `true`                  | not passed                   | Zulu 21, which `gradle/gradle-daemon-jvm.properties` of the Compose Multiplatform template asks for |
| `true`                  | passed                       | Fails, because a Compose Multiplatform project always runs on the JDK of the template               |
| `false` (default)       | not passed                   | Zulu 25                                                                                             |
| `false` (default)       | passed                       | The one passed, as [actions/setup-java](https://github.com/actions/setup-java) names it             |

Passing only one of `distribution` and `version` fails.

## Renovate presets

| Preset                                                    | What it does                                                                                                                                                                                                                                                                                                                                                                     |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `github>yuyuyuyuyu-dev/repo-config`                       | Extends `config:recommended`, groups minor and patch updates of every kind of dependency into one pull request, and auto-merges it once the checks pass. Only one pull request is open at a time, because a pull request must be up to date with main before it merges. Commits start with `build(deps):` and pull requests are labeled `dependencies`, as Dependabot does.      |
| `github>yuyuyuyuyu-dev/repo-config:compose-multiplatform` | Leaves Compose Multiplatform to [update-compose-multiplatform](https://github.com/yuyuyuyuyu-dev/update-compose-multiplatform): the Gradle plugin, the libraries that [the CHANGELOG](https://github.com/JetBrains/compose-multiplatform/blob/master/CHANGELOG.md) lists as the ones to match it, and `org.jetbrains.compose.components:components-*`, which shares its version. |

Extend them from `renovate.json` of your repository:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>yuyuyuyuyu-dev/repo-config",
    "github>yuyuyuyuyu-dev/repo-config:compose-multiplatform"
  ]
}
```

## License

[MIT](LICENSE)
