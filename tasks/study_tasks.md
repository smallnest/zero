# Zero 源码学习计划

**项目：** zero —— 一个用 Go 写的终端编码 agent（open-source terminal coding agent）
**代码规模：** `internal/` 下约 **464 个源文件 + 496 个测试文件**（2.6 万+ 行）
**构建入口：** `cmd/zero/main.go` → `internal/cli`
**日期：** 2026-07-03
**目标：** 按“执行路径”而非“目录字母序”把 Zero 读通，能独立定位、修改、验证任意一条功能链路。

---

## 0. 核心原则

- **按执行路径读，不要按目录顺序读。** 代码量太大，硬啃学不动。跟着一次真实请求
  从 `main()` 走到 provider，再走回终端，比逐个包看有效得多。
- **测试即文档。** 测试文件（496 个）比源文件还多，命名清晰。读不懂某个函数时，
  先读它的 `_test.go`，用例会告诉你输入/输出契约。
- **边读边跑。** 每个阶段都配一个“动手任务”，在本地 `make build` / `go test` 里验证
  你的理解，而不是只在脑子里过。
- **用 grep/读测试代替猜。** 想知道某个类型谁在用，`grep -rn TypeName internal/`；
  想知道一条 CLI 子命令怎么分发，从 `internal/cli/app.go` 的 `runWithDeps` 开关看起。

---

## 1. 项目速览（已勘察）

Zero 是一个**分层的模块化单体**（layered modular monolith）。一个二进制，多种运行形态：

- 交互式 TUI（`zero`）
- 一次性脚本执行（`zero exec "..."`，支持 text / JSON / stream-JSON）
- 一堆管理子命令（auth、providers、models、sessions、cron、daemon、mcp、skill、specialist…）

关键事实：

- **单一入口极薄。** `cmd/zero/main.go` 只有 11 行，把 `os.Args` 转交给 `cli.Run`。
- **CLI 层是总调度台。** `internal/cli/app.go` 的 `runWithDeps`（第 209 行起）是一个大
  `switch args[0]`，把子命令路由到各自的 `runXxx` 函数。
- **agent 循环是编排核心。** `internal/agent/loop.go` 的 `Run()`（第 107 行）是那个
  “调模型 → 解析 tool call → 执行工具 → 回灌结果 → 再调模型”的回合循环。
- **工具、provider、sandbox、sessions 都是可插拔子系统**，通过接口和注册表接进 agent 循环。

---

## 2. 代码地图（按体量排序，抓大放小）

| 包 | 文件数 | 职责 | 优先级 |
| --- | ---: | --- | --- |
| `internal/tui/` | 174 | 交互式终端界面（Bubble Tea）：model/update/view、选择器、slash 命令 | 中（形态之一） |
| `internal/cli/` | 101 | 子命令分发、参数解析、把各子系统接起来 | **高（总入口）** |
| `internal/sandbox/` | 77 | 跨平台沙箱后端（macOS Seatbelt / Linux Landlock+seccomp / Windows / WSL） | 中 |
| `internal/tools/` | 75 | 全部工具实现 + 注册表（read/edit/bash/exec/grep/glob/web…约 40 个文件） | **高（agent 的手脚）** |
| `internal/agent/` | 47 | agent 运行循环、工具分发、权限状态机、压缩、prompt 组装 | **高（大脑）** |
| `internal/providers/` | 34 | provider 工厂 + anthropic/gemini/openai 适配器 | **高（连模型）** |
| `internal/daemon/` | 34 | 后台守护进程 / 远程 serve / attach | 低 |
| `internal/specialist/` | 31 | 子 agent（specialist）定义与加载 | 中 |
| `internal/agenteval/` | 31 | 离线 agent 评测框架 | 低 |
| `internal/mcp/` | 28 | MCP（Model Context Protocol）客户端集成 | 中 |
| `internal/oauth/` | 26 | OAuth 登录 / 订阅型 provider 认证 | 低 |
| `internal/config/` | 25 | 配置加载与 provider profile | **高（到处被依赖）** |
| `internal/swarm/` | 22 | 多 agent 协作原语 | 低 |
| `internal/sessions/` | 16 | 本地会话持久化（create/fork/resume、事件日志） | **高（状态）** |
| `internal/lsp/` | 15 | LSP 导航（definition/references/implementation） | 低 |

> 其余小包（`modelregistry`、`hooks`、`skills`、`plugins`、`repomap`、`streamjson`、
> `redaction`、`reasoning` 等）在需要时按需读，别一开始就钻进去。

---

## 3. 核心执行路径（先把这条主线走通）

一次 `zero exec "..."` 或一轮 TUI 对话，数据是这样流动的：

