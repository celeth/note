# Knowledge Base

## 观察整合（Observation Consolidation）

- **定义**：在记忆被保留（retained）后，系统（Hindsight）会将相关事实自动整合为“观察（Observation）”——一种**去重的、以证据为基础的信念**，由记忆库在大量记忆之上逐步构建。
- **核心机制**：
  - **去重（Deduplication）**：将重叠/重复事实合并为一条持久观察，避免重复堆积。
  - **证据追踪（Evidence tracking）**：每条观察需引用支持它的源记忆（含精确原文引用），并维护“证明次数（proof count）”。
  - **持续精炼（Continuous refinement）**：新证据可支持/反驳/扩展观察；观察会被**更新而非覆盖**，并保留历史记录。
  - **新鲜度感知（Freshness awareness）**：若存在较新的已保留记忆但尚未完成整合，reflect 会将相关观察标记为**过期（stale）**，在依赖前先用原始事实核验。


## Mission / Directives / Disposition（用于 reflect 的记忆库配置）

记忆库（memory banks）可通过配置来塑造代理在“反思（reflect）”阶段的推理方式。

- **Mission（使命）**：用自然语言定义该记忆库的身份/定位，指导 Hindsight 在反思时优先关注哪些知识，并提供推理上下文。
  - 例："我是一个专注于机器学习的研究助理。我更偏好简单而非最前沿。"

- **Directives（指令）**：必须严格遵守的硬性规则/护栏（合规与安全约束），在任何情况下都不能违反。
  - 例："绝不推荐具体股票"、"始终引用来源"。

- **Disposition（倾向）**：影响推理与解读风格的软性特质（可用 1-5 量表），例如怀疑主义、字面主义、共情等；用于细微调整解释方式。

> 说明：这些设置**只影响 reflect（反思）操作**，**不影响 recall（回忆/检索）**。
## 术语：disposition（英→中）

常见含义（需结合上下文）：
1. 性情 / 气质 / 性格倾向（例：a calm disposition＝沉稳的性情）
2. 倾向 / 意向 / 态度（例：a disposition to help＝乐于助人的倾向）
3. 处置 / 处理 / 安排（尤指资产、案件等）（例：disposition of assets＝资产处置；case disposition＝案件处理结果/结案）
4. 部署 / 布置（较少见，偏正式）（例：troop disposition＝兵力部署）

备注：建议先确认语境（法律/资产/案件/性格/技术字段）再定译。

# Deep Agents：`create_deep_agent` 的 `skills` 参数加载流程（SDK 内部机制总结）

## 结论概览
- `create_deep_agent(..., skills=...)` **不会**在构建阶段就把技能文件内容读进来。
- 它做的是：把 `skills`（技能来源目录列表）装配成 `SkillsMiddleware` 并放入 agent 的 middleware 栈。
- 技能真正的“加载/发现”发生在 agent **首次运行前**（`before_agent` / `abefore_agent`），由 `SkillsMiddleware` 通过 `backend` 去扫描目录、下载 `SKILL.md` 并解析 YAML frontmatter，生成 `skills_metadata` 写入 state。
- 每次模型调用前（`wrap_model_call`），`SkillsMiddleware` 会把“技能清单（元数据+路径）+ progressive disclosure 使用说明”注入 system prompt。技能全文由模型需要时再 `read_file` 读取（按路径）。

---

## 1. `skills` 参数在 `create_deep_agent` 中如何接入加载链路

### 1.1 主 agent 的 middleware 栈装配
位置：`deepagents/graph.py` 的 `create_deep_agent`

核心逻辑（概念）：
1) 构造基础 middleware 列表（例如 `TodoListMiddleware()`）
2) 如果 `skills is not None`：
   - 插入 `SkillsMiddleware(backend=backend, sources=skills)`
3) 继续插入 `FilesystemMiddleware`、`SubAgentMiddleware`、`SummarizationMiddleware` 等

因此：**`skills` 只是触发“启用 SkillsMiddleware”。**

### 1.2 subagent 的 skills
- 如果某个 declarative `SubAgent` spec 自己带 `skills` 字段，则该 subagent 的 middleware 栈里也会插入 `SkillsMiddleware(backend=backend, sources=subagent_skills)`。
- 默认 general-purpose subagent 也会在 `skills` 非空时安装 `SkillsMiddleware`（用于让 GP subagent 同样看到技能库）。

---

## 2. SkillsMiddleware 的“加载”发生在何时：`before_agent` / `abefore_agent`

