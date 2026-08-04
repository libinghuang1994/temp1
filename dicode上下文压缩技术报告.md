# DiCode 上下文压缩技术报告

## 1. 报告摘要

DiCode 的上下文压缩不是“把聊天记录改写成一段短文本”这么简单，而是一套围绕模型请求边界建立的上下文管理工程。它的核心目标是：

1. 在接近模型上下文窗口时释放足够空间；
2. 保留用户最新意图、未完成工作、失败诊断、工具调用配对和文件状态；
3. 不物理删除可恢复、可审计的原始历史；
4. 对手动压缩、自动压缩和 Provider overflow 使用统一但可区分的编排；
5. 首次压缩尽量将模型请求降到上下文窗口的约 10%；
6. 没有新增用户意图时，拒绝无收益的连续压缩，避免额外摘要模型调用；
7. 保留 Legacy 路径作为运行时回滚和历史兼容依赖。

当前实现采用 V2 结构化压缩流水线，并在此基础上增加了动态工具输出预算。固定的 2,048-token 工具结果总额度已经取消；新额度依据模型实际上下文窗口和本次请求中不可压缩的固定开销动态计算。

---

## 2. 问题背景

代码智能体的上下文不同于普通聊天。一次任务可能同时包含：

```text
用户需求
  + 系统提示词
  + 工具定义
  + 多轮 assistant 推理和回答
  + tool_use / tool_result 配对
  + 大文件读取结果
  + 命令输出和测试失败
  + TODO、模式和工作流状态
  + 修改过的文件和恢复信息
  + Provider cache / reasoning / image 开销
```

其中最容易膨胀的是工具结果。例如读取四个大文件时，原始工具正文可能占据数万 token。单纯生成摘要仍可能留下大量近期工具输出，因此出现“压缩前 67%，压缩后仍有 30%”的现象。

但如果粗暴删除工具内容，又会产生另一类问题：

```text
删除失败日志       -> 模型不知道为什么测试失败
删除工具调用 ID    -> tool_use / tool_result 配对失效
删除最新用户要求   -> 模型继续执行过期目标
删除修改文件证据   -> 模型重复修改或错误回滚
覆盖磁盘历史       -> 无法恢复、rewind 或审计
连续重复摘要       -> 产生费用，但上下文几乎不下降
```

因此，DiCode 将“历史事实”和“本次模型真正看到的内容”分离处理。

---

## 3. 三种历史视图

### 3.1 总体关系

```text
                         持久化
                    api_conversation_history.json
                              ^
                              |
                              |
  工具/用户/模型事件 ---> Canonical API History
                              |
                              | summary / condenseParent /
                              | truncationParent / 投影裁剪
                              v
                    Effective Request Projection
                              |
                              | Provider 消息格式转换
                              v
                       实际模型 API 请求


  用户可见事件 ------> UI History ------> ui_messages.json / Webview
                           ^
                           |
                    lifecycle ID 关联
```

### 3.2 Canonical API History

Canonical history 是 DiCode 维护的任务事实记录，典型持久化位置为：

```text
<globalStorage>/tasks/<taskId>/api_conversation_history.json
```

它保存恢复、继续、rewind、兼容旧任务所需的历史。压缩不会仅为了减少 token 而物理删除这里的原始消息。

### 3.3 Effective Request Projection

Effective projection 是从 Canonical history 派生的临时模型请求视图。摘要边界以前的旧消息、被截断的工具正文或非当前 truncation 分支可以继续存在于 Canonical history 中，但不会完整发送给模型。

可将两者理解为：

```text
Canonical history = 账本，记录发生过什么
Request projection = 本次工作台，只摆放模型当前需要看到什么
```

### 3.4 UI History

UI history 面向用户展示工具执行、回答、错误、摘要和压缩生命周期。它与 API history 有关联，但不是模型上下文的唯一来源，也不能用 UI 中显示的消息数量直接推断实际请求 token。