```
cmd/zero/main.go:9  main()
  └─ cli.Run(os.Args[1:], stdout, stderr)
       └─ internal/cli/app.go:209  runWithDeps()      # switch args[0] 分发
            ├─ (无参数)         → runInteractiveTUI()  # internal/tui/run.go:14  Run()
            └─ "exec" / "-p"    → runExec()            # internal/cli/exec.go:126
                 └─ internal/agent/loop.go:107  agent.Run(ctx, prompt, provider, options)
                      │  ── 回合循环 (for turn := 0; turn < maxTurns) ──
                      ├─ partitionTools()              # 本回合暴露哪些工具
                      ├─ compactor.maybeCompact()      # 上下文接近窗口就压缩
                      ├─ provider.Complete(request)    # 调模型（见下）
                      ├─ 解析返回里的 tool calls
                      ├─ executeToolCall()             # loop.go:774 执行单个工具
                      │    ├─ 权限判定 / 沙箱请求
                      │    ├─ hook: beforeTool / afterTool
                      │    └─ registry 里对应 tools.Tool.Run()
                      └─ 把工具结果回灌进 messages，进入下一回合
```

Provider 侧（模型怎么被选中和调用）：

```
internal/providers/factory.go:39  New(profile, options)
  └─ resolveProfile() 决定 kind（anthropic / gemini / openai / codex / 兼容端点）
       └─ 返回一个实现 zeroruntime.Provider 的适配器
            └─ anthropic|gemini|openai/provider.go 各自把统一请求翻译成厂商 API
```

会话状态（TUI/exec 都会落盘）：

```
internal/sessions/store.go:202  NewStore()
  ├─ Create()        # 新会话
  ├─ Fork()          # 分叉一次已有会话
  ├─ ListResumable() / LatestResumable()   # resume 支持
  └─ AppendEvent()   # 事件日志（逐条 append）
```

---

## 4. 构建 / 测试 / 文档入口

**构建与验证（来自 `Makefile`）：**

```bash
make build        # go build -o zero ./cmd/zero  —— 产出 ./zero
make build-all    # go build ./...               —— 编译全部
make test         # go test ./... -race -count=1  —— CI 同款（带竞态检测）
make test-quick   # go test ./...                 —— 快，无 race
make vet          # go vet ./...
make fmt          # gofmt -w（改写）
make fmt-check    # gofmt -l（只检查，CI 用）
make lint         # = fmt-check + vet             —— 开 PR 前跑这个
```

**跑单个包的测试（学某个包时最常用）：**

```bash
go test ./internal/agent/ -run TestRun -v
go test ./internal/tools/ -v
```

**文档（`docs/`）——按角色分组：**

- 用户：`docs/INSTALL.md`、`docs/UPDATE.md`、`docs/oauth-subscriptions.md`
- 自动化/集成：`docs/STREAM_JSON_PROTOCOL.md`、`docs/GITHUB_ACTION.md`、`docs/SPECIALISTS.md`
- 维护者：`docs/BENCHMARK.md`、`docs/PERFORMANCE.md`、`docs/AGENT_EVALS.md`、`docs/NPM_WRAPPER_SMOKE.md`
- 扩展机制：仓库根的 `AGENTS.MD`（AGENTS.md / specialists / skills / hooks / MCP / plugins 全在这）
- Prompt 本体：`internal/agent/system_prompt.md`（agent 的系统提示词，直接读它最能理解“它以为自己是谁”）

---

## 5. 分阶段学习计划（动手清单）

> 用法：每完成一项就把 `[ ]` 改成 `[x]`。每个阶段末尾都有一个“验收动作”，
> 跑通了才算真懂。预估时间按每天 1–2 小时算。

### 阶段 A —— 建立地图与跑通构建（0.5 天）

- [ ] 读 `README.md`（前 120 行）+ `docs/README.md`，搞清 Zero 有哪几种运行形态
- [ ] `make build`，成功产出 `./zero`；`./zero --help` 看子命令列表
- [ ] `make test-quick`，确认本地测试基线能过（记下耗时）
- [ ] 读本文档第 2、3 节，对照 `internal/` 目录，能说出前 6 个大包各自干嘛
- [ ] **验收：** 画一张“子命令 → 处理函数”的对应表（看 `internal/cli/app.go` 的 switch）

### 阶段 B —— 走通 CLI 分发层（0.5–1 天）

- [ ] 读 `cmd/zero/main.go`（11 行）→ `internal/cli/app.go` 的 `Run` / `runWithDeps`
- [ ] 跟踪 `zero exec` 这条 case：找到 `internal/cli/exec.go:126 runExec`，通读它做的准备工作
- [ ] 搞清 `appDeps` 是什么、`fillAppDeps` 注入了哪些依赖（provider、store、registry）
- [ ] 看 `newCoreRegistry`（`app.go:819`）如何把工具集注册进来
- [ ] **验收：** 用 `go test ./internal/cli/ -run Exec -v` 跑几个 exec 相关测试，读懂 1 个用例

### 阶段 C —— 攻克 agent 循环（1.5–2 天，最重要）

