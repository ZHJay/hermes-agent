# Hermes Issue Autopilot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个独立的 Hermes 用户插件，自动调查并修复 `NousResearch/hermes-agent` 的 bug issue，且仅在冲突复查、回归验证和目标测试全部通过后创建 draft PR。

**Architecture:** 源码位于独立仓库 `/Users/zhanghjay/Desktop/hermes-issue-autopilot`，通过软链安装到 `~/.hermes/plugins/issue-autopilot`。LaunchAgent 每两分钟启动一个带文件锁的扫描进程；进程最多并发三个 issue worker，每个 worker 依次完成 Codex 调查、claim、TDD 修复、独立验证和 draft PR 发布。

**Tech Stack:** Python 3.11–3.13、stdlib、PyYAML 6.0.3、SQLite、Git/GitHub CLI、Codex CLI、pytest、macOS launchd。

## Global Constraints

- 上游仓库固定为 `NousResearch/hermes-agent`，不允许配置为其他仓库。
- 不修改 Hermes core，不注册模型工具；插件只注册 `hermes issue-autopilot` CLI。
- 所有 GitHub 发布动作由编排器执行，Codex 不继承 GitHub/SSH 凭据。
- 每个修复必须包含回归测试、非空 issue-scoped diff，并通过编排器独立重跑。
- 不合并 PR、不修改标签、不关闭 issue；PR 正文不用 `Fixes/Closes` 自动关闭语法。
- 实现源码与提交位于独立仓库；Hermes core 仓库除本计划文档外不承载插件代码。

---

## Public Interfaces and Data Contracts

插件公开命令：

- `start`：生成默认配置、初始化数据库和专用 clone、安装并 bootstrap LaunchAgent。
- `stop`：bootout LaunchAgent 并删除 plist，保留数据库、日志和 worktree。
- `status`：显示服务状态、最近扫描、活动 worker、claim、测试、cooldown 和 draft PR。
- `scan-once`：同步执行一次完整扫描；若 `scan.lock` 已被占用则安全退出。
- `logs [--follow] [--issue N] [--lines N]`：默认显示 scanner 最后 200 行，指定 issue 时显示对应 worker 日志。

默认 `config.yaml`：

```yaml
schema_version: 1
upstream_repository: NousResearch/hermes-agent
poll_interval_seconds: 120
max_workers: 3
codex_command:
  - /Applications/ChatGPT.app/Contents/Resources/codex
  - exec
codex_timeout_seconds: 3600
test_timeout_seconds: 1800
cooldown_seconds: 86400
fork_repository: ZHJay/hermes-agent
push_remote_name: fork
branch_prefix: codex/issue-
```

`max_workers` 只接受 1–3；命令数组禁止 shell 字符串、`danger-full-access` 和 sandbox bypass 参数。

核心类型为 `IssueCandidate`、`ConflictReport`、`InvestigationResult`、`TestSpec`、`FixResult` 和 `DiffEvidence`。`GitHubClient`、`CodexWorker` 和 `RepositoryManager` 使用 Protocol，以便端到端测试注入 fake。

SQLite 使用 `issues`、`attempts`、`external_effects` 三张表。Issue 状态限定为：

`investigating → claim_pending → claimed → fixing → validating → pr_pending → draft_pr`

失败或冲突进入 `cooldown`，关闭 issue 进入 `closed`。`external_effects` 以 issue、动作类型和隐藏 marker 组成唯一键，为 comment 和 PR 提供 outbox 式幂等恢复。

### Task 1: 独立仓库、插件桥接和配置

**Files:** `plugin.yaml`、根 `__init__.py`、`pyproject.toml`、`issue_autopilot/{plugin,cli,config,models}.py`、`tests/test_plugin_config.py`

- [ ] 先编写失败测试，验证插件只注册一个 CLI、零工具/钩子，并验证配置默认值和非法配置。
- [ ] 创建 Python 包和 `kind: standalone` manifest；`register(ctx)` 仅调用 `ctx.register_cli_command(...)`。
- [ ] 实现首次启动时原子生成配置，以及固定上游、worker 上限、timeout、fork 名称和 Codex argv 校验。
- [ ] 运行 `uv run --extra dev pytest tests/test_plugin_config.py -q`。
- [ ] 提交 `chore: scaffold issue autopilot plugin`。

### Task 2: 持久状态、状态机与幂等副作用

**Files:** `issue_autopilot/state.py`、`tests/test_state.py`

