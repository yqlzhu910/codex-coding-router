<p align="center">
  <img src="assets/banner.svg" alt="Codex Coding Router banner">
</p>

<p align="center">
  <strong>Windows-only · PowerShell 5.1+ · MIT licensed</strong><br>
  <a href="README.zh-CN.md">简体中文</a>
</p>

# Codex Coding Router

Codex Coding Router is a portable Skill and three-agent package for assigning
repository work to the smallest suitable coding agent. It keeps read-only
discovery separate from workspace writes, escalates difficult work instead of
stretching a lightweight worker beyond its role, and permits only one writer at
a time.

![Routing architecture](assets/routing-diagram.svg)

## Routing model

| Task shape | Route | Model | Reasoning | Sandbox |
|---|---|---|---|---|
| Tiny, context-complete task | Primary agent | Your current primary model | Current setting | Current setting |
| Large read-only discovery, references, call chains, logs, build evidence | `code_reader` | `gpt-5.6-luna` | `medium` | `read-only` |
| Fully specified local edit or ordinary small bug | `code_worker` | `gpt-5.6-luna` | `medium` | `workspace-write` |
| Algorithms, architecture, cross-module work, performance, memory safety, concurrency, numerical methods, geometry, voxels, or difficult debugging | `code_expert` | `gpt-5.6-sol` | `high` | `workspace-write` |

The router preserves explicit agent choices and read-only constraints. It does
not change the primary model.

### Escalation and write serialization

- `code_reader` gathers evidence only. If the evidence reveals difficult
  reasoning or a required change, preserve it and escalate to `code_expert`.
- `code_worker` handles bounded, mechanical work. If scope becomes difficult
  or unclear, stop it and pass its evidence and exact changes to
  `code_expert`.
- Complex implementation never moves back from `code_expert` to
  `code_worker`.
- Only mutually independent read-only work may run in parallel.
- Wait for related readers before starting a writer.
- Never run `code_worker` and `code_expert` concurrently. At most one agent may
  write to a workspace.
- No agent stages or commits unless the user explicitly requests it.

## Manual installation

Installation is intentionally manual. The repository contains no installer,
does not require administrator privileges, and does not access the network.
The commands below stop before copying if any managed destination already
exists. Review an existing file and merge it yourself; do not overwrite it.

### 1. Preflight and copy the package

Open Windows PowerShell 5.1 or later in the repository root, then run:

```powershell
$repoRoot = (Get-Location).Path
$codexRoot = Join-Path $env:USERPROFILE '.codex'
$agentRoot = Join-Path $codexRoot 'agents'
$skillRoot = Join-Path $env:USERPROFILE '.agents\skills\coding-model-router'

$copyPlan = @(
    [pscustomobject]@{
        Source = Join-Path $repoRoot 'agents\code_reader.toml'
        Destination = Join-Path $agentRoot 'code_reader.toml'
        IsDirectory = $false
    }
    [pscustomobject]@{
        Source = Join-Path $repoRoot 'agents\code_worker.toml'
        Destination = Join-Path $agentRoot 'code_worker.toml'
        IsDirectory = $false
    }
    [pscustomobject]@{
        Source = Join-Path $repoRoot 'agents\code_expert.toml'
        Destination = Join-Path $agentRoot 'code_expert.toml'
        IsDirectory = $false
    }
    [pscustomobject]@{
        Source = Join-Path $repoRoot 'skill\coding-model-router'
        Destination = $skillRoot
        IsDirectory = $true
    }
)

$missing = @($copyPlan | Where-Object {
    -not (Test-Path -LiteralPath $_.Source)
})
if ($missing.Count -gt 0) {
    throw "Run these commands from the codex-coding-router repository root."
}

$conflicts = @($copyPlan | Where-Object {
    Test-Path -LiteralPath $_.Destination
})
if ($conflicts.Count -gt 0) {
    $conflicts | ForEach-Object { Write-Warning $_.Destination }
    throw 'Nothing copied. Review and merge the existing files manually.'
}

$copyPlan | ForEach-Object {
    New-Item -ItemType Directory `
        -Path (Split-Path -Parent $_.Destination) `
        -Force | Out-Null

    if ($_.IsDirectory) {
        Copy-Item -LiteralPath $_.Source `
            -Destination $_.Destination `
            -Recurse
    }
    else {
        Copy-Item -LiteralPath $_.Source `
            -Destination $_.Destination
    }
}
```

This copies only three agent templates and the `coding-model-router` Skill.
Restart Codex or start a new task after completing the manual configuration
below.

### 2. Manually merge `config.toml`

Open `%USERPROFILE%\.codex\config.toml`. If an `[agents]` table already exists,
edit that table; do not add a duplicate. Preserve the primary model and all
unrelated settings. Ensure the table contains:

```toml
[agents]
enabled = true
max_concurrent_threads_per_session = 2
```

The limit of `2` allows useful read-only concurrency while the Skill enforces
its stricter single-writer rule.

### 3. Optionally merge a global routing reminder

Implicit invocation is already enabled in the installed Skill. If you also
want a persistent reminder, manually merge the following into
`%USERPROFILE%\.codex\AGENTS.md`. Preserve all existing guidance and omit this
block if it conflicts with your global rules.

```markdown
## Coding model routing

- Use $coding-model-router for substantive repository coding work.
- Run parallel agents only for mutually independent read-only tasks.
- Wait for related readers before starting a writer.
- Never run more than one workspace-writing agent.
```

## Using the router

### Explicit invocation

Name the Skill in your request:

```text
Use $coding-model-router to trace this call chain and implement the smallest verified fix.
```

You may also request a specific agent or a read-only investigation. Explicit
choices take priority over automatic routing.

### Implicit invocation

The packaged
`skill/coding-model-router/agents/openai.yaml` contains:

```yaml
policy:
  allow_implicit_invocation: true
```

With this default, Codex may invoke the Skill when its description matches a
substantive coding task. Automatic invocation is contextual, not a guarantee
that every request will spawn an agent.

To disable implicit invocation, manually open the installed file at
`%USERPROFILE%\.agents\skills\coding-model-router\agents\openai.yaml` and change
only the policy value:

```yaml
policy:
  allow_implicit_invocation: false
```

Explicit `$coding-model-router` requests continue to work. Disabling this flag
does not remove any optional reminder you manually added to global
`AGENTS.md`.

## Model availability

The package references `gpt-5.6-luna` and `gpt-5.6-sol`. Model IDs and custom
agent support can vary by Codex version, account, organization, region, and
rollout. Availability is not guaranteed. If an ID is unavailable, choose a
model your environment supports and preserve each agent's responsibility,
reasoning level, and sandbox boundary.

## Security and compatibility

- Supported platform: Windows with Windows PowerShell 5.1 or later.
- The package does not modify the primary model.
- Installation commands resolve the current profile dynamically and refuse
  same-name destinations.
- Configuration and global guidance are manual merges; nothing overwrites
  unrelated settings.
- The repository contains no credentials, telemetry, remote images, tracking
  resources, or network-dependent assets.
- Build success is build evidence only; runtime correctness and performance
  require separate verification.

## Manual removal

After reviewing the destinations, remove only the three installed agent files
and the installed `coding-model-router` Skill directory. Then manually remove
only the keys or optional guidance that you added. Do not delete or replace the
complete `config.toml` or global `AGENTS.md`.

## License

[MIT](LICENSE)
