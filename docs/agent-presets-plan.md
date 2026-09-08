# Agent Presets — `/agents` 面板分组预配置实施方案 (v1)

> 状态：设计稿，未实施。代码锚点行号基于本仓库 `study-v18.1.14` 分支（与已发布的 `v18.1.14` 二进制一致）。

## 1. 目标

在 `/agents` 面板内增加"分组预配置"能力：把一组 `agent → model` 覆盖存成命名预设（preset），一键整组切换，用于按时间段/成本策略切换子 agent 模型（例如白天高峰组、夜间空闲组）。

**非目标（v1）**：预设不携带 `prewalk` / `advisor` / `disabledAgents`；不做按时间自动切换（留给外部 cron 或用户扩展）；不跨 profile 同步。

## 2. 现状锚点（已核对）

| 位置 | 作用 |
| --- | --- |
| `src/config/settings-schema.ts:5154-5165` | `task.agentModelOverrides` / `agentPrewalk` / `agentAdvisor` 三个 record 声明 |
| `src/config/settings-schema.ts:370-374` | `RecordDef<T>`：`default: Record<string, T>`，`T` 为泛型（`statusLine.segmentOptions`、`images.urls.credentials` 已是嵌套对象值，说明嵌套 record 结构可行） |
| `src/config/settings.ts:662-687` | `set(path, value)`：写 **global 层**、标记 modified、`#queueSave()` 防抖落盘；保存时按 modified path 与磁盘文件做合并，不整体覆盖 |
| `src/config/settings.ts:637-659` | `get(path)`：global + project + runtime override 合并后取值，缺省回落 schema default |
| `src/cli/config-cli.ts:216-230` | `omp config set` 的 `record` 分支按 JSON 解析 —— schema 加键后 CLI 自动可用 |
| `src/modes/components/agents-hub.ts` | 面板本体（1473 行） |
| `src/modes/controllers/selector-controller.ts:484` | 面板唯一宿主，`AgentsHubComponent.create(...)`；本方案不需要改宿主 |
| `test/agents-hub.test.ts` | 9 条契约测试，`hub.handleInput(...)` 驱动 + 断言 `settings.get(...)` |

`agents-hub.ts` 内部关键锚点：

| 行 | 符号 | 说明 |
| --- | --- | --- |
| ~62-70 | `HubAgent` | `AgentDefinition` + `disabled` / `overrideModel` / `prewalkOverride` / `advisorOverride` |
| ~52-58 | `SidebarEntry` | `kind: "all" \| "source" \| "new" \| "separator"` |
| ~78-90 | `StripChip.action` | `toggle \| property \| set \| pick \| pattern` |
| 305-345 | `#reload()` | 读 4 个 settings record，构建 `#allAgents` |
| 350-373 | `#buildSidebar()` | 生成 `All agents` → 来源分组 → `+ New agent` |
| 379-385 | `#buildRows()` | 侧栏 scope + 搜索过滤 → `ListRow[]` |
| 438-447 | `#toggleAgent()` | 写 `task.disabledAgents` |
| 449-462 | `#persistRecord()` | 写 `task.agentModelOverrides` / `agentPrewalk` / `agentAdvisor` |
| 533-556 | `#openAgentStrip()` | 一级 strip（enable/model/prewalk/advisor） |
| 558-609 | `#openPropertyStrip()` | 二级 strip（pick model…/pattern…/clear override） |
| 611-660 | `#openPatternStrip()` / `#submitPattern()` | 内联 `Input` 文本输入范式 |
| 623-649 | `#activateStripChip()` | chip 动作分发 |
| 838-960 | `handleInput()` | 键位总分发；Esc → `#callbacks.onCancel()` |
| 962-969 | `#activateRow()` | Enter 激活 body 行 |
| 971-1007 | `#handleStripInput()` | strip 内的左右/Enter/Esc |
| 1053-1075 | `#moveSidebar()` | 侧栏上下移动（跳过 separator） |
| 1077-1156 | `#routeMouseEvent()` | 侧栏/列表/芯片的鼠标命中 |
| 1170-1203 | `#renderSidebar()` | 侧栏渲染 |
| 1205-1223 | `#statusRow()` | body 顶部状态行 |
| 1225-1315 | `#renderList()` | body 列表 + 选中 agent 详情块 |
| 1317-1380 | `#renderCreate()` | 第三种 body 模式（AI 生成 agent），可作为 preset 详情的模板 |
| 1382-1403 | `#footerHint()` | footer 提示文案 |
| 1405-1439 | `#renderFooter()` | chip strip 渲染 + `#chipRanges` 命中区记录 |
| 1441-1473 | `render()` | 组装 sidebar/body/footer |

