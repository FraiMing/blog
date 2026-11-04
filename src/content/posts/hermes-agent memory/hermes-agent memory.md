---
title: hermes-agent memory
published: 2026-10-25
description: '学习完 hermes 的 memory 与 context 源码后做的小笔记总结'
image: ''
tags: [hermes, memmory, context]
category: 'hermes'
draft: false 
lang: ''
---

# Hermes Agent · Memory 与 Context 学习总结


## 一、先看全局： turn 的完整链路

先展示下 turn 的完整链路，后面的小节其实都是链上的一段。

```text
【session 启动，一次性】
  MEMORY.md / USER.md
    └─ load_from_disk() → 冻结快照 → build_system_prompt() → agent._cached_system_prompt
    └─ 同一份字节写进 sessions.system_prompt 

【每个 turn】
  build_turn_context()
    ├─ ① 复制 history，append 本轮 user
    ├─ ② 取 _cached_system_prompt（只读，压缩不碰）
    ├─ ③ 两层门闸 → 估 token（messages 粗估 + system/4）
    ├─ ④ 超阈值 → 压缩循环（≤3 轮）：切 head/middle/tail，middle 交辅助模型摘要
    ├─ ⑤ memory provider：on_turn_start + prefetch
    └─ ⑥ 打包 TurnContext

【调用主模型】
  API(system = active_system_prompt； messages = 压缩后的 messages + prefetch 拼接)
    └─ tool loop …（本章不展开）
    └─ 结束后：conversation_history = 压缩后的 messages（不可逆）
```

## 二、模型分工

**从 API 调用中能看出一个 turn 里有两类独立的模型调用**：

- 辅助模型：负责压缩、编写摘要
- 主模型： 负责回答等主要工作

下面是作者给自己复习的文档查询（`/export`下）：

| 产物 | 是什么 | 给谁看 |
|---|---|---|
| `01_aux_prompt.md` | 摘要模型的完整输入 | 辅助模型 |
| `02_aux_response.md` | 摘要正文 | 拼进 messages |
| `03_main_system.md` | 主模型的 system prompt | 主模型（独立参数） |
| `04_main_messages.md` | 主模型的 messages | 主模型 |
| `05_main_response.md` | 主模型输出 | 人 |

## 三、Memory 的三层载体

### 功能定位

整体的 memory 由下列三层组成，它们相互组装又相互兜底：

| | 载体 | 容量 | 每 turn 成本 | 失效模式 |
|---|---|---|---|---|
| L1 内置 | MEMORY.md / USER.md → system | 固定硬性字符 | 全程都付 | **记错**（但稳定地错） |
| L2 外部 | MemoryProvider（prefetch / 工具） | 后端无上限 | 只付召回那点 | **漏检**（查不到等于没有） |
| L3 会话 | conversation_history | 受窗口限制 | 线性增长 | **遗忘**（被压缩逐渐折叠） |

读法：任何单层都无法同时满足「保真 + 容量 + 成本」。L1 兜 L3 的遗忘，L2 兜 L1 的容量，L3 兜 L2 的漏检。

### 核心实现

1. 一个 Store，两个状态。**
`MemoryStore` 同时维护 live entries 和 `_system_prompt_snapshot`：

- live：`add / replace / remove` 工具调用改它，也写盘
- snapshot：**只在 `load_from_disk()` 时生成一次**，`format_for_system_prompt()` 只读它

2. `MemoryManager` 管 builtin + 选择**至多一个**外部 provider（第二个直接 reject）。

| 方法                    | 插到哪                 | 内容性质                                                    |
| ----------------------- | ---------------------- | ----------------------------------------------------------- |
| `system_prompt_block()` | system，成为组成的一部分                 | 静态（"你接入了 honcho，可以用 memory_search"）             |
| `prefetch(query)`       | **本轮 user 消息内部** | 动态（本轮召回），外包一层 `<memory-context>` + System note 用于说明内容身份，以免真的当成 user 消息 |
| `on_pre_compress()`     | 辅助模型的 prompt      | 别丢的身份事实                                              |
| `get_tool_schemas()`    | tool 消息              | 拉取式（模型主动查）                                        |

关键决策：**会变的内容一律不许进 system**，因为 system 是前缀，改它等于全段 cache 作废。

3. conversation_history 的修改不可逆。
进场 `list(history)` + append，只增不减；出场 `conversation_history = list(messages)`。
压缩结果覆盖回历史，**不可逆**。