位置：`deepagents/middleware/skills.py`

### 2.1 只加载一次（跨 turn 复用）
在 `before_agent` / `abefore_agent` 的开头：
- 如果 state 已经含有 `skills_metadata`（即使为空列表），则**直接跳过**加载并返回 `None`。
- 这样在同一会话/同一 checkpoint 的后续 turn 不会重复扫描技能目录。

### 2.2 backend 获取方式
`SkillsMiddleware` 支持两种 backend 形态：
- 直接传入 backend 实例（最常见）
- 传入 backend factory（callable）：middleware 会构造 `ToolRuntime` 以便在运行期解析 backend

### 2.3 加载的产物是“元数据”而不是全文
加载结果写入 state 的关键字段：
- `skills_metadata`: `list[SkillMetadata]`
- 可选 `skills_load_errors`: `list[str]`（用于记录 source 无法列出/权限等错误）

其中 `SkillMetadata` 由 `SKILL.md` 的 YAML frontmatter 解析而来，至少包含：
- `name`
- `description`
- `path`（指向具体的 `.../SKILL.md` 路径）
以及可选字段：
- `license`
- `compatibility`
- `metadata`（dict）
- `allowed_tools`（实验字段）

---

## 3. 每个 source 目录下如何“发现技能”：`ls` -> 下载 `SKILL.md` -> 解析 frontmatter

### 3.1 Source 的语义（非常关键）
`skills` 参数传入的是 **sources（技能库目录）**，不是某个 skill 自身目录。

期望结构（示例）：
```
/skills/                # source（skills 参数里应传这个）
  data-insert/          # skill 目录（ls 时会被识别为 is_dir）
    SKILL.md            # 必须存在且带 YAML frontmatter
  code-review/
    SKILL.md
```

加载算法（概念）：
1) 对每个 `source_path`：
   - 调用 `backend.ls(source_path)` 列出条目
   - 过滤 `is_dir=True` 的条目作为候选 skill 目录
2) 对每个候选 skill 目录拼接 `SKILL.md` 路径：`<skill_dir>/SKILL.md`
3) 批量下载：`backend.download_files([<skill_dir>/SKILL.md, ...])`
4) 对每个下载结果：
   - 若 `error == file_not_found`：认为该子目录不是 skill，跳过（预期行为）
   - 其它 error：记录 warning 并跳过
   - content 解码为 UTF-8
   - 解析 YAML frontmatter（`--- ... ---`）拿到 `name`/`description` 等字段
5) 同名 skill 的覆盖规则：**last-one-wins**
   - sources 按传入顺序处理
   - 若多个 source 提供了相同 `name` 的 skill，后处理的会覆盖前面的

---

## 4. 技能如何呈现给模型：system prompt 注入（progressive disclosure）

每次模型调用前，`SkillsMiddleware` 会：
- 读取 state 中的 `skills_metadata`
- 将其渲染成一段 system prompt 片段（包含技能位置、技能列表、以及“需要时用 read_file 读取 `path`”的说明）
- 追加到 system message（不是替换）

因此：
- 模型默认只看到“技能名称 + 描述 + 读取路径”
- 模型需要技能详细步骤时，应调用 `read_file(file_path=<skill['path']>, limit=1000)` 读取全文
- 这是典型的 progressive disclosure：避免把所有技能全文塞入上下文导致 token 膨胀

---

## 5. Windows/路径注意事项（常见踩坑点）
1) `skills` 参数里的路径是 backend 内部路径，必须用 POSIX 风格（推荐以 `/` 开头）：
   - ✅ `skills=["/skills/"]`
   - ⚠️ `skills=["skills/"]` 有些 backend 可能容忍，但跨环境不稳定
   - ❌ `skills=[r"C:\repo\skills"]`（这是 OS 路径，不是 backend 路径）

2) `FilesystemBackend(root_dir=...)` 的 `root_dir` 才是 OS 路径：
   - 可以是 Windows 路径（推荐 `str(Path(...))`）
   - 相对路径会受当前工作目录（cwd）影响，建议在入口处先 `resolve()` 成绝对路径

3) 默认 backend 为 `StateBackend()`：
   - 不会访问磁盘
   - 使用 skills 时必须在每次 `invoke()` 的 payload 里通过 `files={...}` 注入 `/skills/.../SKILL.md` 等虚拟文件

---

