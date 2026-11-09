---
title: Agent Eval & Test 测评部分
published: 2026-11-09
description: '借 hermes 源码中的部分内容，初步学习 Agent 的测评板块'
image: ''
tags: [Eval, Test]
category: 'hermes'
draft: false
lang: ''
---

## 全局：展示链路定位

先给张链路图：

```text
Runtime（节 4/5）
  conversation_loop:  while 未收敛:
      build messages → apply_cache_markers（第四节）→ API call → tool exec
      ├─ 每步写 trace 事件（第二节）
      └─ 超阈值时压缩历史（第五节）
          │
          │ 跑完 → 冻结成 (messages, trace, workflow)
          ▼
Eval（节 1/2/3），在循环之外，事后执行
  九项 check（第三节）
      ← 判定标准来自不变量（第一节）
      ← 数据来自 trace（第二节）
  → PASS / FAIL → RCA 报告 → 变成新 case（回归集）
```

然后我们留下三句话，用它们记住全局，可以看完后回来：

> **Runtime 负责跑，Trace 负责记，Eval 负责评。** Eval 不在 `while` 里，它消费的是一份冻结的死者（不再调模型）。
>
> **Eval 能断言什么，上限等于埋点埋了什么。** 观测设计必须先于评测设计。
>
> **Eval 部分其实都在守同三件事**：前缀字节稳定、消息 role 布局合法、预算可收敛。九项 check 不过是这三条的可观测投影。

## 测评分层

Agent 的测评主要分为两个层面：Test 单元测试与 Eval 打分评测。我们先用一张简单的表看清区别：

| 类型 | 规模 | 单次成本 | 覆盖 |
| --- | --- | --- | --- |
| 单测 | 成百条测试用例 | 不到 0.1 秒，免费 | 代码分支 |
| Eval | 几条跑出来的 case | 真实跑 loop，几秒到几分钟 + API Key | 行为形状 |

功能上的重点在于：单测的定位是检查代码上写得对不对，Eval 的定位是 Agent 的行为结果跑得好不好。

或者我们再用真实实现来理解。三个映射，是同一件事在两个层的两种写法：

| 单测 | Eval 对应项 | 分工 |
| --- | --- | --- |
| `TestCanCarryMarker`（谁能承载 cache marker） | `system_stable` | 前者测机制对不对，后者测这次运行有没有用对机制 |
| `TestTurnPairPreservation`（切点不拆对） | `role_alternation` | 前者守切点正确，后者守切完模型不会误读 |
| `TestMemoryContextFencing`（`<memory-context>` 围栏 + 剥伪造闭合标签） | system 静态段稳定 | 前者守记忆不被当指令，后者守前缀不变 |

### 为什么我们同时需要单元测试与打分测评？

1. **输入来源不同**：单测输入是人构造的，只能覆盖“我能想到的坏情况”；Eval 输入是真实运行产物。
2. **有些性质只在真实轨迹中存在**：例如 `system_stable` 断言的是六次请求之间的涌现属性；prompt_caching 模块自己不知道自己被调了六次，只有 trace 知道。
3. **成本差三个数量级**：只有 Eval，则一个小的代码问题都需要真实跑 loop 检查才能发现；只有 Test，则无法整体评价成果。这决定了分工——每次 commit 跑单测，每次发版跑 eval 集。

它们之间的不同也注定了：**覆盖率一词在两层含义不同**，没必要拿一者覆盖率去证明另一者。单测讲代码行覆盖，Eval 讲行为空间覆盖，它们都需要定义自己的覆盖维度。

> 接下来我们先讲 Eval。

## 一、Logging / Trace：信号从哪来？

我们先要清楚：拿什么去测评？观测不到的东西，一句测评也写不出来。

有这样一句话分工值得背：**Trace 查因果，Metrics 看 SLO，Log 查细节。**

- Trace 是结构化的事件序列，记录发生的事实与原因
- Log 是文本的事件，叙述发生了哪些事件
- Metrics 是一类“数字 + 时间轴”的结构，表达最近整体发生得怎么样

打分需要字段而不是措辞，所以 Eval 只用 trace。

### 核心实现

#### Python 中 log 的四个角色 + 两个修饰

| 角色 | 大白话 | 实例 |
| --- | --- | --- |
| logger | 说话的出口，按名字分类（点号 = 父子） | `agent.conversation_loop` |
| LogRecord | 那张纸条，`.info()` 那一刻定型所有字段 | 一次调用一个 |
| handler | 收件人（往哪送） | `agent.log` / `errors.log` |
| formatter | 排版规则 | `"%(asctime)s %(levelname)s%(session_tag)s %(name)s: %(message)s"` |
| filter | 闸门（放行 / 拦下） | `_ComponentFilter` |
| level | 音量旋钮 | DEBUG < INFO < WARNING < ERROR |

