# actions

Shared GitHub Actions for VeyronSakai repositories.

This repository hosts reusable composite actions that are shared across multiple repositories.

## Actions

| Path | Description | Outputs |
| --- | --- | --- |
| `git/config` | Reads `.lfsconfig` from the calling repository with `gh api` and extracts the Git LFS endpoint URL. | `lfs-url` |
| `git/checkout` | Runs `actions/checkout@v6` and can optionally read `.lfsconfig` to inject a custom Git LFS endpoint first. | `lfs-url` |
| `unity/editor-version` | Reads `m_EditorVersion` from `ProjectSettings/ProjectVersion.txt`. | `unity-version` |
| `unity/product-name` | Reads `productName` from `ProjectSettings/ProjectSettings.asset`. | `product-name` |
| `unity/batch-mode` | Runs the Unity editor CLI in batch mode, resolving the editor path from the project's editor version. | `unity-version`, `log-path` |
| `firebase/distribute-app` | Uploads an app binary to Firebase App Distribution using the standalone Firebase CLI. | — |

## Versioning

Releases are managed with [release-drafter](https://github.com/release-drafter/release-drafter). Each PR is categorized by label (`breaking changes` / `enhancement` / `bug`) and accumulated into a draft release by the **Release** workflow. Running that workflow manually (`workflow_dispatch`) publishes the current draft and moves the major (`vX`) and minor (`vX.Y`) tags to the released commit.

Pin actions to a moving tag rather than a branch:

```yaml
- uses: VeyronSakai/actions/unity/batch-mode@v0      # latest within the major line
- uses: VeyronSakai/actions/unity/batch-mode@v0.1    # latest within the minor line
```

Throughout this README, `@<ref>` stands for such a tag (or a commit SHA when you need to pin exactly).

## Usage

```yaml
- uses: VeyronSakai/actions/git/checkout@<ref>
  with:
    lfs: "false"
```

`git/checkout` always runs `git clean -df` and `git reset --hard HEAD` after checkout. `clean` is also passed through to `actions/checkout` itself.

Enable `lfs: "true"` when the calling repository needs `actions/checkout` to respect a custom Git LFS endpoint from `.lfsconfig`. In that case, also pass `github-token`.

```yaml
- uses: VeyronSakai/actions/git/checkout@<ref>
  with:
    github-token: ${{ github.token }}
    lfs: "true"
```

### Unity actions

The `unity/*` actions assume a self-hosted macOS (or Windows) runner where Unity Hub is already installed and the editor is licensed. They resolve the editor executable from the project's editor version under the Unity Hub editor root.

`unity/batch-mode` is self-contained: it resolves the Unity version from `ProjectVersion.txt` and the editor path on its own, so it does not need `unity/editor-version` first. By default the editor is looked up under the standard Unity Hub install location; pass `unity-editor-path` to point at a specific executable. First-class inputs cover only the general-purpose flags (`execute-method`, `no-graphics`, `quit`, `silent-crashes`); command-specific flags go through `additional-args` (e.g. `-buildTarget Android`, `-runTests`, or a development build via `-developmentBuild true`), given as a space-separated string (each whitespace-delimited token becomes one argument, so a value cannot itself contain spaces).

Run a static method (script compile check):

```yaml
- uses: VeyronSakai/actions/unity/batch-mode@<ref>
  with:
    execute-method: Editor.ScriptCompileChecker.EntryPoint.Check
    additional-args: -buildTarget Android
```

Run tests (set `quit: "false"` so the editor stays alive for `-runTests`):

```yaml
- uses: VeyronSakai/actions/unity/batch-mode@<ref>
  with:
    quit: "false"
    additional-args: -runTests -testResults UnityTestResults.xml
```

Build a player (extra flags via `additional-args`):

```yaml
- uses: VeyronSakai/actions/unity/batch-mode@<ref>
  with:
    execute-method: VeUnityBuild.Editor.Presentations.BatchEntryPoint.Build
    additional-args: -buildTarget Android -buildMode release -buildConfig Assets/LocalAssets/Settings/AndroidDevBuildConfig.asset

- uses: VeyronSakai/actions/unity/product-name@<ref>
  id: product-name
```

### Firebase App Distribution

`firebase/distribute-app` downloads the standalone Firebase CLI (no Node required), caches it under
`RUNNER_TOOL_CACHE`, and uploads a binary. Authentication goes through Workload Identity Federation:
the action runs [`google-github-actions/auth`](https://github.com/google-github-actions/auth) to exchange
the job's OIDC token for short-lived credentials of a service account holding the
**Firebase App Distribution Admin** role (`roles/firebaseappdistro.admin`) on the app's project. No service
account key is involved.

The calling job must grant `id-token: write`, otherwise no OIDC token is issued and the exchange fails.

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: VeyronSakai/actions/firebase/distribute-app@<ref>
    with:
      binary-path: path/to/App.ipa
      app-id: ${{ vars.FIREBASE_IOS_APP_ID }}
      workload-identity-provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      service-account: ${{ vars.GCP_SERVICE_ACCOUNT }}
      testers: someone@example.com
```

`workload-identity-provider` is the full resource name
(`projects/<number>/locations/global/workloadIdentityPools/<pool>/providers/<provider>`). Restrict the
provider with an `--attribute-condition` on the repository owner and grant
`roles/iam.workloadIdentityUser` to the matching principal set; otherwise any GitHub repository can
authenticate through it.

`testers` (comma-separated emails) and `groups` (comma-separated group aliases) may be combined. Addresses
passed to `testers` that are not yet registered are added to the project and invited automatically, so there
is no need to create testers or groups up front.

**Give at least one of them.** With neither, the CLI still exits 0: the release is created but distributed to
nobody, which looks like a successful build until someone notices no email arrived. The action emits a warning
annotation in that case.

Pin `firebase-tools-version` (default `latest`) when a reproducible CLI version matters. The cache is keyed by
that value, so `latest` is fetched once per runner and then reused. Pinning is recommended here: the CLI's
Application Default Credentials handling has regressed before
([firebase-tools#10716](https://github.com/firebase/firebase-tools/issues/10716)), and with `latest` such a
release reaches the runners unreviewed.
