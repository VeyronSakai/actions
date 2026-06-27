# actions

Shared GitHub Actions for VeyronSakai repositories.

This repository hosts reusable composite actions that are shared across multiple repositories.

## Actions

| Path | Description | Outputs |
| --- | --- | --- |
| `git/config` | Reads `.lfsconfig` from the calling repository with `gh api` and extracts the Git LFS endpoint URL. | `lfs-url` |
| `git/checkout` | Runs `actions/checkout@v6` and can optionally read `.lfsconfig` to inject a custom Git LFS endpoint first. | `lfs-url` |
| `unity/project-version` | Reads `m_EditorVersion` from `ProjectSettings/ProjectVersion.txt`. | `unity-version` |
| `unity/product-name` | Reads `productName` from `ProjectSettings/ProjectSettings.asset`. | `product-name` |
| `unity/batch-mode` | Runs the Unity editor CLI in batch mode, resolving the editor path from the project version. | `unity-version`, `editor-path`, `log-path` |

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

The `unity/*` actions assume a self-hosted macOS (or Windows) runner where Unity Hub is already installed and the editor is licensed. They resolve the editor executable from the project version under the Unity Hub editor root.

`unity/batch-mode` is self-contained: it resolves the Unity version (from `ProjectVersion.txt` when `unity-version` is empty) and the editor path on its own, so it does not need `unity/project-version` first. First-class inputs cover the common flags (`execute-method`, `build-target`, `run-tests`, `no-graphics`, `quit`, `silent-crashes`, `force-development-build`); anything else goes through `additional-args`, one argument per line so values with spaces or secrets are never re-split by the shell.

Run a static method (script compile check):

```yaml
- uses: VeyronSakai/actions/unity/batch-mode@<ref>
  with:
    execute-method: Editor.ScriptCompileChecker.EntryPoint.Check
    build-target: Android
```

Run tests (set `quit: "false"` so the editor stays alive for `-runTests`):

```yaml
- uses: VeyronSakai/actions/unity/batch-mode@<ref>
  with:
    run-tests: "true"
    quit: "false"
    additional-args: |
      -testResults
      UnityTestResults.xml
```

Build a player (extra flags and secrets via `additional-args`):

```yaml
- uses: VeyronSakai/actions/unity/project-version@<ref>
  id: unity-version

- uses: VeyronSakai/actions/unity/batch-mode@<ref>
  with:
    unity-version: ${{ steps.unity-version.outputs.unity-version }}
    build-target: Android
    execute-method: VeUnityBuild.Editor.Presentations.BatchEntryPoint.Build
    additional-args: |
      -buildMode
      release
      -buildConfig
      Assets/LocalAssets/Settings/AndroidDevBuildConfig.asset

- uses: VeyronSakai/actions/unity/product-name@<ref>
  id: product-name
```