一条日志的旅程：取出口 → **建纸条（字段此刻定型）** → 查级别 → 沿父链 propagate → 每个 handler 过 filter + formatter → emit。

写日志过程中抛异常不会传回业务代码，走 `handleError()` 打一行 `--- Logging error ---` 就结束——**这条日志静默消失，业务毫无感知**。

#### log 分流靠两种互不相同的机制

| 文件 | 筛选方式 | 结果 |
| --- | --- | --- |
| `agent.log` | root 上无 filter，INFO+ | catch-all 全收 |
| `errors.log` | level 等级在 WARNING+ 以上 | 只收对应 error 事件 |
| `gateway.log` / `gui.log` | **filter 按 logger 名前缀**，且 mode 决定是否创建 | 只收本组件 |

日志分层，更容易找出对应内容与识别出问题。**级别选错 = 信号丢失，且不报错**。

#### trace 的 schema（正例 21 个事件 / 4 种 kind）

| kind | 字段 | 回答什么 |
| --- | --- | --- |
| `api_request` | `tool_names`、`message_roles`、`system_fingerprint` | 这次发出去的上下文长什么样 |
| `api_response` | `content`、`tool_calls` | 模型选了什么工具 |
| `tool_result` | `name`、`args`、结果 | 工具跑出什么 |
| `loop_exit` | `reason`、`budget_used/max`、`interrupted` | 为什么停 |

`message_roles` 每项是 `{i, role, chars, tool_calls}`——**只记角色、长度、有无 tool_calls，不记正文**。体积可控、不落隐私、对不变量够用。

#### log 与 trace 互不可替代

| 维度 | log | trace |
| --- | --- | --- |
| 结构特性 | 隐式结构 | 显式结构（kind + 字段） |
| 内容可靠性 | 低 | 高（按 schema 稳定写字段） |
| 缺失时表现 | 静默不可见 | 可被 `count()` 抓到 |
| 内容详细程度 | 更丰富详细 | 仅限 schema 内字段 |

也因此，审计和打分必须基于 trace 数据，不能基于 log 叙述。

### 漏洞与优化

- **`session_tag` 必须用 LogRecord factory，不能用 Filter**

  logger 上的 filter 不作用于从子 logger 传播上来的记录——Python 的 `Logger.handle()` 在传播时只调 handler 的 filter。所以“在 root logger 上装 filter 统一补字段”这条路从机制上就不成立。

  对比三种方案：root filter（拦不住子 logger）、每 handler 装 filter（能拦，但必须人人记得装）、factory（全局一次，兜底）。factory 赢在把“人人记得做的事”变成了“系统自动做的事”——把约定变成机制。

## 二、不变量：一条断言到底在测什么？

这是整门课的判定标准与基础。单测与 Eval 本质上都在写不变量；这一节说明“什么样的断言值得写、什么样的断言是负债”。

它解决的问题也很实际：CI 里如果有一堆经常无缘无故失败的测试，团队会养成“看见红就重跑”的习惯，于是真正的故障也被淹没了。

### 核心实现

> A test is a change-detector if it fails whenever data that is **expected to change** gets updated.

判据就一句话：**一个本来就该变的东西变了，这个测试会失败吗？** 会 → 它是 change-detector。

#### 三层不变量

| 层 | 形态 | 例子 | 抗漂移性 |
| --- | --- | --- | --- |
| L1 集合关系 | `A ⊆ B`、`A ∩ B = ∅` | `expected_tools ⊆ 实际工具`；`工具 ∩ forbidden = ∅` | 强 |
| L2 条件约束 | `P ⇒ Q`、`a ≤ b` | `grace ⇒ api_calls == budget_max + 1`；`used ≤ max` | 中（关系藏在字段之间） |
| L3 跨事件恒定性 | 所有事件上某量不变 | 6 次 `api_request` 的 system 指纹一致；`tool_names` 集合不变 | 最强，也最难写 |

L3 是 Prompt Cache 那条线的全部内容——它断言的不是一条消息，而是**多次请求之间的一致性**。这也是 Eval 必须看 trace 而不能只看 messages 的原因。

#### 数值断言的合法性判据