## 3. 数据模型

新增一个 settings record：

```ts
// src/config/settings-schema.ts
const DEFAULT_AGENT_PRESETS: Record<string, Record<string, string>> = {};

// 紧邻 task.agentModelOverrides (L5154)
"task.agentPresets": {
	type: "record",
	default: DEFAULT_AGENT_PRESETS,
},
```

YAML 形态（`~/.omp/agent/config.yml`）：

```yaml
task:
  agentPresets:
    peak:
      scout: qwen-local/Qwen
      sonic: qwen-local/Qwen
      task: qwen-local/Qwen
      reviewer: volcengine/ark-code-latest
      security-reviewer: volcengine/ark-code-latest
    offpeak:
      scout: deepseek/deepseek-chat
      task: deepseek/deepseek-chat
      reviewer: deepseek/deepseek-reasoner
```

约定：

- 值语义与 `task.agentModelOverrides` 完全一致：agent 名 → model selector（支持 `@role` 别名与 `:level` 后缀）。
- **读时归一化**（面板内 `#normalizePresets()`）：只保留 `typeof value === "string"` 且 `trim()` 非空的条目；预设名去空白、长度 ≤ 32、不含空格；非法条目静默丢弃并在状态行提示一条 `N malformed preset entries ignored`。手改 YAML 不应导致面板崩溃。
- 写时始终整体写 `task.agentPresets`（`settings.set` 的路径级合并保证不会碰同文件其他键）。

**为什么不落独立文件**：settings 自带 global/project/overlay 分层、`omp config get/set`、防抖落盘与并发合并，且面板已有 `settings.set("task.agentModelOverrides", ...)` 的同款写入路径。独立 YAML 文件（用户扩展目前的做法）会丢掉分层与 CLI。

**嵌套 record 已实测可行**（在已发布的 v18.1.14 二进制上）：

```bash
PI_CODING_AGENT_DIR=/tmp/probe omp config set statusLine.segmentOptions '{"peak":{"scout":"qwen-local/Qwen"}}'
# → 落盘为 statusLine.segmentOptions.peak.scout，get 回读为嵌套对象
```

说明 record 值只要是 JSON 对象即可（`RecordDef<T>` 的 `T` 不限于标量），`task.agentPresets` 不需要改 schema 类型系统。

## 4. 语义

- **apply（replace，默认）**：`settings.set("task.agentModelOverrides", { ...preset })`。
  整组替换，保证"切组"结果确定，不会残留上一组的 agent 条目。若当前覆盖里存在 preset 未列出的 agent，先弹确认（见 §5）。
- **apply（merge）**：`settings.set("task.agentModelOverrides", { ...current, ...preset })`。
  只覆盖 preset 列出的 agent，其余保留；用于"只切换部分 agent"。
- **active preset 派生，不新增状态**：面板按 `keys(preset) == keys(current) && 每个值相等` 判断哪个 preset 与当前覆盖一致，用于侧栏高亮与状态行。避免引入 `task.activePreset` 这类需要迁移和一致性维护的状态。
- **生效时机**：应用后无需通知会话。`src/task/structured-subagent.ts:299` 在每次 spawn 时读取 `task.agentModelOverrides`，运行中的会话在下一次 task 派发即生效（该行为已在二进制 v18.1.14 上实测）。
- **不变量**：`agentModelOverrides` 仍是唯一的模型覆盖来源；preset 只是它的命名快照。任何其他消费者（`vibe/runtime.ts:400`、`security/coordinator.ts:233`、`selector-controller.ts:872`）都不需要改动。

