<h1 align="center">Unreal AI GAS</h1>

<p align="center">
  A <b>Gameplay Ability System (GAS)</b> tool extension for
  <a href="https://github.com/IvanMurzak/Unreal-MCP">AI Game Developer (Unreal-MCP)</a>.
  Lets an AI agent inspect Gameplay Abilities and Attribute Sets, add an Ability System Component to
  actors, grant abilities, and read an actor's ability/attribute state — all from inside the Unreal
  Editor.
</p>

---

**Unreal AI GAS** is an Unreal Engine **`Type=Editor` plugin** that implements the Unreal-MCP
contract `IUnrealMcpToolProvider` and contributes a focused family of MCP tools wrapping Unreal's
**GameplayAbilities** (Gameplay Ability System) plugin. Unreal-MCP discovers the provider at boot
(and live, when the plugin loads later) and merges these tools into the advertised set, so an AI
agent can drive ability-system workflows against the live editor. Enabling / disabling the extension
live-updates what the AI sees.

> Authoring is **C++** (unlike Unity's C# `[McpPluginTool]`). The extension takes a compile-time
> dependency on the engine's `GameplayAbilities` plugin — that dependency **is the gating**: the
> extension won't compile or load unless GAS is present in the host project.

## Status

This repository is a freshly-scaffolded **skeleton**: the gating against the `GameplayAbilities`
plugin is wired and the CI is live, but the GAS tools are **not implemented yet** — the provider
currently registers only the sample `hello-extension` tool the template ships, which the
implementation step replaces with the tools below.

## Tools (planned)

This extension will contribute the following GAS tools (ids are kebab-case, prefixed `gas-`; handlers
run on the game thread and call GAS / editor APIs directly). Mutating tools validate engine state
defensively and return a structured error rather than crashing the editor.

| Tool | Kind | What it does |
| --- | --- | --- |
| `gas-list-abilities` | read-only | List the `UGameplayAbility` classes (blueprint + native) available in the project. |
| `gas-list-attribute-sets` | read-only | List the `UAttributeSet` classes available in the project. |
| `gas-add-ability-system` | mutating | Add a `UAbilitySystemComponent` to a named actor in the editor world. |
| `gas-grant-ability` | mutating | Grant a `UGameplayAbility` to a named actor's ability system. |
| `gas-get-ability-system` | read-only | Inspect an actor's ability system: granted abilities + current attribute values. |

> The exact tool set is finalized during implementation; each tool ships with one UE Automation spec
> and one E2E `unreal-mcp-cli` check. `extension.json` `tools[]` and this table are the source of truth.

## Install

Install into any UE project that has the **UnrealMCP core plugin** available (the project path is a
**positional** argument):

```bash
# From the published GitHub Release:
unreal-mcp-cli install-extension com.ivanmurzak.unreal-ai-gas <UEProject>

# Offline / from a local checkout (no published release needed):
unreal-mcp-cli install-extension com.ivanmurzak.unreal-ai-gas <UEProject> --source <path-to-this-repo>/UnrealAIGAS
```

The CLI resolves the release source zip (`releases/download/v<version>/UnrealAIGAS-<version>.zip`),
drops the plugin into `<UEProject>/Plugins/UnrealAIGAS/`, enables it **and** the gating
`GameplayAbilities` engine plugin in the `.uproject`, and the editor compiles it from source on next
open (or pass `--build` to compile now via UBT). The same capability backs the AI-Game-Dev desktop
app button and the in-editor Extensions panel.

## Layout

```
UnrealAIGAS/                                  the UE plugin
├── UnrealAIGAS.uplugin                        descriptor; Type=Editor; Plugins: [ UnrealMCP, GameplayAbilities ]
└── Source/UnrealAIGAS/
    ├── UnrealAIGAS.Build.cs                   deps: UnrealMcpRuntime + UnrealMcpEditor + GameplayAbilities(+Editor)
    └── Private/
        ├── UnrealAIGASModule.cpp              the IUnrealMcpToolProvider + module; registers the tools
        └── Tests/UnrealAIGASSpec.cpp          UE Automation specs (one It(...) per tool)
commands/                                      bump-version / get-version / update-core / init
Tests/e2e/                                     E2E unreal-mcp-cli tool checks (one per tool)
extension.json                                 install-catalog / compatibility manifest
.github/workflows/                             CI: test_pull_request + release (+ reusable test_unreal_plugin)
```

## Develop locally

The fastest loop is a directory junction into the UE 5.7 testbed (which already has `Plugins/UnrealMCP`):

```powershell
# Junction this plugin into a UE C++ project that has Plugins/UnrealMCP available:
cmd /c mklink /J "<UEProject>\Plugins\UnrealAIGAS" "<thisRepo>\UnrealAIGAS"

# Build the editor target with UBT:
& "C:\Program Files\Epic Games\UE_5.7\Engine\Build\BatchFiles\Build.bat" `
  <UEProject>Editor Win64 Development -project="<UEProject>\<UEProject>.uproject" -WaitMutex

# Run this extension's Automation specs (filter = the module name):
& "C:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" `
  "<UEProject>\<UEProject>.uproject" -nullrhi -nosplash -unattended `
  -ExecCmds="Automation RunTests UnrealAIGAS; Quit" -ReportExportPath="<dir>" -log
```

Enable both plugins in the project, open the editor, and connect AI Game Developer (the Unreal-MCP
UI / sidecar); `StartupModule` registers the provider as a modular feature, so the GAS tools appear
in the tool list immediately. See the
[Unreal-MCP extension author guide](https://github.com/IvanMurzak/Unreal-MCP/blob/main/docs/EXTENSIONS.md).

## Release

Versioning is single-sourced from the `.uplugin` `VersionName`. Bump it in lock-step:

```powershell
./commands/bump-version.ps1 -NewVersion "0.2.0"   # updates .uplugin + GetExtensionVersion() + extension.json
```

Push to `main`. **`release.yml` is version-gated**: when the `VersionName` is a new value with no
existing tag, it runs the full test suite, packages the plugin **source** into a single
`UnrealAIGAS-<version>.zip`, and creates an **atomic GitHub Release** (tag `v<version>`) carrying
that one zip — the exact asset the installer downloads. The extension ships as source and UE compiles
it on the consumer's next editor open. (Track the core version floor with `./commands/update-core.ps1`.)

## CI

| Workflow | When | What |
| --- | --- | --- |
| `test_unreal_plugin.yml` | reusable | UBT host-editor build + UE Automation specs for one UE version (5.7) |
| `test_pull_request.yml` | PR | the reusable test (UE 5.7) + E2E `unreal-mcp-cli` tool checks |
| `release.yml` | push to `main` | version-gated → full tests → package source zip `UnrealAIGAS-<version>.zip` → atomic GitHub Release (tag `v<version>`) |
| `bump_version.yml` | manual | runs `bump-version.ps1`, opens a release PR |

The plugin / E2E jobs run on a **self-hosted Windows UE runner** and are **never red-by-absence** —
they stay *skipped* until a runner is registered and the repo variables are set:

- `UNREAL_RUNNER_READY = true` — enables the UBT build + Automation legs.
- `UNREAL_E2E_READY = true` — enables the E2E `install-extension` + tool-invocation leg.
- `UNREAL_HOST_PROJECT` — absolute path on the runner to a host `.uproject` with UnrealMCP available.

See [`docs/claude/ci.md`](docs/claude/ci.md) and [`docs/claude/release.md`](docs/claude/release.md).

## License

[Apache-2.0](LICENSE).
