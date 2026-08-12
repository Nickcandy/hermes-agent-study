# Hermes Agent 核心链路复习笔记

> 整理自本轮源码学习对话。目标不是覆盖整个 Hermes，而是掌握 Agent 中间最重要的链路：Tool、Skill、MCP、Context、校验、失败恢复与压缩。

## 0. 五分钟复习版

### 最重要的一张图

```text
用户消息
   ↓
组装 Context（系统提示、历史消息、记忆/文件、Skill 索引）
   ↓
清理成 api_messages（只改本次请求副本）
   ↓
调用 LLM
   ├─ 没有 tool_calls → 本轮结束，返回最终文本
   └─ 有 tool_calls
          ↓
      解析和校验参数
          ↓
      权限、审批、副作用与重复调用保护
          ↓
      执行 Hermes Tool / Plugin Tool / MCP Tool
          ↓
      Tool Result 写回 messages
          ↓
      再次调用 LLM，让模型基于结果继续推理
```

### 四个概念不要混

| 概念 | 本质 | Hermes 的主要发现方式 | 真正加载/执行方式 |
|---|---|---|---|
| Tool | 可执行函数 | Core Tool 常驻；Plugin/MCP Tool 可经 BM25 搜索 | 发送结构化参数并执行 |
| Skill | 给 Agent 的工作说明书 | 名称和短描述暴露给 LLM，由 LLM 判断是否需要 | `skill_view` 读取完整 `SKILL.md` |
| Context | 本次请求实际携带的工作集 | 确定性组装为主；部分 Memory/历史可搜索召回 | 进入 `api_messages` |
| MCP | 外部能力接入协议 | 连接 Server 后通过 `tools/list` 发现 Tool | MCP Client 调用远端 Server |

所以它们并不都是“BM25 + LLM 召回”。

- Tool Search：代码执行 BM25，LLM 负责提出查询词并从结果中选择。
- Skill：主要是描述索引 + LLM 判断 + `skill_view` 渐进加载，不是默认 BM25。
- Context：当前消息、系统提示等直接组装；只有 Memory、历史搜索或插件 Context Engine 可能使用 FTS、向量或 LLM。
- MCP：是一种 Tool 的来源和通信协议，本身不是召回算法。

---

## 1. 渐进式披露是什么

渐进式披露的目标是：先让模型知道“有哪些能力”，真正需要时再加载完整定义或正文。

```text
第一层：名字 + 短描述
第二层：完整参数 Schema 或完整 SKILL.md
第三层：真正执行 Tool 或按 Skill 工作
```

这样做主要解决两个问题：

1. Tool Schema 和 Skill 正文都很占 Token。
2. 每轮都改变 Tool 列表或系统提示会破坏 Prompt Cache。

Hermes 因而保留常用 Core Tool，把大量 Plugin/MCP Tool 放到 `tool_search`、`tool_describe`、`tool_call` 三个桥接 Tool 后面；Skill 则先暴露索引，需要时再调用 `skill_view`。

源码入口：

- [`tools/tool_search.py`](../tools/tool_search.py)
- [`tools/skills_tool.py`](../tools/skills_tool.py)
- [`model_tools.py`](../model_tools.py)

---

## 2. Tool Search 到底怎么搜索

### 2.1 搜索对象

Hermes 为每个可延迟 Tool 建立一段搜索文本：

```text
Tool 名称（拆开 snake_case）
+ Tool description
+ 顶层参数名称
```

完整 JSON Schema 正文不会放进索引，避免噪声。

### 2.2 排序算法

`tool_search` 使用代码内实现的 BM25：

```text
LLM 生成 query
   ↓
Hermes 对 Tool Catalog 做 BM25 打分
   ↓
按分数降序取 Top K
   ↓
返回名称、来源和短描述给 LLM
```

默认 `limit=5`，最大值默认 20；可以由调用参数和配置调整。BM25 没有命中时，会退回 Tool 名称子串匹配。