### 漏洞与优化
- **prefetch 拼进 user 消息，会跟着这一轮被持久化** →  不要用裸文本拼接承载结构性信息。把注入内容做成结构化的 content part（如 `{"type":"memory_context", "items":[...]}`），持久化时按 type 过滤
- **记忆的写入时机依赖模型自觉 ** → 把"记忆抽取"从"压缩时顺手做"改成独立的、可重放的离线任务：以 session 为输入批量抽取

## 四、system 的组装与冻结

把 system prompt 读盘组装后冻结起来，并保证这个参数**在有效期内一个字节都不变**。这样的操作是在帮你买缓存。

### 核心实现

一共四步：

① 读盘并切条目：load_from_disk()

② 冻结快照：`_sanitize_entries_for_snapshot` + `_render_block`

③ 拼成整段：prompt_parts.append(...)

④ 缓存到 agent：restore_or_build_system_prompt()



a）**④ 为什么"恢复"排在"构建"前面：**
`restore_or_build_system_prompt` 先查 `sessions.system_prompt` 这一列，能读回就**原样复用**，读不到才 build 并写回。因为恢复一个已有 session 时，你要的是**当时那份字节**，不是"按现在的配置重新算一份"——后者哪怕内容逻辑相同，也可能因排序/环境信息抖动而前缀不一致。

b）**缓存的账：**
策略是单一布局 `system_and_3`：最多 4 个 `cache_control` 断点（system + 最后 3 条非 system），统一 TTL，docstring 估算省 ~75% 输入成本。机制是**前缀累积**：前缀字节要一致且命中位置连续，所以 system 一变，**它后面所有断点全部作废**。

衍生出有这样一条原则：**system 的第一目标是字节稳定，不是信息最新。**

c）**system 的有效期不是整个 session**

system 并非完全在该 session 中不变，否则没必要单独载入它

它常终止于三件事：
1. 下一次 build（新 session / 换模型 provider / DB 行坏了/ 压缩后rebuild）
2. failover 改写 `Model:` / `Provider:` 行
3. 进程结束（内存缓存消失）

压缩时顺手 rebuild 是出于压缩这一行为会打断 cache，那反正 cache 已经断了顺手做不会产生什么成本

### 漏洞与优化
这一节如果考虑漏洞更多是产品策略与算法上的的优缺，例如改 MEMORY.md 后同一个 session 内不生效是提高稳定性而舍弃一些更新

## 五、Context 的组装

`build_turn_context`把「磁盘配置 + 存下来的历史 + 本轮输入」变成「一次 API 调用的两个入参」

大概是这样一个流程：append user → 取 cached system → 两层门闸 → 压缩循环 → prefetch → 打包 TurnContext。

### 核心实现

1. **先 append 再估 token**：否则漏算本轮输入，而它常是历史里最长的一条。

2. **压缩在 prefetch 之前**：否则召回内容会躺在 messages 里被当成 middle 摘要掉，一次性信息被永久写进历史。

3. **压缩后回写 `conversation_history`**：压缩不只改本次 wire，还改了"以后每次启动读到的那个东西"。

4. 来看看两层门闸：
```python
if len(messages) > protect_first_n + protect_last_n + 1:  # 门 1：纯计数 O(1)
	# 门 1 的含义是切完 head/tail 后 middle 非空
    return True
return estimate_tokens_rough(messages) >= threshold_tokens  # 门 2：字符统计 O(n)
```
它们用于省完整估算的成本

5. **TurnContext 的 11 个字段**分四类去向：wire（messages / active_system_prompt）、回写（conversation_history）、定位（current_turn_user_idx / task_id / turn_id）、注入（ext_prefetch_cache / plugin_user_context）。

## 六、Context 的压缩

上下文快撑爆时，把中间轮次交给辅助模型做成结构化摘要，头（system + 首条）和尾（最近几轮）原样保留。**它是唯一允许改历史的入口**，也是唯一愿意付 cache 代价的时刻。

### 核心实现

1. 按配置索引切割

切割不可能只按照感觉走，需要合理的算法来决策

执行一次压缩：head 保留 + middle 变摘要 + tail 保留
```python
head   = messages[:protect_first_n] # 源码为3
tail   = messages[-protect_last_n:] # 源码为20
middle = messages[protect_first_n:-protect_last_n]
```
步骤：
1. 不够长 / 没超阈值 → 原样返回
2. 切开 head / middle / tail
3. middle 序列化，可选拼上 provider_hint（身份事实）
4. 组 prompt，调 summarize_fn 拿摘要
5. 用 SUMMARY_PREFIX + 摘要 + END 包成一条 user 消息插回去
（head + [summary_msg] + tail）