---

## 4. V2 总体架构

```text
             +----------------------+
             |  压缩入口             |
             | manual / automatic /  |
             | provider overflow     |
             +-----------+----------+
                         |
                         v
             +----------------------+
             | Rollout / UAT 路由    |
             +-----+------------+---+
                   |            |
              Legacy 路径       V2 路径
                   |            |
                   |            v
                   |   +----------------------+
                   |   | 构造有效请求投影      |
                   |   +----------+-----------+
                   |              |
                   |              v
                   |   +----------------------+
                   |   | Token usage snapshot |
                   |   +----------+-----------+
                   |              |
                   |              v
                   |   +----------------------+
                   |   | Budget decision      |
                   |   +----------+-----------+
                   |              |
                   |              v
                   |   +----------------------+
                   |   | Thrash / repeat guard|
                   |   +----------+-----------+
                   |              |
                   |       automatic 可先尝试
                   |              v
                   |   +----------------------+
                   |   | 旧工具结果普通裁剪    |
                   |   +----------+-----------+
                   |              |
                   |         重新计数
                   |              |
                   |       +------+------+
                   |       |预算已恢复？ |
                   |       +--+--------+-+
                   |        是|        |否
                   |          |        v
                   |          |  +----------------+
                   |          |  | 近期完整轮次选择|
                   |          |  +-------+--------+
                   |          |          |
                   |          |          v
                   |          |  +----------------+
                   |          |  | 结构化摘要旧历史|
                   |          |  +-------+--------+
                   |          |          |
                   |          |          v
                   |          |  +----------------+
                   |          |  | 重注入持久/保护态|
                   |          |  +-------+--------+
                   |          |          |
                   |          |          v
                   |          |  +----------------+
                   |          |  | 动态工具输出预算|
                   |          |  +-------+--------+
                   |          |          |
                   +----------+----------+
                              |
                              v
                    重计数、提交兼容边界
                              |
                              v
                        下一次模型请求
```

V2 没有替换整个消息系统，而是把计数、预算判断、裁剪、摘要、状态保护、回退和生命周期拆成可独立测试的组件。

---

## 5. Token 计数与触发策略

### 5.1 为什么不能只看 UI 百分比

UI 显示的 usage 通常来自上一轮 Provider 返回值，而下一轮请求还会新增工具结果、用户反馈和环境信息。如果每次又用本地 tokenizer 重算全部历史，不同 tokenizer 或 Provider cache 语义可能导致 UI 与触发器不一致。

DiCode 使用 Usage 锚定策略：

```text
下一轮估算输入
    = 最近一次 Provider 实际 Usage
    + 该模型响应以后新增消息的本地估算
```

对于同一响应中的并行工具调用，边界按 response ID 回溯到该响应的第一个 assistant 分片，再累计其间及之后的所有工具结果，避免漏算穿插在多个 assistant 分片间的结果。

### 5.2 输入预算

```text
contextWindow = 模型上下文窗口
outputReserve = 输出 token 预留
buffer        = contextWindow * 10%

hardInputAllowance
    = contextWindow - outputReserve - buffer
```

触发来源分为：

```text
soft threshold     用户配置的自动压缩百分比达到阈值
hard input limit   计数输入超过安全输入额度
provider overflow  Provider 明确返回上下文超限
manual             用户主动执行压缩
```

Provider overflow 是独立事实，不会被伪装成本地阈值触发。

---

## 6. 两层工具输出治理

工具输出治理必须区分两个阶段，它们的目的和保护范围不同。

### 6.1 第一层：摘要前的旧工具结果普通裁剪

自动压缩首先检查较旧、已完成、可安全移除正文的只读工具结果，例如：

```text
read_file
search_files
codebase_search
list_files
file_outline
list_code_definition_names
```

满足条件的旧正文替换为稳定占位符：

```text
[Old tool result content removed to reduce context size]
```

此阶段特点：