## 6. 一句话总结（流程图式）
1) 调用 `create_deep_agent(skills=[...])`
2) 组装 middleware：插入 `SkillsMiddleware(backend=..., sources=skills)`
3) agent 首次运行前：`SkillsMiddleware.before_agent/abefore_agent`
4) 对每个 source：`backend.ls()` -> 找子目录 -> 下载 `<dir>/SKILL.md` -> 解析 frontmatter -> 汇总到 `skills_metadata`（last-one-wins）
5) 每次模型调用前：把技能清单与使用说明注入 system prompt
6) 需要细节时：模型按路径调用 `read_file` 读取技能全文


# `create_deep_agent` 流程总结（Deep Agents SDK）

> 目标：把 **模型 + 工具 + 默认/可选 middleware（skills、filesystem、subagents、summarization、memory、HITL 等）** 组装成一个可运行的 LangGraph Agent（`CompiledStateGraph`）。

---

## 1. 输入参数与前置处理

### 1.1 解析/构造模型（`model`）
- `model is None`：
  - 发出弃用警告（未来不再支持默认模型）
  - 使用默认 Anthropic 模型 `claude-sonnet-4-6` 构造实例
- 否则：
  - `resolve_model(model)`：支持 `"provider:model"` 字符串或 `BaseChatModel` 实例

### 1.2 选择 HarnessProfile（与模型/提供方匹配）
- `_harness_profile_for_model(model, _model_spec)`
- Profile 会影响：
  - `tool_description_overrides`：改写 tool 描述
  - `excluded_tools`：要排除的工具
  - `extra_middleware`：额外插入的 middleware
  - `excluded_middleware`：过滤掉的 middleware
  - `base_system_prompt` / `system_prompt_suffix`：影响 system prompt 拼装
  - `general_purpose_subagent`：默认通用子代理策略（启用/描述/提示词）

### 1.3 校验 Profile 的 middleware 排除配置
- `_validate_excluded_middleware_config(...)`
- 防止把关键 scaffolding middleware（如 `FilesystemMiddleware` / `SubAgentMiddleware`）过滤掉导致功能失效

### 1.4 工具预处理（仅改描述，不删工具）
- `_apply_tool_description_overrides(tools, profile.tool_description_overrides)`
- 真正的工具“排除”由后续 `_ToolExclusionMiddleware` 统一执行

### 1.5 backend 默认值
- 未提供 `backend` → 默认 `StateBackend()`（内存态文件系统）
- 该 backend 会被 Filesystem/Skills/Memory/SubAgent 等 middleware 共用

---

## 2. 处理 subagents：拆分与补全

对 `subagents` 逐个处理，分为三类：

### 2.1 AsyncSubAgent（异步/远端子代理）
- 判定：spec 中含 `graph_id`
- 收集到 `async_subagents`
- 后续用 `AsyncSubAgentMiddleware` 暴露异步子代理工具（启动/查询/取消等）

### 2.2 CompiledSubAgent（已编译 runnable）
- 判定：spec 中含 `runnable`
- 原样加入 `inline_subagents`，由 `SubAgentMiddleware` 统一路由到 `task` 工具入口

### 2.3 SubAgent（声明式同步子代理）
- 需要补全默认配置并组装该 subagent 的 middleware 栈：
  - 解析子代理自己的 model/profile
  - 权限 `permissions`：优先使用 subagent 自己的，否则继承父级
  - 构建基础 middleware：
    - `TodoListMiddleware()`
    - `FilesystemMiddleware(backend=..., _permissions=...)`
    - `create_summarization_middleware(subagent_model, backend)`
    - `PatchToolCallsMiddleware()`
  - 若 subagent 自己声明了 `skills`：追加 `SkillsMiddleware(backend=..., sources=subagent_skills)`
  - 再追加 subagent 自己的 `middleware`（如有）
  - profile 侧：
    - 插入 profile 的 `extra_middleware`
    - 若 profile 有 `excluded_tools`：追加 `_ToolExclusionMiddleware`
    - 追加 `AnthropicPromptCachingMiddleware`
    - 应用 profile 的 `excluded_middleware` 过滤并校验覆盖情况
  - `interrupt_on`：
    - subagent 未声明 → 继承顶层 `interrupt_on`
    - subagent 声明 → 覆盖顶层设置
  - tools：
    - subagent 未声明 tools → 继承父级 tools（并做描述改写）

---