- [ ] 用临时数据库编写状态迁移、compare-and-swap、cooldown 和 restart recovery 失败测试。
- [ ] 创建带 `PRAGMA user_version` 的 schema；启用 WAL、busy timeout 和每线程独立连接。
- [ ] 实现 `claim_pending`、`pr_pending`、release comment 等外部动作的 marker/outbox 恢复。
- [ ] 将中断的调查/修复恢复到原 worktree；验证阶段重跑测试；PR pending 阶段先搜索已有 PR。
- [ ] 运行 `uv run --extra dev pytest tests/test_state.py -q`。
- [ ] 提交 `feat: add durable autopilot state`。

### Task 3: GitHub 选择、冲突检测和发布接口

**Files:** `issue_autopilot/github.py`、`issue_autopilot/templates.py`、`tests/test_github.py`

- [ ] 用 fake GitHub client 覆盖 bug 标签并集、feature 排除、claim race、timeline commit 和 PR 冲突。
- [ ] 候选 issue 必须含 `type/bug` 或 `bug`，排除 `type/feature`、`feature`、`enhancement`；其他风险标签保留。
- [ ] 通过分页 REST/`gh` 调用读取 issue、全部 comments、timeline，并搜索精确 `#N` 或 issue URL 的 open/merged PR。
- [ ] 将其他 assignee、其他用户的 autopilot marker 或明确“正在处理”评论视作 claim；忽略当前用户已释放的旧 claim。
- [ ] claim、release 和 PR 正文写入稳定 HTML marker；创建前搜索 marker/head branch，模糊失败后再次搜索。
- [ ] claim 内容包括范围、测试计划和风险标签；PR 包括 issue 链接、复现、实现和精确测试结果，不使用自动关闭关键字。
- [ ] 运行 `uv run --extra dev pytest tests/test_github.py -q`。
- [ ] 提交 `feat: add GitHub conflict and publishing client`。

### Task 4: 专用 clone、worktree 和可信测试验证

**Files:** `issue_autopilot/repository.py`、`issue_autopilot/processes.py`、`tests/test_repository.py`

- [ ] 使用临时 bare upstream/fork 编写 clone、三 worktree 隔离、重启复用和无 force-push 测试。
- [ ] 在 `~/.hermes/issue-autopilot/repository/` 维护专用 clone，在 `worktrees/issue-N/` 创建 `codex/issue-N-<slug>` 分支。
- [ ] 每轮从最新 upstream `main` 创建 worktree；Codex 不得提交，HEAD 必须仍等于记录的 base SHA。
- [ ] 用临时 Git index 生成包含 untracked 文件的 binary patch，并校验 diff 非空、路径落在调查阶段的 scope prefixes、同时包含回归文件和非测试修复。
- [ ] 在最新 `main` 的临时验证 worktree 中仅应用 test patch，要求目标测试以预期特征失败；再应用 full patch，要求同一测试在 timeout 内退出 0。
- [ ] 支持受控 runner：`python -m pytest`、workspace `npm test -- ...` 和仓库 `scripts/run_tests.sh`；拒绝绝对路径、`..`、shell 操作符和未知 runner。
- [ ] Python 依赖使用共享、锁文件 keyed 的 uv 环境；Node runner 缺依赖时在 issue worktree 执行一次锁文件约束的 `npm ci`。
- [ ] 运行 `uv run --extra dev pytest tests/test_repository.py -q`。
- [ ] 提交 `feat: add isolated worktree verification`。

### Task 5: 两阶段 Codex worker

**Files:** `issue_autopilot/codex_worker.py`、`issue_autopilot/prompts.py`、`issue_autopilot/schemas/*.json`、`tests/test_codex_worker.py`

- [ ] 使用 fake Codex executable 测试调查成功、无法复现、结构错误、timeout、无 diff 和测试失败。
- [ ] 调查阶段只验证 current `main`，输出复现证据、scope prefixes 和 `TestSpec`，不得留下 tracked diff。
- [ ] 冲突复查及 claim 成功后，修复阶段必须先增加回归测试，再做最小兼容修复，不得 commit、push 或运行 `gh`。
- [ ] 两阶段均通过 stdin 传入带“不可信 issue 内容”边界的 prompt，并使用 JSON Schema 与 `--output-last-message`。
- [ ] Codex 固定追加 `--ignore-user-config --ephemeral --sandbox workspace-write`、`approval_policy="never"` 和禁用 sandbox 网络。
- [ ] 子进程环境移除 GitHub token、SSH agent、云凭据等变量；日志只记录脱敏后的阶段、时长和结果摘要，不落原始 prompt/API 响应。
- [ ] 运行 `uv run --extra dev pytest tests/test_codex_worker.py -q`。
- [ ] 提交 `feat: add structured Codex workers`。