当然不会只按条数切，还需要考虑 token 等，配置只是保下限

2. 辅助模型提示词

这里不是很想粘出来，作者详 prompt_template.py 与 02_middle摘要，其中有完善的说明

但是我们可以看看实现：
```
middle ──┬─ provider_hint # 这些部分详见 prompt_template.py
         ├─ SUMMARIZER_PREAMBLE
         ├─ FIRST_PASS_INSTRUCTIONS
         └─ _template_sections
              ↓  拼成一条 user prompt
        压缩prompt.md
              ↓  辅助模型（summarize_fn）
           摘要.md
              ↓  + SUMMARY_PREFIX + END_MARKER
     包装后的摘要消息.md（role=user）
              ↓
     head + 它 + tail = 压缩完的上下文
```

### 漏洞与优化

- **压缩不可逆，且没有原文归档** → 摘要里为每段保留**可回查的指针**（turn 范围 / 消息 id / 归档路径）。原文落到 L2 或冷存储，摘要只写"发生了什么 + 去哪查"
- **没有摘要质量校验** → 压缩前从 L1/L2 取一份**关键事实清单**（身份、目标、硬约束），压缩后做一次轻量核对（关键词命中/小模型判定），缺了就把这些事实原文追加到摘要末尾

## 问题速查

稍微粘些 AI 给出的面试问题，有兴趣看就看吧

**Q1：为什么 system 不放在 messages 里？**
协议上 system 是独立顶层参数；工程上更本质：两者**变更节奏不同**（system 每 session 一次，messages 每轮追加）。放一起的话，每加一个改 messages 的功能就要补一个"别动 system"的特判；分开存放把这条约束变成"物理不可能"。

**Q2：记忆为什么要"冻结"？**
为了让 system 的字节在 session 内完全不变，从而让服务端前缀缓存持续命中。冻结的代价是"本轮写入的记忆本轮看不到"，换来的是整个前缀不失效。**这是拿"记忆新鲜度"换"钱和延迟"。**

**Q3：那缓存到底什么时候断？**
三种：下一次 build（新 session / 换模型 / DB 行坏）、failover 改写身份行、进程结束。压缩是唯一被承认的"主动断"的时刻。

**Q4：压缩为什么只动 messages 不动 system？**
压缩的本质是"对数据做有损摘要"。数据可以改写，配置不能摘要——你没法"把 system 压短一点"，那等于改系统定义。要换 system 只能**重建**，这是完全不同的动作。

**Q5：摘要为什么要包一层长前缀？**
① system 冻结，动态护栏不能放进去；② 护栏必须紧贴摘要才有局部语义。同一条消息里的"这是引用"比 system 里隔几千字的"仅供参考"有效得多。

**Q6：多轮压缩怎么不把信息磨没？**
`_previous_summary` + `ITERATE_INSTRUCTIONS`：把"旧摘要 + 新原文"重新整合，而不是"对摘要再摘要"。同时 `protect_first_n` 在首次压缩后 decay 到 0，避免早期消息不朽。

**Q7：压缩最大的代价是什么？**
两份：一次摘要调用的钱（demo 实测一条摘要消息 1049 token），以及**不可逆的信息损耗**（边界错位、细节漂移）。所以压缩只在接近窗口上限时做，而且必须配护栏。

**Q8：这套系统最脆的一环？**
`session.system_prompt` 那一行。它坏了会让 `restore` 退化成每轮 rebuild，缓存永久 miss，成本翻倍且**没有任何功能报错**——故障表现为账单变贵。

**Q9：三层记忆能不能砍成一层？**
不能，除非你能同时满足保真、容量、成本。L1 保真但装不下，L2 装得下但不保证找得到，L3 精确但会过期。三层是三个失败模式的互相兜底，不是功能堆叠。

**Q10：如果让你重做，最先改什么？**
三件事：① 给消息稳定 id，消灭"位置型句柄"；② 压缩摘要带可回查指针，把"有损删除"变成"有损索引"；③ 启动时做参数自检（floor < threshold），把"配错了也能跑"变成"配错了跑不起来"。

## 尾声

这篇文章主要是讲讲 hermes agent 的 memory 与 context 压缩部分的工作逻辑以及实现核心

总体来讲 memory 与 context 的主要设计还是在算法层面，怎么记、怎么存、怎么压以及何时做都取决于算法设计

而这些算法、参数设计是经过 AI 前言工程师们创新与检验过的，日常更多只要抄框架然后定制化改就行，大的框架与参数没必要自己乱改

Thanks for Reading