## 3. 自动注入默认 general-purpose subagent（可被 profile 禁用）
- 若调用者没显式提供名为 `general-purpose` 的同步子代理：
  - 且 profile 未禁用 `general_purpose_subagent.enabled`
  - 则自动构造默认通用子代理并插入 `inline_subagents` 头部
- 若顶层传了 `skills`，默认通用子代理也会安装 `SkillsMiddleware`

---

## 4. 构建主 agent 的 middleware 栈（核心）

### 4.1 Base stack（基础能力）
按顺序组装：
1. `TodoListMiddleware()`
2. **可选** `SkillsMiddleware(backend=..., sources=skills)`（仅当传了 `skills`）
3. `FilesystemMiddleware(backend=..., _permissions=..., custom_tool_descriptions=...)`
4. **可选** `SubAgentMiddleware(...)`（当 `inline_subagents` 非空）
   - 提供 `task` 工具，并把可用 subagent 列表烘焙进描述
5. `create_summarization_middleware(model, backend)`
6. `PatchToolCallsMiddleware()`

### 4.2 Async subagents
- **可选** `AsyncSubAgentMiddleware(async_subagents=...)`（当存在异步子代理）

### 4.3 插入用户 middleware
- `create_deep_agent(middleware=[...])` 会插入在此处

### 4.4 Tail stack（profile、工具排除、缓存、记忆、HITL）
1. profile 的 `extra_middleware`
2. **可选** `_ToolExclusionMiddleware(excluded=profile.excluded_tools)`
3. `AnthropicPromptCachingMiddleware(..., unsupported_model_behavior="ignore")`（无条件插入；非 Anthropic 模型 no-op）
4. **可选** `MemoryMiddleware(backend=..., sources=memory, add_cache_control=True)`（当传了 `memory`）
5. **可选** `HumanInTheLoopMiddleware(interrupt_on=...)`（当传了 `interrupt_on`）

### 4.5 应用 profile 的 excluded_middleware 过滤与校验
- `_apply_excluded_middleware(...)`
- `_verify_excluded_middleware_coverage(...)`：确保排除条目确实命中某个 middleware（防止写错静默失效）

---

## 5. 拼装最终 system prompt

- 先得到 `base_prompt = _apply_profile_prompt(profile, BASE_AGENT_PROMPT)`
- 再融合调用者的 `system_prompt`：
  - `system_prompt is None`：只用 `base_prompt`
  - `system_prompt` 为 `str`：`system_prompt + "\n\n" + base_prompt`
  - `system_prompt` 为 `SystemMessage`：
    - 保留原 content blocks
    - 追加一个 text block 写入 `base_prompt`
    - 用于保留 Anthropic `cache_control` 等控制块

---

## 6. 创建并返回最终 Agent（LangGraph 编译产物）
调用 `langchain.agents.create_agent(...)` 传入：
- `model`
- `system_prompt`
- `tools`（用户工具，已做描述改写）
- `middleware`（完整栈）
- `response_format` / `context_schema` / `checkpointer` / `store` / `debug` / `name` / `cache`
- `state_schema=_DeepAgentState`（messages 用 DeltaChannel，减少 checkpoint 膨胀）
- `transformers=[_subagent_factory]`（用于 subagent run 追踪/标注）

最后 `.with_config(...)` 设置默认运行配置（如 `recursion_limit`、版本元数据等）。

---

## 7. 一句话理解
`create_deep_agent` = **解析模型与 profile → 处理/补全子代理 → 组装 middleware 栈（skills/filesystem/subagents/summarization/cache/memory/HITL）→ 拼装 system prompt → 编译成可运行图并返回**。


# LangChain v1 `create_agent` 解读（middleware / invoke / 注入 / 控制流）

> 文件来源：`langchain-master/libs/langchain_v1/langchain/agents/factory.py`  
> 目标：解释 `create_agent` 如何把 **模型 + 工具 + middleware** 编译成一个可 `invoke/stream` 的 Agent（LangGraph `CompiledStateGraph`），以及 middleware 如何影响运行。

---

## 1. `create_agent` 的产物是什么？

`create_agent(...)` 返回一个 **编译后的 LangGraph 状态图**（`CompiledStateGraph`）。该图维护 agent 的 state（至少包含 `messages`），并执行经典的 agent loop：

1. 调用模型（生成 `AIMessage`，可能带 `tool_calls`）
2. 若有 `tool_calls`：执行工具（生成 `ToolMessage`）
3. 把新消息写回 state
4. 重复直到退出条件满足（无 tool_calls / structured_response 已生成 / return_direct 等）

