<p align="center">
  <img src="assets/banner.svg" alt="Codex Coding Router 横幅">
</p>

<p align="center">
  <strong>仅支持 Windows · PowerShell 5.1+ · MIT 许可证</strong><br>
  <a href="README.md">English</a>
</p>

# Codex Coding Router

Codex Coding Router 是一个可移植的 Skill 与三代理路由包，用于有实际收益、边界
明确的委派。主代理默认负责分析、实现、验证和困难推理；只有可分离子任务能提供
独立证据、节省上下文或执行时间时才委派。工作区写入始终串行。

![路由架构](assets/routing-diagram.svg)

## 路由模型

| 任务形态 | 路由 | 模型 | 推理强度 | 沙箱 |
|---|---|---|---|---|
| 默认：分析、实现、测试、构建与困难推理 | 主代理 | 当前主模型 | 当前设置 | 当前设置 |
| 独立的只读定义、引用、调用链或日志证据收集 | `code_reader` | `gpt-5.6-luna` | `max` | `read-only` |
| 值得委派且步骤完全确定的机械修改或常规执行 | `code_worker` | `gpt-5.6-luna` | `max` | `workspace-write` |
| 边界明确的独立困难分析、指定实现或专家审查 | `code_expert` | `gpt-6-astra` | `xhigh` | `workspace-write` |

路由器会尊重用户明确指定的代理与只读限制，并且不会更改主模型。

### 委派与写入串行化

- 复杂度、测试和文件数量本身不会触发委派。
- `code_reader` 只收集证据；`code_worker` 处理步骤完全确定的机械工作与常规执行。
  两者都将超出范围的问题和证据返回主代理，worker 同时报告准确的工作区改动。
- `code_expert` 是可选的协作专家，不是必须升级到的目标。子代理不再继续委派。
- 只有互相独立的只读任务可以并行。
- 启动写代理前，必须等待相关 reader 完成。
- 不得并发运行 `code_worker` 与 `code_expert`；任何时刻最多一个工作区写代理，
  主代理也计入该限制。
- 除非用户明确要求，否则代理不得暂存或提交改动。

## 手动安装

本项目刻意不提供安装器。安装不需要管理员权限，也不会访问网络。下面的命令会先
检查全部目标；只要发现一个同名文件或目录，就在复制前停止。请自行审查并合并已有
文件，不要覆盖。

### 1. 预检并复制包文件

在仓库根目录打开 Windows PowerShell 5.1 或更高版本，然后运行：

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
    throw "请在 codex-coding-router 仓库根目录运行这些命令。"
}

$conflicts = @($copyPlan | Where-Object {
    Test-Path -LiteralPath $_.Destination
})
if ($conflicts.Count -gt 0) {
    $conflicts | ForEach-Object { Write-Warning $_.Destination }
    throw '未复制任何内容；请手动审查并合并已有文件。'
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

以上命令只复制三个代理模板和 `coding-model-router` Skill。完成下面的手动配置后，
请重启 Codex 或开始一个新任务。

### 2. 手动合并 `config.toml`

打开 `%USERPROFILE%\.codex\config.toml`。如果已经存在 `[agents]` 表，请编辑该表，
不要创建重复表。保留主模型和全部无关设置，并确保表中包含：

```toml
[agents]
enabled = true
max_concurrent_threads_per_session = 2
```

线程数 `2` 允许有价值的只读并行；Skill 会进一步执行更严格的单写者规则。

### 3. 可选：合并全局路由提醒

安装后的 Skill 已默认启用隐式调用。如果还需要持久提醒，可把下面内容手动合并到
`%USERPROFILE%\.codex\AGENTS.md`。请保留全部现有指导；若与全局规则冲突，则不要
添加。

```markdown
## Coding model routing

- Use $coding-model-router when choosing roles for useful, separable delegated work or when explicitly requested.
- Let the primary agent handle analysis, implementation, tests, and fixes by default; complexity alone does not require delegation.
- Run parallel agents only for mutually independent read-only tasks.
- Wait for related readers before starting a writer.
- Never run more than one workspace-writing agent, including the primary agent.
- Never run code_worker and code_expert concurrently. Honor explicit agent selection and read-only requests.
```

## 使用路由器

### 显式调用

在请求中写出 Skill 名称：

```text
使用 $coding-model-router 判断这次调查是否适合拆出一个独立的只读子任务。
```

也可以明确指定某个代理或要求只读调查。用户的明确选择优先于自动路由。

### 隐式调用

包内的 `skill/coding-model-router/agents/openai.yaml` 包含：

```yaml
policy:
  allow_implicit_invocation: true
```

在默认设置下，当有实际收益、边界明确的委派需求与 Skill 描述匹配时，Codex 可以
隐式调用它。调用 Skill 不要求创建子代理。普通局部修改、测试、构建和困难推理
都不是自动触发路由的理由。

若要关闭隐式调用，请手动打开安装后的
`%USERPROFILE%\.agents\skills\coding-model-router\agents\openai.yaml`，只修改策略值：

```yaml
policy:
  allow_implicit_invocation: false
```

显式 `$coding-model-router` 请求仍然有效。关闭该开关不会删除你手动加入全局
`AGENTS.md` 的可选提醒。

## 模型可用性

本包引用 `gpt-5.6-luna`（`max`）与 `gpt-6-astra`（`xhigh`）。模型 ID 和自定义代理支持会随 Codex
版本、账户、组织、地区与发布批次而变化，不保证所有环境都可用。若某个 ID 不可用，
请选择当前环境支持的模型和推理强度，同时保持各代理的职责与沙箱边界。
当前任务的工具接口和可用角色优先于旧示例；不要假设完整历史继承支持模型或推理
强度覆盖。配置文件改变不代表运行中的任务已重新加载，请在新任务中检查。

## 安全与兼容性

- 支持平台：Windows，Windows PowerShell 5.1 或更高版本。
- 本包不会修改主模型。
- 安装命令动态解析当前用户目录，并拒绝同名目标。
- 配置与全局指导均由用户手动合并，不覆盖无关设置。
- 仓库不含凭据、遥测、远程图片、跟踪资源或依赖网络的素材。
- 构建成功只能证明构建通过；运行正确性与性能需要另行验证。

## 手动移除

审查目标后，只删除已安装的三个代理文件和 `coding-model-router` Skill 目录。然后
仅手动移除自己添加的配置键或可选指导。不要删除或替换完整的 `config.toml` 或全局
`AGENTS.md`。

## 许可证

[MIT](LICENSE)