## 5. UX 设计

侧栏新增一个 `Presets` 分区（位于来源分组之后、`+ New agent` 之前）：

```
 All agents   5
 ──────────────
 Project      1
 Bundled      5
 ──────────────
 Presets
  ● peak      5
  ● offpeak   4
  + New preset…
 ──────────────
 + New agent
```

- `SidebarEntry.kind` 增加 `"preset"` 与 `"new-preset"`；`annotation` 显示该预设覆盖的 agent 数；与当前覆盖一致时用 `theme.status.enabled` 高亮，否则用 `theme.fg("dim")`。
- 选中 preset（Enter / 点击）→ body 切换为 **preset 详情**（第四种 body 模式，参照 `#renderCreate` 的结构）：
  - 标题行：`Preset peak · 5 agents`（与当前覆盖一致时追加 `(active)`）
  - 表格：`agent → selector`，对每个条目显示与当前覆盖的差异标记：`=`（相同）/ `≠`（将被覆盖）/ `+`（当前不存在）
  - preset 里引用了已不存在的 agent：标 `missing`，但仍保留在数据中（agent 集合是动态的，删除会破坏用户数据）
- footer strip（复用 `StripChip` 机制）：
  `[ apply ]  [ merge ]  [ rename… ]  [ delete ]`
- `+ New preset…` → 内联 `Input` 输入名字（沿用 `#openPatternStrip` / `#submitPattern` 的范式）：Enter 创建、Esc 取消；默认内容为**当前 `task.agentModelOverrides` 的快照**，因此"调好一组再存下来"是自然流程。重名则拒绝并在状态行提示。
- `rename…` 同样走内联 `Input`；`delete` 直接执行并在状态行给出 `Deleted preset offpeak`（预设是小数据，不做二次确认，但可通过 Ctrl+R 重载文件恢复）。
- Esc 逐级回退：preset 详情 → 侧栏；与现有 strip 行为一致。
- `apply` 确认：仅当当前 `agentModelOverrides` 中存在 preset 未列出的条目时，footer 临时切换为确认条 `[ replace — drop N entries ] [ merge instead ] [ cancel ]`；否则直接应用。
- 鼠标：`#routeMouseEvent()` 的侧栏分支已按 `#entries` 通用遍历，只需为新 `kind` 加分发（与 `new` 分支同构）。

## 6. 代码改动清单

### 6.1 `packages/coding-agent/src/config/settings-schema.ts`

- 新增 `DEFAULT_AGENT_PRESETS` 常量（紧邻 `DEFAULT_AGENT_MODEL_OVERRIDES`）。
- 在 `task.agentModelOverrides`（L5154）之后插入 `task.agentPresets`。

Schema 接线是自动的：`SettingPath` / `SettingValue<P>` 由 `SETTINGS_SCHEMA` 推导（`settings-schema.ts:6088`），`SETTING_PATH_SEGMENTS` 由 `Object.keys(SETTINGS_SCHEMA)` 生成（`settings.ts:176-178`）。加键后 `settings.get/set`、`omp config get/set/list`（`config-cli.ts:216` 的 record-JSON 分支）立即生效，无需改其他地方。

### 6.2 `packages/coding-agent/src/main.ts`

`HOST_DEFAULTED_SETTING_PATHS`（L140-157）列出 RPC/ACP 协议宿主**不继承**用户全局配置的 task 设置，现有 `task.disabledAgents` / `agentModelOverrides` / `agentPrewalk` / `agentAdvisor` 都在其中。把 `task.agentPresets` 加进去：preset 是交互式 `/agents` 面板的本地偏好，嵌入式宿主应只拿 schema 默认值（`{}`），除非它自己显式配置。

不加入的后果：RPC/ACP 宿主会继承用户机器上的 preset 列表。

### 6.3 `packages/coding-agent/src/modes/components/agents-hub.ts`

类型层：