---

## 2. 初始化阶段：模型、system_prompt、tools、structured output

### 2.1 初始化 model
```python
if isinstance(model, str):
    model = init_chat_model(model)
```
支持 `"provider:model"` 形式，统一为 `BaseChatModel`。

### 2.2 system_prompt 统一为 SystemMessage
```python
if system_prompt is not None:
    system_message = SystemMessage(content=system_prompt)  # 或直接使用传入的 SystemMessage
```

### 2.3 tools 空值处理
```python
if tools is None:
    tools = []
```

### 2.4 structured output（ResponseFormat）策略处理
`response_format` 可为：
- `ToolStrategy / ProviderStrategy / AutoStrategy`
- 或直接传 schema（Pydantic/TypedDict/JSON schema 等）

逻辑要点：
- 原始 schema 会先包成 `AutoStrategy(schema=...)`（保留“自动选择策略”的意图）
- 但为了在创建 agent 时就能计算出“structured output tools”，会将 AutoStrategy 临时转换为 ToolStrategy 来生成工具绑定（`OutputToolBinding`）并记录到 `structured_output_tools`

---

## 3. Middleware 如何影响 `invoke`：三种核心注入/拦截机制

### 3.1 注入工具：`middleware.tools`
`create_agent` 会收集所有 middleware 暴露的 `tools` 字段：

```python
middleware_tools = [t for m in middleware for t in getattr(m, "tools", [])]
available_tools = middleware_tools + regular_tools
```

结论：**推荐把“长期可用工具”放到 `middleware.tools` 或 create_agent(tools=...)**，而不是运行时动态塞进 request.tools（动态工具需要额外处理，见下文）。

### 3.2 拦截/包裹工具执行：`wrap_tool_call` / `awrap_tool_call`
`create_agent` 会收集所有实现了 `wrap_tool_call` 的 middleware，使用 `_chain_tool_call_wrappers` 组合成一个 wrapper 链（责任链）：

- middleware 列表中的第一个 wrapper 是最外层（outermost）
- 请求流：outer -> ... -> inner -> tool
- 响应流：tool -> inner -> ... -> outer

并用 `traceable(...)` 包装每一层，支持 LangSmith tracing。

### 3.3 拦截/包裹模型调用：`wrap_model_call` / `awrap_model_call`
同理，`create_agent` 会将多个 `wrap_model_call` 组合为一个“模型调用责任链”：

- 第一个 middleware 为最外层
- 每层都可以修改 `ModelRequest`（system prompt、tools、response_format、model_settings 等）
- 每层可以返回：
  - `ModelResponse`
  - `AIMessage`
  - `ExtendedModelResponse`（带额外 `Command`）
- 组合逻辑会累计各层产生的 `Command`（inner-first，然后 outer），最终并入 model 节点返回值

---

## 4. ToolNode：工具执行节点如何建立？

### 4.1 provider built-in 工具 vs 客户端工具
`tools` 分两类：
- `dict`：provider 内置工具配置（不进 ToolNode）
- `BaseTool/callable`：需要客户端执行（进 ToolNode）

代码逻辑：
```python
built_in_tools = [t for t in tools if isinstance(t, dict)]
regular_tools = [t for t in tools if not isinstance(t, dict)]
available_tools = middleware_tools + regular_tools
```

### 4.2 何时创建 ToolNode？
```python
tool_node = ToolNode(
    tools=available_tools,
    wrap_tool_call=wrap_tool_call_wrapper,
    awrap_tool_call=awrap_tool_call_wrapper,
) if available_tools or wrap_tool_call_wrapper or awrap_tool_call_wrapper else None
```

结论：
- 只要有客户端工具或有 wrap_tool_call，就会创建 `tools` 节点。
- 这样也支持“动态工具执行”的场景（如果 middleware 在 wrap_tool_call 中自行处理）。

---

## 5. State schema 合并：middleware 可以扩展 state

`create_agent` 会把所有 middleware 的 `state_schema` 与基础 state 合并：

```python
base_state = state_schema if state_schema is not None else AgentState
state_schemas = [*(m.state_schema for m in middleware), base_state]
resolved_state_schema, input_schema, output_schema = _resolve_schemas(state_schemas)
```

要点：
- **base_state 放最后**，同名字段后者覆盖前者。
- 允许调用者传入 `state_schema` 覆盖 middleware 默认字段声明（例如把 messages 变成 DeltaChannel 等）。

---