```python
assert count <= 4     # ✅ 不变量：4 是 Anthropic 协议断点上限
assert count == 4     # ❌ 快照：现在策略恰好打满
assert api_calls == 7 # ❌ 快照：这是“这轮模型决定调几次”的产物
```

判断依据从来不是“有没有数字”，而是**这个数字变化意味着什么**：协议常量、物理上限、枚举定义的变化 = 违规；运行产物、统计量、路径序列的变化 = 正常。

#### 不变量还可能“太弱”

```python
assert len(messages) > 0        # 有对话就成立，测了个空气
assert isinstance(tools, list)  # 类型断言，不是行为契约
assert True                     # 真仓里真的存在
```

综上，合格的不变量必须同时满足两点：**可被违反**（存在一个真实的坏实现会让它红）+ **违反意味着真问题**。

### 漏洞与优化

- **不变量只被要求“现在能过”，没有被要求“能够失败”**

  一个永远绿的 check，比没有更危险。

  → “测试测试的测试”：给每条 check 配一个必红用例（mutation / canary case），即构造一个已知违反该性质的输入，断言 check 必须返回 False

- **不变量只能覆盖“你想得到”的坏情况**

  更多漏洞都在于“你想不到”。

  → 把线上事故冻结成 fixture 并加进回归集，把经验总结下来，唯有如此

## 三、Harness：把一次运行变成一份成绩单

Eval 的**执行层**。输入是一份冻结的 `messages + trace + workflow`，输出是一个布尔值 + 一份 RCA（根本原因分析）报告。

| 限定 | 含义 | 后果 |
| --- | --- | --- |
| 冻结 | 不重新调模型 | 可复现，这是回归测试的前提 |
| 规则 | 不用 LLM judge | 只能判“可机械判定”的关系 |
| 关系 | 不断言全文快照 | 换模型仍可用 |

所以它的准确名字是 **contract harness（契约检查器）**，不是 quality eval；或者说，它是一道客观题而非主观题。

### 核心实现

九项 check，分两类、四个数据源：

| # | check | 判定 | 数据源 | 防什么 |
| --- | --- | --- | --- | --- |
| 1 | `tools_subset` | `expected − actual == ∅` | trace 的 api_response | 漏调必要工具（不查就答） |
| 2 | `no_forbidden` | `actual ∩ forbidden == ∅` | 同上 | 越权（跑了不该跑的代码 / 写操作） |
| 3 | `steps` | `api_calls ≤ max_steps` | **workflow 字段** | 不收敛（空转烧钱） |
| 4 | `exit` | `reason ∈ allowed_exits` | workflow / loop_exit | 收尾方式不合预期 |
| 5 | `final_text` | 非空 | messages | 没交出东西（卡在 tool_calls） |
| 6 | `role_alternation` | 禁止连续 assistant/system | messages | 布局被破坏 → 前缀不可复用 |
| 7 | `system_stable` | 各次 request 的 system 指纹一致 | trace 的 api_request | 缓存失效 |
| 8 | `tools_frozen` | 各次 request 的 `tool_names` 集合不变 | 同上 | 中途改 schema → 整段前缀作废 |
| 9 | `budget_consistent` | `used ≤ max` 且 `reason` 与 `api_calls` 自洽 | workflow | 账不平（声称耗尽但预算没用完） |

**前五项是“这张卷子该怎么做”（case 期望），后四项是“任何运行都得守的规矩”（循环不变量）。** 把 case 的 `expected_tools` 全删掉，后四项照样跑、照样能红——这是区分两类 check 的最快方法。

`system_stable` 和 `tools_frozen` 是唯一必须读 trace 的两项，也是站 1 存在的理由。

**RCA 三段式输出**：`root_cause`（判定）→ `evidence`（证据）→ `fix_hypothesis`（修复假设）。结构是刻意的，且假设必须可证伪（改完能再跑一次验证）。

### 漏洞与优化

- **RCA 不要设计成规则拼串**

  用大量 if 与字符串拼接的方式展现，这样既不能体现 failed 顺序，也没有优先级体现，甚至会丢失规则项。

  → 把 `root_cause` 从“字符串”改成一个候选集 + 权重的结构。多因故障本来就是常态，应将“拼成一个串”的线性结构转换为树形因果

- **RCA 只能到“直接原因”，回答不了“为什么”**

  它告诉你“模型用了禁用工具”，没回答“为什么模型会去用它”。

  → 这点暂时无法用机器替代，还得是人来判断

## 四、Prompt Caching：4 个断点怎么分