```ts
interface SidebarEntry {
	kind: "all" | "source" | "new" | "separator" | "preset" | "new-preset";
	presetName?: string;
	// …
}

// StripChip.action 增加：
| { kind: "preset"; action: "apply" | "merge" | "rename" | "delete"; preset: string }
```

状态层：

```ts
#presetDetail: string | null = null;      // 正在查看/编辑的预设
#presetInput: Input | null = null;        // 新建/重命名的名字输入
#presetInputMode: "create" | "rename" | null = null;
#presetConfirmDrop: number = 0;           // >0 时 footer 进入确认态
```

方法层（新增，均保持 `#private` 命名风格）：

| 方法 | 职责 |
| --- | --- |
| `#presets()` | 读 `task.agentPresets` + 归一化（唯一读取入口） |
| `#persistPresets(next)` | `settings.set("task.agentPresets", next)` + `#notice` + `requestRender` |
| `#applyPreset(name, mode)` | replace / merge 写 `task.agentModelOverrides`，写 notice |
| `#createPreset(name)` | 以当前覆盖为快照写入 |
| `#renamePreset(from, to)` / `#deletePreset(name)` | 数据操作 |
| `#openPresetDetail(name)` / `#closePresetDetail()` | body 模式切换 |
| `#presetStrip(name)` | 构建 footer chips |
| `#renderPreset(width, rows)` | 详情渲染（复用 `#resolvePatterns` 做 selector 预览） |
| `#handlePresetInput(data)` | 名字输入（Enter/Esc/字符），与 `#handleCreateInput` 同构 |
| `#activePresetName()` | 派生 active preset |

接线点（改动最小化）：

- `#buildSidebar()` L350：`#presets()` 非空时插入 `separator` + `Presets` 分区 + 每个预设行 + `new-preset` 行。
- `#renderSidebar()` L1170：`icon` 按 kind 取；annotation 为数量。
- `#moveSidebar()` L1053：`kind === "preset"` 时不调用 `#buildRows()`（body 由 `#presetDetail` 决定），但需要清空搜索态。
- `handleInput()` L838：在 `#createActive` 分支旁增加 `#presetInputMode` 分支；`#focus === "scope"` 的 Enter 分支对 `new-preset` / `preset` 分别调用 `#beginPresetCreate()` / `#openPresetDetail(name)`。
- `#renderFooter()` L1405 / `#footerHint()` L1382：preset 详情与确认态的文案与 chips。
- `render()` L1441：body 分支顺序改为 `#createActive` → `#presetDetail` → `#assigning` → `#renderList`。
- `#statusRow()` L1205：preset 详情态显示 `Preset <name> · N agents`。
- `#routeMouseEvent()` L1077：侧栏 `kind === "preset"` 打开详情；`new-preset` 进入新建流程。

### 6.4 `packages/coding-agent/test/agents-hub.test.ts`（+ `test/acp-lazy-startup.test.ts`）

新增 `describe("AgentsHub presets")`，用例：

1. 侧栏渲染 `Presets` 分区、名称与数量。
2. Enter 打开 preset 详情，body 列出 `agent → selector`。
3. `apply`（replace）后 `settings.get("task.agentModelOverrides")` **等于** preset 映射（含清掉一个预设未列出的旧条目）。
4. `merge` 后保留未列出的 agent。
5. 当前覆盖与某 preset 一致时该 preset 显示 active。
6. `+ New preset…` 输入名字后快照当前覆盖；Esc 取消不写入。
7. 重名新建被拒绝且不覆盖已有预设。
8. `rename` / `delete` 后 `task.agentPresets` 的键集合正确。

沿用现有测试脚手架：`Settings.isolated()`、`vi.spyOn(discovery, "discoverAgents")`、`hub.handleInput("\r")` / `"\x1b[C"` / 逐字符 `type()`、`hub.render(120)` 去 ANSI 后断言。

另在 `test/acp-lazy-startup.test.ts` 的 "honors explicit host-defaulted and todo settings for protocol hosts" 用例（L240-288）的 `explicit` 映射里加 `"task.agentPresets": { peak: { scout: "anthropic/claude-sonnet-4-5" } }`，确保协议宿主启动不会把它打回默认值。