## 6. 模型节点（model node）：真正调用 LLM 的地方

### 6.1 ModelRequest 构造
`model_node` / `amodel_node` 会构建 `ModelRequest`，其中包含：
- `model`
- `tools=default_tools`
- `system_message`
- `response_format`
- `messages=state["messages"]`
- `state`（全量 state）
- `runtime`（LangGraph runtime）

### 6.2 模型调用的“核心执行函数” `_execute_model_sync/_async`
执行流程：
1. `_get_bound_model(request)`：
   - 合并 tools + structured output tools（如 ToolStrategy）
   - 根据 `ProviderStrategy`/`ToolStrategy`/无结构化输出，决定 `bind_tools` 或 `bind`
   - 检查“未知客户端工具”并在必要时报错（防止 middleware 动态塞了 ToolNode 不认识的 tool）

2. 拼装 messages：
   - 如果 `system_message` 非空，则 `[system_message, *messages]`

3. 调用模型：
   - `model_.invoke(messages)` / `await model_.ainvoke(messages)`

4. `_handle_model_output(output, effective_response_format)`：
   - ToolStrategy：解析 structured tool call 参数，写入 `structured_response`，并插入 ToolMessage
   - ProviderStrategy：直接从模型返回 JSON 解析 structured_response
   - 若解析失败：可能插入错误 ToolMessage 触发重试（取决于 handle_errors 策略）

### 6.3 wrap_model_call 的作用方式
- 若没有 middleware：直接执行 `_execute_model_sync` 返回 `ModelResponse`
- 若有 middleware：走 `wrap_model_call_handler(request, _execute_model_sync)`
- 最后 `_build_commands` 将：
  - `messages`（和可选 `structured_response`）封装成第一个 `Command(update=...)`
  - 再把 middleware 累积出来的 commands append 到列表中

---

## 7. Middleware hooks 作为“图节点”插入（before/after）

对每个 middleware，如果其 hook 被重写，就会增加对应节点：
- `{m.name}.before_agent`
- `{m.name}.before_model`
- `{m.name}.after_model`
- `{m.name}.after_agent`

这些节点用 `RunnableCallable` 包装以兼容 sync/async，并被加入 StateGraph。

---

## 8. 控制流：如何形成 agent loop（关键）

`create_agent` 计算四个关键节点：

- `entry_node`：启动时从 START 进入的节点  
  `before_agent(若有)` -> `before_model(若有)` -> `model`

- `loop_entry_node`：每轮循环入口（工具执行完回到这里）  
  `before_model(若有)` 否则 `model`

- `loop_exit_node`：每轮模型输出后的“出口节点”  
  `after_model(若有)` 否则 `model`

- `exit_node`：最终退出到 `after_agent(若有)` 否则 `END`

### 8.1 model -> tools / end / model（条件边）
使用 `_make_model_to_tools_edge(...)`，规则（简化）：
1. 若 state 有 `jump_to`（middleware 写入）→ 跳转 model/tools/end
2. 找不到 AIMessage → end
3. AIMessage 没 tool_calls → end（经典退出）
4. 有 pending tool_calls（且非 structured tool）→ tools（Send 给 ToolNode）
5. 已有 structured_response → end
6. tool_calls 看似都已被 ToolMessage 覆盖 → 回 model（处理人工注入 ToolMessage 等）

### 8.2 tools -> model / end（条件边）
使用 `_make_tools_to_model_edge(...)`：
1. 若所有客户端工具都 `return_direct=True` → end
2. 若执行了 structured output tool → end
3. 否则 → 回 loop_entry_node（继续让模型处理工具结果）

---

## 9. 责任链 vs 工作流图：两种机制并存

`create_agent` 同时使用两套“扩展机制”：

### 9.1 责任链（类似 Spring Boot 拦截器 / Filter chain）
- `wrap_model_call`：包裹模型调用（可改 request、可短路、可累计 commands）
- `wrap_tool_call`：包裹工具执行（鉴权/审计/重试/动态替换工具等）

### 9.2 图节点 + 边（workflow / state machine）
- before/after hooks 被编译成图节点
- 通过 `jump_to` + `_add_middleware_edge(...)` 能改变图的走向（model/tools/end）

---

## 10. 一句话总结
`create_agent` = **把“模型↔工具循环”做成 LangGraph 状态机**，并把 middleware 以两种方式接入：
1) 责任链式 `wrap_model_call` / `wrap_tool_call`（拦截器/责任链）
2) 图节点式 `before_*` / `after_*`（工作流切面 + 可跳转控制流）