### Task 6: 扫描编排、并发和发布闭环

**Files:** `issue_autopilot/orchestrator.py`、`tests/test_orchestrator.py`

- [ ] 先建立 fake GitHub + fake Codex + real temporary Git repositories 的端到端测试。
- [ ] `scan_once()` 获取 `scan.lock`，先恢复未完成 issue，再按 issue number 升序填充空闲槽位。
- [ ] 使用 `ThreadPoolExecutor(max_workers=config.max_workers)`；一个 issue 从调查到发布始终占用一个 worker 槽。
- [ ] 调查确认复现后执行完整冲突检查、持久化 claim pending、upsert claim，并再次检查 claim race。
- [ ] 修复后执行 diff audit、最新 main 的 regression/full patch 验证，然后立即做第二次完整 GitHub 冲突检查。
- [ ] 通过后才 commit、非 force push 到 fork，并创建以 `NousResearch/hermes-agent:main` 为 base 的 draft PR。
- [ ] 无法复现、无有效修复、测试失败或发布前冲突时 upsert release update，并设置 24 小时 cooldown；网络/CLI 暂时故障保留可恢复状态，不发布误导性评论。
- [ ] draft PR 创建成功后清理本地 worktree，保留 branch、测试证据和 PR URL。
- [ ] 运行 `uv run --extra dev pytest tests/test_orchestrator.py -q`。
- [ ] 提交 `feat: orchestrate issue autopilot workflow`。

### Task 7: LaunchAgent、状态展示和运维文档

**Files:** `issue_autopilot/service.py`、`issue_autopilot/cli.py`、`README.md`、`tests/test_service_cli.py`

- [ ] 用临时 HOME 和 fake `launchctl` 测试 plist、start/stop/status、日志选择和重复启动。
- [ ] plist 固定为 `~/Library/LaunchAgents/ai.hermes.issue-autopilot.plist`，使用 `StartInterval=120`、`RunAtLoad=true` 和独立 stdout/stderr 日志。
- [ ] `start` 写入解析后的 Python/source/PATH，验证 `gh`、Codex、Git、fork 和认证状态后 bootstrap。
- [ ] `status` 只读 launchd 与 SQLite；`logs` 支持 scanner、issue worker、tail 和 follow。
- [ ] README 记录独立仓库初始化、软链、`hermes plugins enable issue-autopilot --no-allow-tool-override`、配置和故障恢复。
- [ ] 运行 `uv run --extra dev pytest tests/test_service_cli.py -q`。
- [ ] 提交 `feat: add issue autopilot service CLI`。

## Test and Acceptance Plan

- `uv run --extra dev pytest -q` 全量通过。
- 标签测试证明 bug 被选中、纯 feature 被排除、风险标签不会被静默过滤。
- race 测试证明 claim 后出现外部 claim 时会释放，不进入修复。
- restart 测试分别中断在 claim comment、Codex、验证、push 和 PR 创建边界，确认无重复 comment、worktree 或 PR。
- 并发测试记录峰值 worker 数为 3，并证明每个 worker 的 Git common dir 相同但 worktree 和 branch 独立。
- 失败矩阵覆盖无法复现、调查脏 diff、无生产变更、baseline 不失败、full patch 测试失败、最新 main 已修复和 pre-PR 冲突；所有路径的 PR 创建次数均为 0。
- 成功路径必须产生一个 commit、一次 push、恰好一个 draft PR，且 PR 正文包含 issue URL、风险标签、复现、实现和精确测试证据。
- 插件注册 smoke test 只运行 `hermes issue-autopilot status`；实现验收期间不对真实 issue 运行 `scan-once`，也不实际启动 LaunchAgent。

## Assumptions and Defaults

- 独立源码仓库采用 `/Users/zhanghjay/Desktop/hermes-issue-autopilot`，安装方式为软链。
- worktree 使用 Autopilot 专用 clone，不复用当前 Hermes checkout。
- 当前 GitHub 身份和默认 fork 为 `ZHJay` / `ZHJay/hermes-agent`。
- v1 的服务管理只支持 macOS LaunchAgent。
- cooldown 默认 24 小时；到期后可重新调查，但复用并更新已有 marker comment。
- closed-unmerged PR 不构成永久冲突；open/merged PR、其他 contributor claim 和已进入 main 的关联 commit 构成冲突。
- 所有临时 Git 清理只作用于 `~/.hermes/issue-autopilot/` 下由插件创建并在数据库登记的 worktree。