```text
作用位置       Effective projection
Canonical 历史 不修改
最近保护范围   最近两个用户轮次
最小预期收益   1024 tokens
工具配对       保留 tool_use ID 和 tool_result ID
正文处理       整段替换为占位符
```

以下内容不会被普通裁剪：

```text
pending / incomplete 工具
orphan 或重复 tool ID
错误结果
失败诊断
修改文件证据
非文本/媒体结果
未知工具
写文件、修改状态等有副作用工具
Skill、MCP 和需要继续的自定义工具状态
最近两个用户轮次内的结果
```

裁剪后必须重新计数。只有实际预算恢复时，才可以跳过摘要。

### 6.2 第二层：摘要后的动态头尾截断

摘要完成后，近期 replay 或保护态中仍可能包含大工具结果。此阶段不再把它们全部替换掉，而是按动态共享总预算截断中间、保留头尾。

当前可做有界文本截断的工具包括：

```text
读取/搜索/列表工具
execute_command
read_command_output
access_mcp_resource
```

截断形式：

```text
原始结果：

  [开头................................................中间................................................结尾]

模型投影：

  [开头.....................]

  [Tool output truncated: approximately N tokens omitted.
   Re-run this tool with a narrower query if more detail is needed.]

  [.....................结尾]
```

头尾分配约为：

```text
可用正文空间 = 总字节额度 - 截断标记字节
头部          = floor(可用正文空间 * 75%)
尾部          = 剩余 25%
```

保留较多头部是因为文件声明、命令名称和初始错误上下文通常位于前方；保留尾部则用于保存测试总结、退出状态和最终错误。截断按 UTF-8 字节边界进行，不会切断多字节字符。

同一请求有多个候选结果时使用共享总预算，并向较新的结果赋予更高权重：

```text
旧结果  <------------------------------>  新结果
权重 1              2              3              4
较少额度                                           较多额度
```

失败证据、修改文件证据、非文本结果、错误结果和无法确认配对的结果仍保持原样。

---

## 7. 动态工具输出预算

### 7.1 固定 2048 的问题

早期实验使用固定共享额度：

```text
maxTotalToolOutputTokens = 2048
```

确定性测试边界按 4 bytes/token 估算，因此无论是一读取、四读取还是读取加命令，工具结果总量都表现为固定的：

```text
2048 tokens * 4 bytes/token = 8192 bytes
```

这个常量适合作为诊断安全值，却不适合作为所有模型和所有请求的最终策略：

```text
32K 模型窗口  与  128K / 200K / 1M 窗口不应共享同一绝对额度
系统提示词短  与  工具定义很多的请求不应共享同一剩余额度
固定开销很高  时继续保留 2048 tokens 可能无法达到 10%
固定开销很低  时只留 2048 tokens 又会无谓丢失有用证据
```

### 7.2 当前计算方法

当前常量：

```text
目标请求比例       TARGET = 10%
工具输出最小额度   MIN    = 256 tokens
工具输出最大额度   MAX    = 12,288 tokens
```

第一步，用最小工具额度构造一次真实的最小请求投影：

```text
minimumProjection =
    system prompt
  + tool definitions
  + structured summary
  + recent replay
  + persistent context
  + protected continuation
  + non-tool messages
  + MIN 工具结果
```

第二步，使用实际模型计数能力测量 `minimumProjectionTokens`。

第三步，计算：

```text
targetRequestTokens
    = floor(contextWindowTokens * 0.10)

availableAboveMinimum
    = max(0, targetRequestTokens - minimumProjectionTokens)

toolOutputTokens
    = min(MAX, MIN + availableAboveMinimum)
```

ASCII 示意：

```text
模型上下文窗口 W
+------------------------------------------------------------------+
|                                                                  |
+------------------------------------------------------------------+

压缩目标 10% = T
+-------------+
| fixed |tool |
+-------------+
        ^     ^
        |     |
        |     +-- 动态剩余空间
        +-------- system/tools/summary/replay 等实测固定开销
```