# LangGraph（`langgraph-main/libs/langgraph/langgraph`）目录功能架构解读

> 目标：解读 `langgraph-main/libs/langgraph/langgraph` 这一 Python 包的功能分层与运行时架构（必要时参考上级目录的 README）。
>
> 核心结论：`langgraph` 是一个**低层编排与运行时框架**，把工作流/agent 表达为**带状态的图（Stateful Graph）**，并提供 durable execution（checkpoint/恢复）、interrupt（HITL）、streaming、memory/store 等基础设施。  
> LangChain v1 的 `create_agent` 就是把 agent loop 编译成 LangGraph 的 StateGraph。

---

## 0. 上级 README 给出的定位（概念层）
从 `langgraph-main/README.md` 与 `langgraph-main/libs/langgraph/README.md`：
- LangGraph 是 “Low-level orchestration framework for building stateful agents”
- 强项：durable execution、human-in-the-loop interrupts、memory（短期+长期）、调试与 LangSmith、部署能力等
- 适用：任何长时间运行、可恢复、状态化的 agent/workflow

---

## 1. 顶层目录总览（`langgraph/langgraph/`）
你在 `langgraph-main/libs/langgraph/langgraph/` 看到的主要模块：

- `graph/`：建图 API（StateGraph/MessageGraph）
- `pregel/`：执行引擎（编译图如何跑、如何循环、并发、checkpoint、重试）
- `channels/`：state 字段的“合并/聚合”语义（reducer）
- `runtime.py`：节点运行时注入对象 Runtime（context/store/stream/metadata）
- `stream/`：流式输出与 transformer 管线
- `_internal/`：内部实现（config、runnable 注入、serde、retry、timeout、queue、cache…）
- `types.py / typing.py / constants.py / config.py / errors.py / warnings.py / utils/`：公共类型、常量、配置与工具

---

## 2. `graph/`：建图层（声明式编排 API）
目录：`langgraph/langgraph/graph/*`

从 `graph/__init__.py` 可见对外导出：
- `START`, `END`：图起止节点标识
- `StateGraph`：最常用的带 state 的图构建器
- `MessageGraph`, `MessagesState`, `add_messages`：面向消息流的图（agent 场景常用）

**职责：**
- 提供用户侧 API：`add_node`、`add_edge`、`add_conditional_edges`、`compile`
- 将声明式图结构编译为可运行的 `CompiledStateGraph`

**与 LangChain 的关系：**
- LangChain v1 `create_agent` 内部用 `StateGraph` 组装 model/tools/middleware 节点，再用 conditional edges 形成 agent loop。

---

## 3. `channels/`：状态合并语义（state 如何“增量更新”）
目录：`langgraph/langgraph/channels/*`

典型 channel：
- `last_value.py`：最后写入者 wins
- `binop.py`：二元聚合（例如 `operator.add`）
- `delta.py`：增量/差量通道（常用于减少 checkpoint 膨胀）
- `topic.py` / `named_barrier_value.py`：更复杂的 topic/同步/屏障语义

**职责：**
- 定义每个 state key 在多节点、多步执行后如何 merge 多次写入（writes）
- 这是 durable execution 的基础：允许图在并发/重试/恢复时仍能一致地合并状态。

---

## 4. `pregel/`：执行引擎（编译图如何运行）
目录：`langgraph/langgraph/pregel/*`

关键文件/模块：
- `main.py`：核心入口（`Pregel` / 编译后运行体）
- `protocol.py`：执行协议（invoke/stream/ainvoke/astream 等）
- `_loop.py` / `_runner.py` / `_executor.py`：调度与执行循环（step 驱动、并发）
- `_checkpoint.py`：checkpoint 写入/恢复（durable execution）
- `_retry.py`：重试策略
- `_read.py` / `_write.py` / `_io.py`：state 读写与 I/O 组织
- `remote.py`：远程执行相关（与 server/remote runtime 对接）
- `debug.py` / `_log.py` / `_draw.py`：调试/日志/可视化辅助

**职责一句话：**
> `graph.compile()` 之后，真正负责让图“可持续运行、可中断、可恢复、可流式输出”的就是 `pregel` 引擎。

---

## 5. `runtime.py`：运行时注入（节点为什么能拿到 store/context/stream）
文件：`langgraph/langgraph/runtime.py`