关键代码：[`tools/tool_search.py`](../tools/tool_search.py#L316-L472)。

### 2.3 LLM 在哪里参与

LLM 不负责计算 BM25 分数，也没有第二次 LLM rerank。LLM 的职责是：

1. 根据用户目标生成搜索词。
2. 阅读 Top K 的名称和描述。
3. 必要时换关键词再次搜索。
4. 调用 `tool_describe` 取得完整参数 Schema。
5. 通过 `tool_call` 执行真正的 Tool。

因此准确表述是：

> Hermes Tool Search 是 BM25 检索，LLM 负责查询规划和结果选择，不是 LLM 根据所有 description 直接做全量召回。

---

## 3. Skill 的发现、加载与失败

### 3.1 Skill 不是 Tool

Skill 是一份工作流程说明书，通常由 `SKILL.md` 加引用资料、脚本和模板组成。Tool 是可执行函数。

```text
Skill：告诉 Agent 应该怎么做
Tool：让 Agent 实际做某个动作
```

一份 Skill 可以指导模型连续调用多个 Tool。

### 3.2 Skill 的渐进加载

```text
扫描 Skill 目录
   ↓
只展示 name + description
   ↓
LLM 判断相关性
   ↓
skill_view(name)
   ↓
完整读取 SKILL.md
   ↓
按 SKILL.md 再读取指定 reference/script/template
```

Hermes 这里主要依赖 LLM 阅读短描述后选择，不是默认先跑 BM25，也不是默认向量检索。

### 3.3 Skill 的“重试”是什么意思

Skill 自身不是一次远程执行，所以没有统一的“Skill 执行重试器”。需要区分：

- `skill_view` 读取失败：这是 Tool 调用失败，可以修正名称或路径后重新调用。
- Skill 指导的某个 Tool 失败：遵守 Tool 的失败和重试规则。
- Skill 正文被 Context Compression 清除：再次调用 `skill_view` 重新加载。
- Skill 的方案不可行：LLM 应根据错误更换方案，不是原样无限重试。

---

## 4. MCP 链路

### 4.1 MCP 是什么

MCP 可以先理解成“Agent 调外部能力的标准协议”。

```text
Hermes（MCP Host）
   ↓
Hermes MCP Client
   ↓  stdio / HTTP 等 Transport
外部 MCP Server
   ├─ Tools
   ├─ Resources
   └─ Prompts
```

它解决的是协议统一，不负责替模型思考。

### 4.2 一条完整 Tool 链路

```text
config.yaml 中启用 MCP Server
   ↓
Hermes 建立连接并 initialize
   ↓
调用 tools/list 发现远端 Tool
   ↓
把 Tool Schema 注册进 Hermes Registry
   ↓
大量 MCP Tool 可隐藏到 Tool Search 后面
   ↓
LLM 搜索、查看 Schema、生成参数
   ↓
Hermes MCP Client 发起 tools/call
   ↓
MCP Server 执行业务逻辑
   ↓
结果标准化为 Tool Result，写回对话
```

主要源码：

- [`tools/mcp_tool.py`](../tools/mcp_tool.py)
- [`model_tools.py`](../model_tools.py)
- [`agent/turn_context.py`](../agent/turn_context.py)

### 4.3 MCP 为什么经常让人混乱

因为 MCP Tool 最终也表现成 LLM Tool：都拥有 name、description、input schema 和 result。区别只在执行来源：

```text
Hermes Core Tool → 本进程的 Python 实现
Plugin Tool      → 插件注册的实现
MCP Tool         → 通过协议调用外部 Server
```

进入 Agent Loop 以后，三者尽量走同一套权限、Hook、审批、结果截断和 Guardrail。

---

## 5. Tool 参数校验：运行前和运行后

### 5.1 运行前

Hermes 至少有这些层次：

1. 参数必须是合法 JSON Object；解析失败则不执行。
2. Tool 必须存在，并且属于当前 Session 获准的 Toolset。
3. Tool Search 的延迟 Tool 会检查 required 参数；缺失时把 Schema 返回给模型。
4. 对部分常见类型错误做安全转换，例如：
   - `"42"` → `42`
   - `"true"` → `true`
   - 单值 → Schema 要求的单元素数组
5. 执行权限、审批、安全策略和副作用检查。
6. Tool 自身还会做业务参数校验。

关键代码：

- [`agent/tool_executor.py`](../agent/tool_executor.py)
- [`model_tools.py`](../model_tools.py#L763)
- [`agent/tool_guardrails.py`](../agent/tool_guardrails.py)

### 5.2 参数错误由代码修还是 LLM 修

原则是只做确定、安全的机械修正：

```text
明确可转换的类型错误 → 代码修正
缺参数、语义错误、路径错误 → 返回 Tool Error 给 LLM
LLM 看到错误后生成新的 Tool Call → 下一次迭代重新执行
```

代码不会擅自猜测缺失的业务参数。

### 5.3 运行后有没有统一输出 Schema 校验

没有一个能证明“所有 Tool 业务结果正确”的通用输出校验器。运行后主要处理：

- 捕获异常并形成结构化 Tool Error。
- 标记成功、失败、取消、超时等状态。
- 截断或清理过大的结果。
- 清理不可信内容和潜在提示注入。
- 记录副作用状态和执行轨迹。
- 对文件修改等特定副作用做专门验证。

真正的业务正确性仍需要 Tool 自身验证，或者由 Agent 再调用读取/查询 Tool 检查结果。

---

## 6. Tool 失败与重试规则

需要把三种重试分开：

| 类型 | 谁负责 | 例子 |
|---|---|---|
| Provider/API 重试 | Hermes 代码 | 429、临时网络错误、流中断、凭证切换 |
| Tool 参数修正重试 | LLM | 缺少参数、参数语义错误、路径不存在 |
| Tool 内部重试 | Tool/MCP Client 自己 | 连接恢复、OAuth 更新、数据库锁退避 |

一般的 Tool Error 会作为 `role=tool` 返回给 LLM，模型决定是否：

- 修改参数重试；
- 先调用其他 Tool 获取缺失信息；
- 换一条路径；
- 向用户说明失败。

Hermes 不应原样无限重试。Guardrail 会识别重复调用、重复失败和停滞路径，并要求模型停止不变参数的重试。

### 有副作用时为什么不能盲目重试

例如“创建订单”调用超时，并不能证明订单没有创建：

```text
请求发出 → 服务端创建成功 → 回包途中断线
```

这时副作用状态是 `unknown`。正确动作是先查询真实状态，再决定是否重试；生产系统还应使用 idempotency key。

---

## 7. Tool 的副作用与持久化

Tool 副作用不等于“写进 SQLite”。

| 副作用 | 实际落点 |
|---|---|
| 修改文件 | 文件系统 |
| 启动命令/进程 | 操作系统或远程执行环境 |
| 调用第三方 API | 外部服务 |
| MCP 操作 | MCP Server 背后的系统 |
| 对话和执行轨迹 | Hermes SQLite Session DB |

SQLite 主要持久化 Session、messages、Tool Call/Result、压缩状态等。它记录“发生过什么”，但不会自动回滚外部世界的副作用。

Hermes 会在 Tool Result 上记录类似 `none`、`committed`、`unknown` 的副作用判断，并结合 checkpoint、审批和重放清理，降低中断后重复执行的风险。

主要源码：

- [`hermes_state.py`](../hermes_state.py)
- [`agent/replay_cleanup.py`](../agent/replay_cleanup.py)
- [`agent/tool_executor.py`](../agent/tool_executor.py)

---

## 8. Agent Loop、Iteration Budget 与 `continue`

### 8.1 Python 的 `continue`

和其他语言的循环一样：结束当前这一轮，立即进入下一轮循环。

```python
for tool_call in tool_calls:
    if 参数解析失败:
        写入错误 Tool Result
        continue  # 不执行下面的 Tool，处理下一个 tool_call
```

它不是“继续执行下面代码”，而是“跳过本轮剩余代码”。

### 8.2 Iteration Budget

`IterationBudget.consume()` 是一个带锁的计数器：

```python
if used >= max_total:
    return False
used += 1
return True
```

- 父 Agent 默认 `max_iterations=500`。
- Subagent 默认独立上限 50，可由 `delegation.max_iterations` 配置。
- `execute_code` 的程序化调用可以 refund，不占普通 Tool Loop 预算。

源码：[`agent/iteration_budget.py`](../agent/iteration_budget.py)。

### 8.3 没有 Tool Call 就结束吗

正常情况下是：

```text
LLM 返回 tool_calls → 执行 Tool，再调用 LLM
LLM 没返回 tool_calls → 把 content 当最终回答，本轮结束
```

如果模型还需要根据 Tool Result 推理，它会在 Tool Result 写回后获得下一次 LLM 调用机会。它可以：

- 再调用 Tool；
- 或直接输出最终答案。

但如果某次响应既没有 Tool Call，又只输出普通文本，Host 没有理由自行再调用一次模型。特殊的空响应、截断响应或 thinking-only 响应另有有界恢复逻辑。

---

## 9. `messages` 与 `api_messages`

### 9.1 两者的区别

```text
messages
  = Agent 内部和 SQLite 中保留的完整轨迹

api_messages
  = 每次请求前从 messages 构造的“上行副本”
```

很多清理只作用于 `api_messages`，不修改原始轨迹。这既保留审计记录，也避免为了适配 Provider 而污染历史和 Prompt Cache。

### 9.2 清理孤立 Tool Result

正常配对：

```text
assistant.tool_calls[id=123]
tool.tool_call_id=123
```

如果 Tool Result 找不到对应 Tool Call，它就是孤立结果。Provider 通常会返回 400，因此发 API 前会删除它。

反过来，如果 Tool Call 的 Result 丢失，预调用清理器可以补一个说明结果不可用的占位 Tool Result，使消息结构合法。

源码：[`agent/agent_runtime_helpers.py`](../agent/agent_runtime_helpers.py#L3275)。

### 9.3 清理 thinking-only 消息

thinking-only 指 assistant 消息只有 reasoning/thinking，没有用户可见 content，也没有有效 Tool Call。

部分 Provider 不接受这种历史结构，所以 Hermes：

1. 只在 `api_messages` 副本中删除 thinking-only assistant 消息。
2. 如果删除后出现相邻的两个 user 消息，再把它们合并。
3. 原始 transcript 和 SQLite 仍保留完整痕迹。

源码：[`agent/agent_runtime_helpers.py`](../agent/agent_runtime_helpers.py#L1365)。

---

## 10. Context Compression

### 10.1 什么时候压缩

核心条件：

```text
prompt_tokens >= threshold_tokens
```

构造器配置默认 `threshold_percent=0.50`，但有效值会按模型调整：

- 小于 512K 上下文的模型，比例最低提升到 75%。
- 计算时扣除预留输出 Token。
- 特殊小窗口退化情况约在有效窗口 85% 触发。
- 优先使用 API 返回的真实 `prompt_tokens`；拿不到时估算 messages + Tool schemas。

源码：[`agent/context_compressor.py`](../agent/context_compressor.py#L2456)。

### 10.2 压什么

```text
受保护头部：系统约束和最初消息
压缩中部：较老的对话与 Tool Result
受保护尾部：最近仍在进行的任务
```

执行顺序：

1. 不调用 LLM，先对旧 Tool Result 去重、缩短和截断旧参数。
2. 用 LLM 把中间消息总结为 Goal、Progress、Decisions、Files、Pending Questions、Remaining Work 等结构。
3. 下一次压缩用“旧摘要 + 新旧消息”滚动更新，不从零总结全部历史。

### 10.3 Tool 链路如何不被压坏

压缩后会检查 Tool Call 和 Tool Result 的 ID 配对：

- 删除失去 Tool Call 的旧 Tool Result。
- 移除失去 Result 的旧 Tool Call。
- 保护尚在执行、Result 还没写回的 in-flight Tool Call。

### 10.4 Skill 正文被压掉怎么办

完整 `skill_view` 内容可能很大。压缩后 Hermes 保留确定性标记：

```text
[SKILL_PRUNED: ... reload with skill_view(...)]
```

以后需要该 Skill 时重新加载。这属于 Context 重新加载，不是 Skill 执行重试。

### 10.5 摘要失败

- 鉴权、额度、网络错误：本次压缩终止，原消息保持不变。
- 普通摘要失败：默认生成本地确定性 fallback 摘要，再移除中间窗口。
- 连续两次压缩节省不足 10%：触发防抖退避。
- 手动 `/compress` 可以强制绕过 cooldown 再试。

### 10.6 SQLite 如何保存压缩结果

在同一个 Session 中原子执行：

```text
旧活动消息：active=0, compacted=1
压缩后消息：active=1
```

恢复会话只加载 `active=1`；旧消息没有删除，仍保留在 SQLite 和 FTS 索引中，可以通过 Session Search 找回。

源码：[`hermes_state.py`](../hermes_state.py#L8287)。

---

## 11. 如果换成千万用户的生产级 Agent

Hermes 展示的是一套完整、可读的单 Agent 核心机制。超大规模生产系统通常会加强下面几层。

### 11.1 召回

```text
租户/权限/平台硬过滤
   ↓
BM25 + 向量混合召回
   ↓
规则或小模型 rerank
   ↓
主 LLM 选择
```

不能先召回再做权限过滤，否则可能泄露其他用户的 Tool、Skill 或 Memory 元数据。

### 11.2 Tool 执行

- JSON Schema + 业务语义校验。
- 鉴权、限流、审批和租户隔离。
- 幂等键、调用账本和副作用状态机。
- 超时、熔断、隔离舱和补偿流程。
- 执行后验证真实业务状态，而不只相信 HTTP 200。

### 11.3 Context 与 Memory

- 数据来源、时间、版本和权限标签。
- 热 Context、短期 Session、长期 Memory 分层。
- 摘要不能当作绝对事实；重要状态重新查询真实系统。
- 检索结果记录 provenance，支持审计和删除。

### 11.4 质量保障

- 离线 Recall@K、NDCG、Tool 选择正确率。
- 在线成功率、重试率、重复副作用率、Token 成本和延迟。
- 对检索、规划、执行、验证分别做评测，不能只看最终回答。

---

## 12. 一组容易混淆的结论

1. Tool Search 的 Top K 是 BM25 代码取的；LLM 不负责打分。
2. Skill 默认不是向量召回；LLM 先看短描述，再调用 `skill_view`。
3. Context 不是一种检索算法，而是最终送进模型的工作集。
4. MCP 是协议和能力来源，不是 Agent 的规划器。
5. 参数能安全机械转换时由代码修；缺失语义由 LLM 修正。
6. Tool Error 写回模型，不等于 Hermes 自动重试。
7. Skill 的失败通常落到“加载失败”或“其指导的 Tool 失败”。
8. Provider 重试、Tool 重试和 LLM 改参重试是三套机制。
9. 没有 Tool Call 通常代表本轮模型已经给出最终回答。
10. `api_messages` 是上行副本，清理它不等于删除持久化历史。
11. SQLite 保存执行轨迹，不承载文件、进程和远端 API 的真实副作用。
12. Context Compression 是有损摘要；当前文件、数据库和远端状态仍应重新检查。

---

## 13. 主动回忆题

复习时先遮住答案，尝试自己说明：

1. 为什么 Hermes 不把所有 MCP Tool Schema 每轮都发给模型？
2. Tool Search 中 BM25 和 LLM 分别负责什么？
3. `tool_search → tool_describe → tool_call` 每一步披露了什么？
4. Skill 和 Tool 最核心的区别是什么？
5. 为什么 Skill 正文压缩后必须重新 `skill_view`？
6. MCP Host、Client、Server 分别是谁？
7. 参数 JSON 格式错误后，Tool 会不会实际执行？
8. 哪些参数错误适合代码修，哪些应交给 LLM？
9. 为什么一次超时不能证明有副作用的 Tool 没有成功？
10. `messages` 和 `api_messages` 为什么必须分开？
11. 什么是孤立 Tool Result？为什么 Provider 会拒绝它？
12. thinking-only 消息为什么只在 API 副本中删除？
13. `continue` 在 Tool 循环中具体跳过什么？
14. 没有 Tool Call 时 Agent Loop 为什么结束？
15. Context Compression 为什么同时保护头部和尾部？
16. 压缩后的旧消息在 SQLite 中真的被删除了吗？

---

## 14. 后续源码学习顺序

下一轮可以按下面顺序继续：

1. **Session 恢复链路**：SQLite 如何恢复摘要、活动消息与 Tool 状态。
2. **Context 组装链路**：系统提示、Context Files、Memory 和当前 User Message 如何合并。
3. **Checkpoint 与中断恢复**：什么时候能安全继续，什么时候副作用未知。
4. **Subagent/Delegation**：父子 Agent 的 Context、预算、结果回传与隔离。
5. **Prompt Cache**：为什么 Hermes 极力保持系统提示和历史前缀稳定。