> 从站 4 开始，我们步入 Test 部分。但这些部分并不是对 Test 整体设计实现的一般通讲，而是对于特殊 Test 的讲解。

Anthropic 厂商对于输入支持一种 `cache_control` 字段的缓存策略，最多可以在上下文中打四个断点。

源码设计中提到：

> “4 个缓存控制断点——系统提示 + 最后 3 条非系统消息，全部使用相同的 TTL。在单次会话中的多轮对话中，可将输入令牌成本降低约 75%。”（中文译文）

```text
messages（system + history）
   ↓  发送前
apply_anthropic_cache_control()   ← 深拷贝，只加字段
   ↓
payload → provider
```

它相当于在 messages 建好之后、payload 发出之前告诉 provider“从开头到这个位置的前缀可以缓存”。

因此这个策略在成本控制上是相当的好用，但同时也就要求要有测试来确保它真的用上了。

### 核心实现

#### “≤ 4 断点”靠代码结构保证，不靠断言

```python
if messages[0].get("role") == "system":
    _apply_cache_marker(messages[0], marker, ...)
    breakpoints_used += 1              # ① system 吃掉 1 个
remaining = 4 - breakpoints_used       # ② 剩 3 个
for idx in non_sys[-remaining:]:       # ③ 只取末尾 remaining 条
    _apply_cache_marker(messages[idx], marker, ...)
```

结构性证明：`breakpoints_used ≤ 1` ⇒ `remaining ≥ 3`；切片 `[-remaining:]` 最多产出 `remaining` 个元素；总数 `1 + remaining = 4`。`assert count <= 4` 只是复核，它守的是“将来有人把 `[-remaining:]` 改成 `[-4:]`”这类改动。

#### 为什么是 system + 3 而不是 4

断点语义是“从开头到这里的**前缀**可缓存”，断点越靠后能复用的前缀越长；但只有 4 个名额，牺牲 1 个给永远不变的 system，剩下 3 个作为滚动锚点覆盖最近几轮。这就是 `system_and_3` 策略名字的由来。

#### “选谁”与“怎么挂”必须分离

```text
_can_carry_marker  →  这条有资格占名额吗（选谁）
_apply_cache_marker →  具体怎么挂上去（怎么挂）
```

调用方先按前者筛出全量候选，再按名额切末尾 3 条，最后才真挂。这个算法本身要求两遍扫描：必须先知道“谁有资格”，才能决定“取哪三条”。

合并成一个 `try_apply() -> bool` 只能边试边打，要么打超了要回滚（已改了 content 结构，回滚很脏），要么从前往后打导致标记全落在最老的 3 条上，可复用前缀变短、白白浪费。

代价就是两份规则必须同步。

在注释中发现，历史上真出过 bug：`content` 是 `[dict, "裸字符串"]` 时，gate 看到前面有 dict 判“可承载”，apply 只看最后一项发现不是 dict 就放弃——最终名额被占掉，标记没打上。

### 漏洞与优化

- **同一套规则写了两遍，靠一条测试人工保同步**

  “选谁”和“怎么挂”是两个函数，规则一致性没有任何机制保证，只能靠测试盯着。这是所有“策略 / 机制分离”都要付的税。

  → 让 gate 直接产出挂载目标而不是布尔值：

```python
def _marker_slot(msg, native) -> tuple[str, int] | None:
    """返回要写入的位置；None 表示不可承载。apply 直接用这个结果，不重新判断。"""
```

这样“判断”和“执行”共享同一个谓词，把两份逻辑收成一个返回值的两个消费者。

`native_anthropic` 协议差异：

| `native_anthropic` | 标记位置 | 坑 |
| --- | --- | --- |
| `True` | 顶层 `msg["cache_control"]` 合法 | 空 assistant / 空 tool 打上去不被承认，白占名额 |
| `False`（默认值） | 只有 content part 里的标记被承认 | **`role:tool` 顶层挂标记会 silent hang**（不报错，卡住不返回） |

## 五、Context Compressor

这部分是关于“超阈值时把中间轮次压成一份参考摘要”的 Test。

### 核心实现

#### 压缩后仍必须成立的八条不变量