定义：
- `Runtime[ContextT]`：注入给 node/middleware 的运行时对象
- `ExecutionInfo`：checkpoint_id、checkpoint_ns、task_id、thread_id、run_id、node_attempt 等执行元数据
- `ServerInfo`：LangGraph Server 注入的 assistant_id/graph_id/user 等
- `RunControl`：协作式 drain（优雅停止/退出信号）

`Runtime` 常用字段：
- `context`：本次 run 的依赖/上下文
- `store`：跨 run 的持久化 store（memory）
- `stream_writer`：写自定义 stream 的 writer
- `previous`：functional API + checkpointer 时可用的上次返回值
- `execution_info`：执行元信息

并提供：
- `get_runtime()`：从 config 中获取当前 run 的 runtime

---

## 6. `_internal/_runnable.py`：Runnable 适配与参数注入机制（非常关键）
文件：`langgraph/langgraph/_internal/_runnable.py`

### 6.1 `RunnableCallable`
LangGraph 内部提供 `RunnableCallable`（比 LC 的 RunnableLambda 更贴合图执行），主要能力：
- 检查目标函数签名
- 按 `KWARGS_CONFIG_KEYS` 自动注入参数（来自 config/runtime），例如：
  - `config: RunnableConfig`
  - `writer: StreamWriter`
  - `store: BaseStore`
  - `previous`
  - `runtime`
  - `error: NodeError`
- 支持 tracing 上下文设置（`set_config_context`），保证 LangSmith 能形成正确 parent/child run 树

因此，node 可以写成：
```python
def node(state, runtime: Runtime[Ctx]): ...
```
运行时就能拿到 runtime（无需手动传参）。

### 6.2 `coerce_to_runnable`
将 callable/dict/generator 等统一转换成 Runnable，便于统一执行语义与组合。

### 6.3 `RunnableSeq`
LangGraph 内部的 sequence runnable（类似 LCEL RunnableSequence），用于：
- 串联执行步骤
- 在 `stream/astream` 中搭建 transform 管线
- 正确处理 tracing/context

---

## 7. `stream/`：流式输出与 transformer 管线
目录：`langgraph/langgraph/stream/*`

核心内容：
- `run_stream.py`：stream 执行入口
- `_mux.py` / `transformers.py`：多路复用与 transformer 链
- `stream_channel.py`：与 channels 协作的 stream 输出

与 LangChain agent 的关系：
- LangChain v1 `create_agent` 在 `compile(transformers=[ToolCallTransformer, *middleware_transformers, ...])` 时，实质是把 stream transformer 管线接入 LangGraph compiled graph。

---

## 8. 其它基础模块
- `constants.py`：如 `START/END`
- `types.py`：如 `Command`、`Send`、interrupt 等（HITL 与控制流依赖）
- `config.py` + `_internal/_config.py`：`ensure_config/patch_config`、callback manager 获取等
- `errors.py`：`NodeError` 等
- `utils/`：通用工具、config/runnable 辅助

---

## 9. 从“建图”到“运行”的一条主链路（架构串联）
1. 用 `graph/StateGraph` 声明节点与边  
2. 每个 state key 的合并语义由 `channels/` 决定  
3. `compile()` 把图转成 `pregel/` 执行引擎可运行结构  
4. 运行时 `pregel/` 驱动 step 循环、并发、重试、checkpoint/恢复  
5. `runtime.py` + `_internal/_runnable.py` 负责把 runtime/store/writer/config 注入节点函数  
6. `stream/` 将执行过程以流式事件/updates 输出，并允许 transformers 改写输出

---

## 10. 建议的源码阅读顺序（如果要继续深入）
1) `graph/state.py`（StateGraph 的 compile 如何组织结构）  
2) `pregel/main.py`（Compiled graph 如何 invoke/stream）  
3) `pregel/_loop.py`（step 驱动、writes 合并、调度关键逻辑）  
4) `channels/*`（状态字段 reducer/聚合机制）  
5) `_internal/_runnable.py`（注入与 tracing/context 机制）



数据源
  ↓
ETL 清洗
  ↓
文档切分
  ↓
Metadata 标注
  ↓
Embedding
  ↓
向量库
  ↓
全文索引 BM25
  ↓
用户问题
  ↓
Query Rewrite / Expansion
  ↓
Hybrid Search
  ↓
Metadata Filter
  ↓
Rerank
  ↓
Context Compression
  ↓
Prompt 约束
  ↓
LLM 生成
  ↓
答案 + 来源引用
  ↓
日志与评估