如果固定开销已经超过目标：

```text
目标 T
+-------------+

最小投影
+------------------+
| fixed overhead |M|
+------------------+

结果：targetFeasible = false
工具结果仍保留 MIN = 256 tokens
实际占比允许略高于 10%，而不是破坏关键继续证据
```

### 7.3 设计性质

动态预算是“尽量接近 10%”而不是无条件保证 10%。以下内容可能使目标不可达：

```text
系统提示词和工具定义本身过大
结构化摘要和持久状态超过目标
受保护失败或修改文件证据很多
非文本、未知工具或 MCP 结果不能安全截断
Provider tokenizer 与本地/确定性计数存在差异
```

这种取舍优先保证正确继续任务，其次才是百分比外观。

---

## 8. 结构化摘要与近期回放

### 8.1 近期完整轮次

V2 默认保留最多两个近期完整用户轮次，而不是按任意消息条数切割：

```text
User turn N-2  | assistant | tool_use | tool_result | 可能进入摘要
User turn N-1  | assistant | tool_use | tool_result | 精确 replay
User turn N    | assistant | tool_use | tool_result | 精确 replay
```

轮次选择不得拆散 `tool_use` 与 `tool_result`。最新请求即使较大，也优先保留其精确措辞。

### 8.2 摘要合同

结构化摘要要求覆盖：

```text
objective
user constraints
completed work
pending work
files
commands / tests
failures
decisions
tool state
```

有效 JSON 会被规范化并稳定渲染到现有 summary 文本；非空但格式不合法的输出可降级为有长度限制的纯文本。空摘要、过大摘要或摘要阶段产生工具调用会失败，且不会提交破坏性的历史变更。

### 8.3 持久上下文和受保护继续态

结构化摘要之外，V2 还重注入两类信息：

```text
Persistent context
  - TODO
  - 工作流模式和 spec 范围
  - modified-file 元数据
  - 允许保存的失败类别
  - 活动文件上下文

Protected continuation
  - 精确失败诊断和输入
  - 活动 Skill
  - 最新成功的项目自定义工具状态
  - 未完成工具的 ID、名称和输入
```

如果关键保护态格式错误、包含不能安全表达的必需非文本内容，或超过保护上限，压缩宁可停止，也不静默丢失事实。

---

## 9. 连续压缩与无收益保护

### 9.1 自动压缩 thrash guard

“有效释放”满足至少一个条件：

```text
释放 >= 1024 tokens 且 >= 压缩前输入的 5%
或
共享输入预算已经恢复
```

两个连续自动 no-progress 结果触发 60 秒 cooldown：

```text
automatic attempt #1 -> no progress
automatic attempt #2 -> no progress
                         |
                         v
                    60s cooldown
                         |
        +----------------+----------------+
        |                                 |
   soft threshold                     hard/provider overflow
   摘要前停止                         使用非破坏性 truncation
```

### 9.2 重复手动压缩提前拒绝

压缩成功后，Task 保存成功边界，并检查此后是否出现新的用户意图：

```text
上次成功压缩
      |
      +-- 仅 final assistant 回复
      +-- 仅已接受 attempt_completion
      +-- 无新 user request
              |
              v
       第二次手动压缩提前跳过
       不调用摘要模型
```

以下情况会允许再次压缩：

```text
用户发送新请求
attempt_completion 收到用户反馈
工具失败后用户继续指示
恢复任务后检测到上次边界之后存在新用户意图
```

这个判断在摘要模型调用之前执行，解决了“二次压缩明知无增量仍消耗一次模型请求”的问题。

---

## 10. 失败、回退和兼容策略

