# repo-config

Workflows, an action and Renovate presets shared by the repositories of yuyuyuyuyu-dev.

## Table of contents

- [What it provides](#what-it-provides)
- [Workflows](#workflows)
  - [Oxfmt](#oxfmt)
  - [Compose Multiplatform lint](#compose-multiplatform-lint)
  - [Deploy to GitHub Pages](#deploy-to-github-pages)
  - [Update yarn.lock](#update-yarnlock)
- [Actions](#actions)
  - [Set up the JDK](#set-up-the-jdk)
- [Renovate presets](#renovate-presets)
- [License](#license)

## What it provides

| Name                                                      | For                            | What it does                                                                   |
| --------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------ |
| [Oxfmt](#oxfmt)                                           | Every repository               | Checks the formatting with [Oxfmt](https://oxc.rs/docs/guide/usage/formatter). |
| [Compose Multiplatform lint](#compose-multiplatform-lint) | Compose Multiplatform projects | Runs Detekt and ktlint, and checks the formatting of the version catalog.      |
| [Deploy to GitHub Pages](#deploy-to-github-pages)         | Compose for Web projects       | Builds the app and deploys it to GitHub Pages.                                 |
| [Update yarn.lock](#update-yarnlock)                      | Compose for Web projects       | Regenerates yarn.lock, opens a pull request and enables auto-merge on it.      |
| [Set up the JDK](#set-up-the-jdk)                         | Compose Multiplatform projects | Sets up the JDK that the Compose Multiplatform template runs Gradle on.        |
| [Renovate presets](#renovate-presets)                     | Repositories that use Renovate | Groups and auto-merges minor and patch updates.                                |

## Workflows

Each workflow is a reusable workflow. Call it from a job in your own workflow, which decides when it runs. The workflows that run Gradle set it up on the JDK of [Set up the JDK](#set-up-the-jdk).

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

### Compose Multiplatform lint

Runs three jobs in parallel: Detekt, ktlint, and `versionCatalogFormat` of [version-catalog-update](https://github.com/littlerobots/version-catalog-update-plugin), which fails when the version catalog is not formatted.

```yaml
jobs:
  lint:
    name: Lint
    permissions:
      contents: read
    uses: yuyuyuyuyu-dev/repo-config/.github/workflows/compose-multiplatform-lint.yml@main
```

| Input         | Default       | Description                       |
| ------------- | ------------- | --------------------------------- |
| `detekt-task` | `detekt`      | The Gradle task that runs Detekt. |
| `ktlint-task` | `ktlintCheck` | The Gradle task that runs ktlint. |

### Deploy to GitHub Pages

Builds the app with Gradle and deploys the built directory to GitHub Pages. When it is called on `workflow_run`, it builds the commit that the triggering run checked, not the latest one.

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
      gradle-arguments: |
        :composeApp:exportLibraryDefinitions -PaboutLibraries.exportPath=src/commonMain/composeResources/files/
        :composeApp:wasmJsBrowserDistribution
      distribution-directory: composeApp/build/dist/wasmJs/productionExecutable
```

| Input                    | Default | Description                                                              |
| ------------------------ | ------- | ------------------------------------------------------------------------ |
| `gradle-arguments`       |         | The arguments to `./gradlew`. Each line runs `./gradlew` once, in order. |
| `distribution-directory` |         | The directory to deploy.                                                 |

The repository has to set the source of GitHub Pages to GitHub Actions in its settings.

### Update yarn.lock

Deletes `kotlin-js-store/yarn.lock` and `kotlin-js-store/wasm/yarn.lock`, whichever exists, and regenerates them with `kotlinUpgradeYarnLock` and `kotlinWasmUpgradeYarnLock`. Regenerating from scratch picks up npm releases within the version ranges, which upgrading the stored lockfile does not. When anything changed, it opens a pull request as a GitHub App, or updates the one already open, and enables auto-merge with squash on a pull request it opens. When nothing changed, it closes the open pull request.

Gradle runs in a job that has no write access. Only the job after it, which runs no build, gets the token of the app.

Neither Dependabot nor Renovate can update these lockfiles, because the Kotlin Gradle Plugin generates them and there is no package.json for them to read.

```yaml
name: Update yarn.lock

on:
  schedule:
    - cron: "41 21 * * *"
  workflow_dispatch:

permissions: {}

jobs:
  update-yarn-lock:
    name: Update yarn.lock
    uses: yuyuyuyuyu-dev/repo-config/.github/workflows/update-yarn-lock.yml@main
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

Sets up Zulu 21, the JDK that `gradle/gradle-daemon-jvm.properties` of the Compose Multiplatform template asks for. Use it in the jobs of your own workflows that run Gradle:

```yaml
steps:
  - name: Set up the JDK
    uses: yuyuyuyuyu-dev/repo-config/.github/actions/setup-jdk@main
```

## Renovate presets

| Preset                                                    | What it does                                                                                                                                                                                                                     |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `github>yuyuyuyuyu-dev/repo-config`                       | Extends `config:recommended`, groups minor and patch updates into one pull request, and auto-merges it once the checks pass. Commits start with `build(deps):` and pull requests are labeled `dependencies`, as Dependabot does. |
| `github>yuyuyuyuyu-dev/repo-config:compose-multiplatform` | Leaves Compose Multiplatform to [update-compose-multiplatform](https://github.com/yuyuyuyuyu-dev/update-compose-multiplatform), which updates it together with the libraries that have to match it.                              |

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