- [ ] 通读 `internal/agent/loop.go:107 Run()` 的回合循环骨架（turn loop）
- [ ] 精读 `executeToolCall`（`loop.go:774`）：参数解析 → 权限 → 沙箱 → hook → 执行
- [ ] 读 `internal/agent/types.go` 的 `Options` 结构体（41 字段），理解它承载了什么
- [ ] 读 `internal/agent/system_prompt.md` + `system_prompt.go`，看 prompt 怎么组装
- [ ] 读 `internal/agent/compaction.go`，理解上下文接近窗口时如何压缩历史
- [ ] 扫一眼权限/沙箱重试族函数（`maybeRetry...`、`shouldRequestPermission`、`effectivePermission`）
- [ ] **验收：** `go test ./internal/agent/ -run TestRun -v`，读懂 loop 的一个端到端测试

### 阶段 D —— 工具子系统（1 天）

- [ ] 读 `internal/tools/registry.go`（`NewRegistry` / `Register` / 查找）
- [ ] 读 `internal/tools/types.go`：`Tool` 接口、`SideEffect`、`Permission`、`Status`
- [ ] 精读 2 个简单工具：`read_file.go` 和 `grep.go`（无副作用类）
- [ ] 精读 2 个有副作用工具：`edit_file.go`（写）和 `bash.go` / `exec_command.go`（执行）
- [ ] 理解 deferred / tool_search 机制（`deferred.go`、`tool_search.go`）——工具按需暴露
- [ ] **验收：** 照 `read_file.go` 的模式，本地写一个玩具工具并注册，跑测试确认能被发现

### 阶段 E —— Provider 适配层（1 天）

- [ ] 读 `internal/providers/factory.go:39 New()` + `resolveProfile`，理解 profile→kind 决策
- [ ] 选一个厂商深入：`internal/providers/anthropic/provider.go`，看统一请求如何翻译成厂商格式
- [ ] 对照 `internal/providers/openai/`（含 `codex.go`）看差异点在哪
- [ ] 读 `internal/config/` 里 provider profile 的加载（`grep -rn ProviderProfile internal/config`）
- [ ] **验收：** 说清“加一个新的 OpenAI-兼容 provider 需要动哪几个文件”

### 阶段 F —— 会话与状态（0.5 天）

- [ ] 读 `internal/sessions/store.go`：`Create` / `Fork` / `ListResumable` / `AppendEvent`
- [ ] 搞清会话落盘位置（`DefaultRoot`）与事件日志格式（`Event` / `EventType`）
- [ ] 跑一次 `zero exec "..."`，然后 `zero sessions` 系列命令观察落盘结果
- [ ] **验收：** 解释 resume 和 fork 的区别，各自读/写了 store 的什么

### 阶段 G —— 选修专题（按兴趣，各 0.5 天）

- [ ] **TUI：** `internal/tui/run.go` → `model.go` 的 `newModel` / `Update` / `View`（Bubble Tea 三件套）
- [ ] **Sandbox：** `internal/sandbox/` 按当前 OS 选一个后端读（mac 读 seatbelt，Linux 读 landlock）
- [ ] **扩展机制：** 照 `AGENTS.MD` 动手加一个 specialist 或 skill，验证被加载
- [ ] **MCP：** `internal/mcp/` + `internal/cli/mcp_*.go`，接一个本地 MCP server 试试
- [ ] **stream-JSON：** `docs/STREAM_JSON_PROTOCOL.md` + `internal/streamjson/`，跑 `zero exec --output-format stream-json`

### 阶段 H —— 综合实战（收尾，1 天）

- [ ] 挑一个真实小改动（如给某工具加一个参数，或改一句 prompt），走完整流程
- [ ] 改代码 → `make lint` → `make test`（或范围内 `go test ./internal/xxx/`）→ 确认绿
- [ ] 用 `./zero exec` 手动验证你的改动确实生效
- [ ] **验收：** 能独立完成“定位 → 修改 → 加/改测试 → 验证”整条链路

---

## 6. 速查表（学习期贴墙用）

| 想找的东西 | 去哪 |
| --- | --- |
| 程序入口 | `cmd/zero/main.go:9` |
| 子命令分发 | `internal/cli/app.go:209 runWithDeps`（大 switch） |
| 一次性执行入口 | `internal/cli/exec.go:126 runExec` |
| agent 回合循环 | `internal/agent/loop.go:107 Run` |
| 单个工具执行 | `internal/agent/loop.go:774 executeToolCall` |
| 工具注册表 | `internal/tools/registry.go` |
| 工具接口定义 | `internal/tools/types.go` |
| 选模型/建 provider | `internal/providers/factory.go:39 New` |
| 系统 prompt 文本 | `internal/agent/system_prompt.md` |
| 会话持久化 | `internal/sessions/store.go` |
| 构建/测试命令 | `Makefile` |
| 扩展机制总说明 | 根目录 `AGENTS.MD` |

**常用命令：**

```bash
make build && ./zero --help          # 构建 + 看命令
go test ./internal/agent/ -v         # 学某包时跑它的测试
grep -rn "SymbolName" internal/      # 找符号的使用点
make lint                            # 开 PR 前必跑
```