### 6.5 文档与 changelog

- `docs/settings.md`：在 Task/Agents 表（`task.agentAdvisor` 行附近，L389）补 `task.agentPresets` 一行，类型 `record`，默认 `{}`，说明"命名分组，由 `/agents` 面板维护；应用时展开写入 `task.agentModelOverrides`"。
- `docs/task-agent-discovery.md`：在 "Model and structured-output precedence" 一节补一句——预设是 UI 层的命名快照，apply 时展开进 `task.agentModelOverrides`，spawn 优先级链不变。
- `packages/coding-agent/CHANGELOG.md`：`## [Unreleased]` 下按现有格式加一条（`### Added`）。

## 7. 验证计划

```bash
cd packages/coding-agent
bun test test/agents-hub.test.ts      # 契约测试（新增用例 + 原有 9 条）
bun run check                         # oxlint + oxfmt + tsgo（提交门禁）
```

手工验证（CONTRIBUTING 明确要求"自己跑通"）：

```bash
bun run dev                           # 从源码起 omp
# /agents → 侧栏 Presets → + New preset…（此时应快照现有覆盖）
# 切到 offpeak → apply → Esc
omp config get task.agentModelOverrides --json
omp config get task.agentPresets --json
# 派发一个 scout，检查子会话 JSONL 的 model_change 是否为预设值
```

边界用例：空预设 apply（写入 `{}`）；预设引用已删除的 agent；手改 YAML 写入非字符串值；项目 `.omp/config.yml` 与全局同时定义 `task.agentPresets`（面板显示合并结果，写入落全局层——与现有 `settings.set` 行为一致，文档需注明）。

## 8. 分阶段

| 阶段 | 内容 | 预估 |
| --- | --- | --- |
| P0 | schema + `omp config` 往返 + 归一化函数 + 单测 | 0.5d |
| P1 | 侧栏分区 + preset 详情 body + `apply(replace)` + 测试 | 1d |
| P2 | `+ New preset…` / rename / delete + 名字输入 + 重名校验 + 测试 | 1d |
| P3 | merge 模式、active 派生、鼠标分发、docs、changelog、`bun run check` | 0.5d |

每阶段结束都应保持 `bun test test/agents-hub.test.ts` 全绿。

## 9. 风险与替代方案

**风险**

- `agents-hub.ts` 是单文件 1473 行的组件，新增第四种 body 模式会继续抬高状态复杂度。缓解：preset 状态独立成 3 个字段 + 独立渲染函数，不与现有 strip 状态交叉；先写契约测试再改 UI。
- 嵌套 record 没有逐键类型校验（`RecordDef<T>` 泛型 + 仅 JSON/YAML 结构校验）。缓解：`#presets()` 统一归一化，非法值丢弃并提示。
- `settings.set` 写 global 层：项目层定义的预设可以显示但不能"就地编辑到项目层"。这是现有 settings 语义，文档注明即可。
- 上游 PR 门槛：`CONTRIBUTING.md` 要求大型 UI 改动先在 Discord 讨论、PR 需人工写说明并附自测证据。建议先在本 fork 落地，再决定是否上游。

**替代方案（不采用及原因）**

- **预设存独立文件**（如 `~/.omp/agent/presets/*.yml`）：丢掉 settings 分层、`omp config` 与并发合并；面板还要自己实现文件发现与监听。
- **spawn 时按 `activePreset` 间接解析**（`agentPresets[active][agent]` 优先于 `agentModelOverrides`）：需要改动 `structured-subagent.ts`、`vibe/runtime.ts`、`security/coordinator.ts`、`selector-controller.ts` 四处解析路径，优先级与"面板显示值 = 实际值"的一致性都要重新定义，且引入 `activePreset` 状态迁移。收益只是"手改覆盖不被组覆盖"，不值得。
- **预设携带 prewalk/advisor/disabled**：v1 只做 model，record 值形态保留扩展空间（未来可放宽为 `Record<string, string | { model?: string; prewalk?: string; advisor?: string }>`，读时兼容两种形态）。