```text
V2 选择/初始化失败
    -> 可单次回退 Legacy

V2 摘要请求已经发出并返回失败
    -> 终态失败，不偷偷发起第二次付费摘要

hard limit / provider overflow
    -> 必要时使用现有非破坏性 sliding-window truncation

重新计数失败
    -> 不提交无法证明安全的新历史

手动压缩 after >= before
    -> 丢弃生成结果，Canonical history 保持不变

用户取消
    -> 不提交压缩变更
```

持久化继续使用既有兼容字段：

```text
isSummary
condenseId
condenseParent
truncationId
truncationParent
```

没有为评测或 V2 偶然增加新的必需前后端、Provider、IPC 或持久化字段。Legacy 仍是仓库级运行时回滚依赖，当前未授权删除。

---

## 11. SpecForge 目标 VSIX 实验

### 11.1 实验方式

实验不是直接调用 TypeScript 私有函数，而是让 CLI ExtensionHost 加载目标 VSIX 中的真实 `extension/dist/extension.js`：

```text
固定 SpecForge workspace
          |
          v
Controlled Repro CLI
          |
          v
CLI ExtensionHost
          |
          v
从目标 VSIX 解压的 extension/dist/extension.js
          |
          v
确定性 fake model boundary
          |
          v
记录实际模型请求投影和 compaction lifecycle
```

确定性模型响应只固定“模型下一步返回什么”，并不预先假定压缩后的历史。压缩结果仍由目标 VSIX 的真实代码根据工作区、工具结果和模型返回运行产生。

### 11.2 最终动态预算结果

测试模型窗口为 128,000 tokens，最终精确包 SHA-256：

```text
88350ec742b77f7e443e39efef92924a9b8393dac733f94d4a40d5541da732f3
```

结果：

| 场景            | 压缩前 tokens | 压缩后 tokens | 窗口占比 | 保留工具结果正文 |
| --------------- | ------------: | ------------: | -------: | ---------------: |
| 单次读取        |        96,602 |        12,803 |   10.00% |      1,444 bytes |
| 四次读取        |       109,448 |        12,848 |   10.04% |      5,556 bytes |
| 两次读取 + 命令 |       103,460 |        13,669 |   10.68% |      1,024 bytes |

三组工具正文不再固定为 8,192 bytes，说明工具额度确实由实际固定开销动态决定。

混合组为 10.68%，原因是不可压缩/受保护的最小投影已经超过 10% 目标；算法使用 256-token 工具下限，没有继续破坏关键内容。

### 11.3 连续压缩结果

同一最终包完成首次压缩后连续触发两次手动压缩：

```text
第一次重复点击：model requests 3 -> 3
                   fixture calls 4 -> 4

第二次重复点击：model requests 3 -> 3
                   fixture calls 4 -> 4
```

两次都观察到用户可见响应，但没有越过摘要模型调用边界。

---

## 12. 测试与质量保障

当前相关验证包括：

```text
动态预算单元测试
  - 32K 窗口且固定开销超目标
  - 128K 窗口
  - 200K 窗口
  - 1M 窗口触发最大上限

工具输出测试
  - UTF-8 头尾截断
  - 共享总预算
  - 新结果权重更高
  - 命令输出
  - 失败/修改文件/非文本保护

Task 编排测试
  - manual / automatic / provider overflow
  - 重复手动压缩
  - accepted completion
  - 用户反馈后允许再次压缩
  - 任务恢复后的边界判断

最终源码回归
  - context-management + Task：263/263 通过
  - TypeScript 类型检查通过
  - Prettier 检查通过
  - git diff --check 通过
  - VSIX ZIP 完整性通过
```

---

## 13. 关键设计决策

### 13.1 非破坏性优先

```text
节省模型 token != 删除历史事实
```

压缩主要改变请求投影。Canonical history 继续服务于恢复、rewind、兼容和审计。

### 13.2 正确继续优先于百分比

10% 是目标，不是可以牺牲失败证据和活动工作流状态的硬指标。固定开销超过目标时，报告不可达并保留最小安全信息。

### 13.3 先做确定性裁剪，再做语义摘要

