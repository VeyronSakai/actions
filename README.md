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
| `unity/batch-mode` | Runs the Unity editor CLI in batch mode, resolving the editor path from the project version. | `unity-version`, `log-path` |

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

The `unity/*` actions assume a self-hosted macOS (or Windows) runner where Unity Hub is already installed and the editor is licensed. They resolve the editor executable from the project version under the Unity Hub editor root.

`unity/batch-mode` is self-contained: it resolves the Unity version from `ProjectVersion.txt` and the editor path on its own, so it does not need `unity/project-version` first. By default the editor is looked up under the standard Unity Hub install location; pass `unity-editor-path` to point at a specific executable. First-class inputs cover only the general-purpose flags (`execute-method`, `no-graphics`, `quit`, `silent-crashes`); command-specific flags go through `additional-args` (e.g. `-buildTarget Android`, `-runTests`, or a development build via `-developmentBuild true`), given as a space-separated string (each whitespace-delimited token becomes one argument, so a value cannot itself contain spaces).

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