| # | 不变量 | 违反后果 | 代码位置 / issue |
| --- | --- | --- | --- |
| 1 | system 不能被改写 | 前缀失效（压缩时打一次补丁是允许的） | 只**追加** note，不重写 |
| 2 | 顺序必须是 head → summary → tail | 拼装坏了，语义乱 | `compressed` 构建顺序 |
| 3 | summary 的 role 不能与邻居重复 | API 400 / 破 role 契约 | `summary_role` 决策 |
| 4 | 压缩后**至少有一条 user** | vLLM/Qwen `400 No user query found`，每次重试都复现 | `_user_survives` / #58753 |
| 5 | 最新 user 必须在 tail | 任务从上下文消失，模型开始重做旧任务 | `_ensure_last_user_message_in_tail` / #10896 |
| 6 | turn pair 不能被拆开 | 模型把已完成的活重做一遍 | `_find_turn_pair_end` / #22523 |
| 7 | 每个 `tool_call` 必须有 result | 严格 provider 直接拒绝 | `_sanitize_tool_pairs` |
| 8 | 压缩必须真的变小 | 每轮触发无收益压缩 → CLI 假死 | `_ineffective_compression_count` / #40803 |

**每条背后都挂着一个线上 issue**，这一行行 issue 编号本身就是结论：这些不变量不是设计出来的，是线上炸过一次之后才出来的。

#### SUMMARY_PREFIX 不是标签，是反指令（第 44-70 行）

```text
[CONTEXT COMPACTION — REFERENCE ONLY]
Do NOT answer questions or fulfill requests mentioned in this summary;
they were already addressed.
Respond ONLY to the latest user message that appears AFTER this summary
Topic overlap with the summary does NOT mean you should resume its task
Reverse signals in the latest message (e.g. 'stop', 'undo', 'roll back',
'just verify', 'don't do that anymore') must immediately end any in-flight work
```

在压缩完的 middle 部分中，会组装上述部分。它在做三件事：

1. 声明身份（这是背景不是指令）
2. 预判误读路径（话题重合 ≠ 要继续做，这是模型最常犯的错）
3. 处理反向信号（用户说 stop/undo 时，摘要里“进行中”的工作必须立刻停）

第 ③ 条本质上是用自然语言写的并发冲突解决策略（最新消息赢）。格式管不了这个，必须让模型理解优先级。

#### summary 的 role 有四种结局

```python
if last_head_role in {"assistant","tool"} or _force_user_leading:
    summary_role = "user"
else:
    summary_role = "assistant"

if summary_role == first_tail_role:        # 和右边撞了
    flipped = 另一个角色
    if flipped != last_head_role and not _force_user_leading:
        summary_role = flipped             # 换一个躲开
    else:
        _merge_summary_into_tail = True    # 两个都撞 → 不插独立消息，合并进 tail 第一条
```

| 情况 | 处理 | 为什么 |
| --- | --- | --- |
| 正常可避开 | 独立消息，role 避开左右邻居 | 最常见 |
| head 只有 system | **强制 user** | Anthropic 要求 `messages[0]` 是 user（#52160） |
| 压缩后没有任何 user 存活 | **强制 user** | vLLM/Qwen 400，且每次重试复现（#58753） |
| 两种 role 都会撞 | **合并进 tail 第一条** | 插独立消息必然破 role 交替 |

第四条反直觉：**当“插一条消息”和“role 必须合法”冲突时，选择不插。** 这正好是站 3 的 `role_alternation` 想守的东西，但压缩器是在生成侧就保证，不是等 eval 检出来。

能生成正确的东西，就不要指望事后检查。

#### 401 时选择完全不压缩、原样返回

因为“压了”和“压对了”是两件事，占位摘要属于“压坏了”：中间 500 条消息永久消失且无替代物；凭证没修好，后续每次照样 401；修好后用户拿到的是已经丢了历史且无法恢复的会话。

而“没压”的代价只是这次没腾出空间——下一轮还能再试，数据丢了就永远没了。

> 通用原则：在“不改”和“改坏”之间，永远选“不改”。

### 漏洞与优化

- **摘要合并进 tail 让消息“变脏”**

  - ① frontend 无法单独渲染摘要（只能靠额外的 `_compressed_summary` 元数据）
  - ② 下次压缩时这条消息不能被当作纯 tail 保护（一半历史一半摘要）
  - ③ 摘要的身份声明变弱（不再是独立消息，而是一段嵌入文本，模型更容易忽略边界）

  → 把“脏”显式化：给消息打结构标记（`meta: {kind: "merged_summary", summary_span: [start, end]}`），让渲染层和下一次压缩都能按标记处理，而不是靠解析文本里的分隔符

- **摘要质量没有机械校验**

  → 压缩前用规则抽取关键事实（用户提出的约束条件、已确认的决定、未完成项），压缩后断言这些事实在摘要里仍可检索到（关键词 / 嵌入相似度）。成本低于全量 judge