```text
便宜、确定性的旧工具裁剪
              |
              v
         重新计数
              |
         仍然超预算？
          /       \
        否         是
        |          |
      结束      调用摘要模型
```

这样可以减少不必要的摘要调用，也使行为更容易测试。

### 13.4 不复制参考项目的数据模型

工程参考 OpenCode 的旧工具输出裁剪、会话继续和测试思想，但适配到 DiCode 既有消息、工具和持久化模型，没有复制另一套会话存储合同。

### 13.5 不在普通修复中修改冻结接口

压缩改动位于 Task、上下文管理组件和请求投影适配器内部，不借机改变 WebviewMessage、Provider、网络、CLI IPC 或持久化公共合同。

---

## 14. 已知限制与风险

1. **10% 不是绝对保证。** 固定开销或受保护内容过大时，最小安全投影可能超过目标。
2. **计数存在 Provider 差异。** 本地 token 计数、Provider 实际 tokenizer、cache 和 reasoning 口径可能不同。
3. **未知工具保守处理。** 无法证明安全的工具结果不会自动截断，因此插件或 MCP 工具很多时压缩率可能降低。
4. **内容识别使用保护规则。** 失败和修改文件证据依赖结构、类型及文本模式；新增工具输出格式需要配套测试。
5. **最大工具预算为 12,288 tokens。** 超大上下文窗口不会无限增加工具正文，避免 replay 重新主导上下文。
6. **自动 thrash guard 是 task-local 状态。** 部分 cooldown 状态在任务进程重启后不会延续；重复手动压缩边界则可从成功压缩历史恢复。
7. **Legacy 尚不能删除。** 它仍承担运行时回滚、历史读取和评测对照职责。
8. **确定性实验不等同于所有真实 Provider。** 它证明目标 VSIX 调用链和预算逻辑可重复，真实模型仍需观察 tokenizer、系统提示词和工具集合差异。

---

## 15. 运维与诊断建议

当目标机器压缩后占比异常时，建议按以下顺序定位：

```text
1. 确认实际加载的 VSIX SHA-256
2. 确认模型 contextWindow 与输出预留
3. 读取 compaction lifecycle 的 beforeTokens / afterTokens
4. 区分触发源：soft / hard / provider overflow / manual
5. 检查 minimumProjectionTokens 是否已经超过 10%
6. 按工具名统计投影中的 tool_result bytes/tokens
7. 检查 protected reason：failure / modified file / non-text / unknown
8. 检查是否存在新用户意图，确认二次压缩是否应被拒绝
9. 对照 Canonical history，确认只是投影变化而非历史丢失
10. 使用固定工作区和确定性响应复现，再测试真实 Provider
```

建议诊断记录至少包含：

```text
extension SHA
model ID / context window
trigger / mode
before / after / released ratio
targetRequestTokens
minimumProjectionTokens
resolved toolOutputTokens
targetFeasible
按工具名聚合的结果大小
是否调用摘要模型
是否选择 fallback
```

不得把 API key、授权头、真实用户内容或完整敏感工具输出写入可提交的报告和夹具。

---

## 16. 结论

DiCode 上下文压缩工程已经从单一摘要动作演进为一个分层的上下文管理系统：

```text
事实层：Canonical history，完整、兼容、可恢复
投影层：Effective request projection，只发送当前必要内容
治理层：预算、裁剪、摘要、保护态、回退、thrash guard
观测层：usage snapshot、lifecycle、确定性 ExtensionHost 实验
```

当前方案解决了两个直接问题：

```text
首次压缩：根据实际窗口和固定开销，将请求尽量压到约 10%
再次压缩：没有新增用户意图时，在摘要模型调用前拒绝无收益请求
```

它同时保留了工程上更重要的底线：不删除 Canonical history、不拆散工具调用配对、不静默丢失失败和活动状态、不修改冻结接口，并继续保留 Legacy 回滚路径。
