# Agent 修复阶段总结与 PPD/TTL 改动计划

更新时间：2026-10-27（新增先验证假设、再定算法的实验计划；GPU/原生 IPC 验收仍待完成）

当前优先级更新：按 [PPD/AMPD 假设验证与改进空间实验设计](./stage5-hypothesis-validation-experiment-design.md)，先完成观测验收和小规模两路径配对实验，确认实际 KV、decode 背景及积压是否造成有代价的错选，再决定新表、窗口指标或拥塞修正。下文阶段 5 的输入组合与实现顺序保留为此前候选，不再视为已经确定要完整实现的算法。既有 Agent/计时/状态语义契约保持不变；本次没有新增 GPU 结果。

## 结论

Agent 与 PPD 的 baseline contract 已经冻结：Agent 侧的 token 预算、reasoning 回放、结构化工具、截断重试和 trace 语义，以及 PPD 侧的 append 计数、calibration loader、matched routing、logical retry、请求级 profiling，均已完成相应验收。后续 serving 对照保持模型、Agent 行为、样本、routing map 和 token 参数不变；详细运行结果继续写入 [实验记录](./agent-ppd-experiment-log.md)，本文只更新当前判断与计划状态。

原始 Dynamic 已完成 offline/live 复现。当前权重下，72 个实际 matched pair 的 score 全为正，另 108 个缺失 cell 默认 PPD，合计 180 个 lookup cell 均选 D-local；关闭 short bypass 后扫描仍全部选 PPD。因此，修 output、QPS 或 context 特征虽然必要，却不能单独让旧表产生 PD 动作。最小 score 接近零，也不能把这一结果推广为“所有负载下 x=1 永远最优”。

阶段 5 第一版已收敛为**实际 KV 与负载状态条件化的 PPD 查表**：用 (N,U_P,U_D) 描述当前请求，用 (B_D,K_D) 描述 D 的 decode 背景，以 P/D 各自的 (Q_tok,Q_att) 描述 prefill 积压；保留原 TTFT/TPOT score 与现有权重。AMPD 前置 TTFT/ITL 路由和精细 F/W/X 预测后置，TTL 当前只留接口。固定 2P:2D 与 affinity，不以提高 P 利用率或 PD 占比为目标。变量推导和当前计划以 [阶段 5 当前方案](./stage5-state-aware-routing-and-ttl-design.md) 为准；[早期问题与备选方案](./stage5-agent-compatible-dynamic-theory.md) 保留研究过程。

c=12 healthy、c=14 operational knee、c=16 stress 是当前模型、样本和轨迹下的实验工作点，不是普适并发上限。固定 map 控制了节点分配，但 live Agent 仍可能走出不同轨迹；严格 serving 因果比较还缺 request/tool-gap replay。现有 retention 只能发现前缀未命中，不能单独判定容量驱逐。将来实现 TTL 仍需 block/LCP 证据与 pin 生命周期保障，Static x=1 的独立验证保留为后续对照，本轮不启动。

“正常时优先 TTFT”未被确认为最终目标。第一版固定原评分是为了控制变量；Agent 完成时间、完整响应/tool-ready、累计 LLM 等待、超时和任务结果仍需联合评价，最终主指标和约束阈值尚待冻结。输入与结果语义已冻结为 [ppd-state-semantics-v1](/root/vllm-ppd/docs/stage5_state_semantics_contract.md)。现已加入默认关闭的 [节点只读观测 API 与就绪记录](/root/vllm-ppd/docs/stage5_readonly_observer.md)，以及独立 pending 账本；CPU 验证已完成，原生 GPU/IPC 和 proxy 身份对账尚未验收，新分类与校准也未实施。

关于文献比较，AMPD 中的 vLLM-Continuum 是 co-located baseline，其 TTFT 收益未必转化为 ITL/SLO 收益；这不等于已经验证“TTL 移植到 PD 后性能退化”。所读 PPD 版本也没有直接给出该对照。PPD 与 TTL 能否互补仍是研究假设，不是已有结论。

现有请求字段见 [serving contract](/root/vllm-ppd/docs/agent_serving_contract.md)，新增状态定义见独立 v1 契约；旧 trace 不自动升级。本次添加了默认关闭的 engine/cache 元数据 hook 与只读接口，但没有改原评分或开启新路由、AMPD gate、TTL，也没有启动 GPU 实验。CPU 开销数据单独写入实验记录，不视为真实 serving 性能。

### 复现使用的代码基线

以下保留已验收的提交标识，不作为当前远端同步状态声明：

- uni-agent：`fix/agent-tool-contract`，`449f80f4383b Add stable routing identity to agent profiles`；
- vllm-ppd：`fix/agent-tool-contract`，contract 提交 `ddffde8fcc9b`，复现基线 `11dccb5d6 Fix small-sample percentiles and TTFT detection`；
- 两仓库的 `agentic-ppd-baseline-v1` 是更早的基线标记，后续修复不应通过移动旧 tag 混入该版本；
- 新实验另行记录实际 commit、dirty diff、配置和 calibration fingerprint。

## 一、Agent 侧已经完成的修改

### 1. Token 和上下文语义

Agent 现在明确区分：

- `max_total_tokens=131072`：整个 episode 累计生成的 completion tokens，不重复累计每轮 prompt；
- `max_tokens_per_turn=16384`：单次模型请求的生成上限；
- `max_context_len=155648`：模型实际上下文窗口。

接近上下文或 generation budget 上限时，Agent 会干净退出，不再发送必然触发 vLLM context overflow 的请求。当前冻结配置使用 Qwen3-30B-A3B-Thinking-2507、BF16 和 `temperature=0`。

### 2. Thinking/reasoning 的记录与回放

Thinking 模型的 assistant `content` 可以为空，因为思考过程位于 `reasoning` 或 `reasoning_content`。Agent 分别记录：

- content；
- reasoning；
- tool calls；
- prompt、completion 和 cached tokens；
- finish reason。

PPD 配置启用 `replay_reasoning=true`。下一轮请求会带回上一轮 reasoning，保留 Agent 计划并减少因省略思考内容造成的前缀差异。但实际可复用长度仍取决于完整 chat template 的 token 序列、token-LCP 和 block 对齐，启用回放不保证生成前缀逐 token 完全一致。

### 3. 截断 tool call 的容错

当前包含两层容错：

1. vLLM 在 `tool_choice=required` 下严格解析被 `max_tokens` 截断的 JSON 并返回 HTTP 400/无可解析 content 时，客户端以 `tool_choice=auto` 重试；
2. 如果 Hermes parser 仍返回不完整 arguments，ReAct 层用 `json.loads` 过滤非法 tool call。

同一 step 的 retry 复用 `client_request_id`，以便 PPD 侧保持同一 logical identity、turn、route 和节点。该机制避免单次截断让整个 episode 崩溃，但失败 generation 已消耗的时间仍属于实验成本，分析时必须单列。

### 4. Agent/tool schema

PPD 默认不再向模型发布带条件参数的 flat `str_replace_editor`，而是使用五个普通 JSON Schema 工具：

| tool | 无条件必需参数 | 后端操作 |
|---|---|---|
| `view_file` | path | view |
| `create_file` | path, file_text | create |
| `replace_text` | path, old_str, new_str | str_replace |
| `insert_text` | path, insert_line, new_str | insert |
| `undo_edit` | path | undo_edit |

`replace_text.new_str` 即使表示删除也必须显式传空字符串。五个 wrapper 复用原 editor 数据面，在 Modal I/O 前用同一 Pydantic model 校验 arguments；绑定同一 sandbox 时共享 undo history。旧 `str_replace_editor` 仍保留以兼容历史配置，但不再出现在 PPD 默认模型工具列表中。

`insert_line` 的 off-by-one 已修复：`0` 表示文件开头，`1` 表示第一行之后。Analyzer 会把新工具名归一为 `editor:view`、`editor:str_replace` 等稳定分类，以保持历史报表可比。

Schema 只能保证参数结构，无法提前判断 `old_str` 是否存在、搜索目标是否正确或修改方案是否有效。工具 handler/RPC 正常也不等于语义成功。

### 5. Profiling 数据契约

每轮 Agent trace 包含：

- 完整 request attempts 和原始 response body；
- assistant content、reasoning 和 tool calls；
- prompt、completion 和 cached tokens；
- LLM 起止时间与总耗时；
- 每个工具的 arguments、observation、handler 状态和耗时；
- Modal sandbox lifecycle/RPC events；
- run、episode、step、logical request 和 physical attempt 关联 ID；
- episode 退出原因及服务器返回的 `ppd_request_metrics`。

多个 tool call 在单个 episode 内顺序执行，不会由同一 episode 同时制造多个工具 CPU 峰值。Trace 为复现目的保存完整 request history，因此文件大小随多轮上下文接近二次增长；分析器应使用 schema 化 step/request 表，而不是依赖文件大小推断工作量。

### 6. 可复现性设置

`temperature=0` 能降低采样随机性，但不能消除 GPU batching/调度、浮点数值、失败重试、Modal/RPC 和动态工具 observation 带来的 trajectory 分叉。因此需要区分：

- live Agent：用于生态真实性、Agent 质量和端到端行为；
- request/tool-gap replay：用于 Static、Dynamic 和 TTL 的严格 serving 因果比较。

## 二、Agent 侧冻结边界与保留的技术债

### 1. 冻结的实验契约

后续对照统一固定：

- uni-agent `449f80f4383b` 及当前 `task_config_ppd.yaml`；
- Qwen3-30B-A3B-Thinking-2507、BF16、155648 context；
- 131072 episode completion budget、16384 per-turn max、`temperature=0`；
- reasoning 回放、五工具 schema、required→auto retry 和现有 ReAct policy；
- SWE-bench sample list、rollout identity、routing map 与 global concurrency；
- 完整 Agent/Modal/request trace 数据契约。

当前 Agent contract 已可作为 serving baseline：reasoning、token、tool arguments、Modal 边界和 request identity 的语义稳定。详细验收数据移至 [实验记录第一章](./agent-ppd-experiment-log.md)。除非发现新的 trace/协议错误，后续不再一边修改 Agent policy 一边比较 serving 策略。

工具结果必须分两层解释：`status=ok` 只代表 handler/RPC 返回；文件是否改变、命令是否达到目的和任务是否解决属于 `semantic_outcome`。这一区分保持为分析契约，不把无效编辑误报为工具成功。

### 2. 非阻塞技术债

- context-limit 预检查仍按 observation 字符数近似，请求成功后才由 vLLM 的真实 `prompt_tokens` 校正；
- trace 每轮保存完整历史，文件体积接近二次增长；
- `semantic_outcome`/`mutation_applied` 尚未成为所有工具的统一结构化字段；
- 模型仍可能重复 view、搜索错误目标或提交不命中的 `old_str`；
- required→auto 只能恢复 episode，不能追回失败 generation 的服务时间；
- Modal 抖动可以拆分统计，但无法由 Agent 代码彻底消除。

这些问题目前不阻塞 PPD/TTL serving profiling。若后续要研究 Agent policy 或质量，应另建实验轴，不能混入 serving-policy A/B。

## 三、PPD 侧已经完成的修复

### 1. Tool append 精确计数

Proxy 以最近一个 assistant message 为边界：

    system/user/assistant/reasoning/tool_calls
                    |
                    +-- reusable context

    最新 tool observation 或 user message
                    |
                    +-- current append

Append 使用与模型一致的 tokenizer 和 chat template 对最新 suffix 计数，解决此前遗漏工具返回内容的问题。这里的精确是 suffix 的 tokenizer 计数；完整 prompt 的真实增量仍需处理模板边界，历史 context 目前仍有字符估算。逻辑 append 也不等于某节点实际必须重算的 tokens。当前还记录：

- `feature_token_estimator`；
- `append_tokens_fallback_est`；
- `append_message_roles`；
- `feature_tokenizer_path/error`；
- `last_message_role`；
- `assistant_reasoning_replayed`。

相关提交：

- `669ac51c6 Tokenize live-agent append suffixes for Dynamic PPD`
- `ceb035cec Allow disabling append tokenizer explicitly`

### 2. Calibration 文件名与 loader 契约

Benchmark writer 统一使用规范小数 QPS 文件名；loader 优先读取规范名称，同时兼容旧整数名称，并校验文件内部 config、workload 和 QPS。启动时会输出 matched calibration coverage，缺失或不匹配不再静默伪装为完整 calibration。

当前 clean QPS≤4 calibration coverage 已由 loader 显式校验；具体 pair/lookup 数量、回归和真实目录验证记录见 [实验记录第一章](./agent-ppd-experiment-log.md)。QPS 6～20 仍没有 clean calibration，后续若策略进入这些档位必须先补数据或显式报告 fallback。

### 3. Matched routing 与 retry identity

当前明确分离两类 identity：

- `run_id + conversation_id + PPD_PROXY_EPOCH`：只负责当前 stack 的 turn state 与 cache affinity；
- `routing_key=instance_id|rollout=N`：只负责跨 Static/Dynamic/TTL matched run 的稳定实验分配。

Agent 从 SWE-bench task 的 `metadata.instance_id` 生成 routing key，经 ContextVar 和 `X-PPD-Routing-Key` 传给 proxy。Proxy 对排序后的 server addresses 使用显式 seed 的 SHA-256 选择 role slot，不再依赖 Python `hash()` 或 ZMQ 注册顺序。

前 16 条 SWE-bench Verified 的冻结映射位于：

`/root/vllm-ppd/configs/routing/swebench_verified_first16_balanced_2x2.json`

正式 paired run 必须设置 `PPD_ROUTING_MAP_STRICT=true`；缺 key、缺 role 或 slot 越界均 fail closed。每个 timing row 和 `/status` 记录 contract version、seed、map fingerprint、候选节点、slot 和 assignment source。`legacy_hash` 只用于历史行为诊断。

稳定 SHA-256 只保证可复现，并不保证有限样本恰好均分。上述显式 map 固定前 16 条的 8/8 episode 分配，用来控制对照实验中的 placement 差异；原始 hash 出现不均衡不等于路由实现错误，8/8 也不等于 token、请求数或瞬时 GPU 压力相等。正常多轮过程中复用已分配的 P/D affinity，Dynamic 改的是在固定 P 或固定 D 上做 prefill，不是每轮把 decode 迁往另一 D。故障、节点变化和状态重置应另行报告。

同一 Agent step 的 transport retry 和 required→auto retry 复用 `client_request_id`。Proxy 只在第一个 physical attempt 上递增 logical turn 并完成一次路由/Dynamic 决策；后续 attempt 复用 turn、地址、route、decision features、QPS snapshot 和 calibration cell。所有统计明确区分 logical request、physical attempt 和 retry。

### 4. 请求级 profiling 与协议语义

P 节点的 prefill-only 请求保留与 D 请求一致的 messages 和完整 tool schemas，但固定为 `tool_choice=none`、`max_tokens=1`、`stream=false`。Vendored vLLM 保持 `exclude_tools_when_tool_choice_none=false`，使 prompt/KV 前缀不因工具 schema 被删除，同时避免 prefill-only 结果进入 required-tool 严格 JSON 解析。

非流式 vLLM response 提供 `ppd_request_metrics` schema v1：

- first queued→first scheduled 的 queue time；
- first scheduled→first token 的 prefill elapsed；
- first token→last token 的 decode elapsed；
- inference elapsed、request TTFT 和 TTFT residual；
- request-level scheduler preemptions；
- prompt、cached、uncached、completion 和 requested-max tokens。

`prefill_elapsed` 是包含 prompt 计算/重算、stall 和 preemption 的阶段墙钟，不等同于纯 GPU kernel time。`uncached=prompt-cached` 还包含合法的新 append；Analyzer 另用相邻 step 计算 `previous_reusable_tokens`、`lost_reusable_tokens` 和 `cache_retention`，retry 的 warm cache 不进入 retention 证据。

Proxy schema v4 为每个 physical attempt 记录 P/D HTTP 边界、两侧 engine metrics、ORCA waiting/KV、requested/actual output、routing/decision metadata、lifetime decision QPS 和只用于观测的 1/5/10/60 秒 window QPS。失败 attempt 也保留同一字段骨架。

Analyzer 在 run 目录生成：

- `request_profile.jsonl`：无损完整字段；
- `request_profile.csv`：稳定分析列；
- `analysis.txt`：per-attempt phase 表和全局 p50/p90/max。

每行分别 join P 与 D/pD 的 enter/max/exit running、waiting、KV、GPU 和 preemption counter delta，同时区分真正 overlap 的其他 LLM requests 与生命周期仍活跃但可能正在运行工具的同节点 episodes。

原始 Dynamic 算法未在复现阶段修改。新增字段包括 `decision_source`、requested/selected QPS、`selected_calibration_cell` 和 calibration availability；窗口 QPS 只记录，不参与原始决策，retry 不计入 logical QPS。完整定义见 `/root/vllm-ppd/docs/agent_serving_contract.md`。

## 四、PPD 当前的问题与备选解决方案

本章保留基础适配和拓展适配的问题分类。当前从候选池中选定状态条件化查表，原评分不变，另建匹配实际 KV/负载的新 calibration；旧表只换字段不会产生新证据。未被选中的预测器、反馈门控和 TTL pin 不属于第一版任务。完整旧问题见 [早期理论方案](./stage5-agent-compatible-dynamic-theory.md)，当前取舍见 [阶段 5 当前方案](./stage5-state-aware-routing-and-ttl-design.md)。

### 4.1 基础适配：修正特征含义、校准范围与决策解释

| 问题 | 已确认现状 | 备选方案与取舍 |
|---|---|---|
| B1 output 长度 | Live proxy 将 `max_tokens`/`max_completion_tokens` 当作 output 特征；ShareGPT benchmark 则读取记录中回答的 token 数并控制生成长度，并无可部署的预测器 | 第一版保留生成上限，但查表改用 N/U_P/U_D，不预测本轮 output；last/median/EWMA 和组件成本路由仅保留为历史备选 |
| B2 QPS 与 concurrency | 原决策用此前 logical requests/proxy uptime；已有 1/5/10/60 秒窗口只用于 profiling。Offline QPS 是 conversation arrival，live 是 LLM request arrival | 先明确到达对象、窗口和 retry 分母，再做同单位 calibration 或新模型。活跃 episode、in-flight requests、engine running/waiting 分开记录，不能拿 concurrency 直接替代 QPS |
| B3 short bypass | append<512 提前选 D-local；短 observation 不保证 D 历史还在，也不保证本地 prefill 无干扰 | 保留原规则作 baseline；关闭/降阈值是敏感性方案；按实际未命中量与延迟成本决定 bypass 更合理，但依赖拓展适配。旧表全正时仅关闭 bypass 仍不改变动作 |
| B4 context 与 workload 覆盖 | 原始历史锚点仅 256/2048 tokens，4096 是分类边界；长 Agent context 外推 large。Suffix 已计数，完整 prompt/context 与模板边界仍需对齐 | 选择真实长 context 和两端前缀状态配对校准。N/U_i 已隐含 C_i，H/A 另作日志；连续耗时拟合不是第一版前提，不能只改阈值不补数据 |
| B5 lookup 动作与优化目标 | 当前 72 个实测 score>0，108 个缺失 cell 默认 PPD；输入修正可改 cell，却改不了动作 | 先审计 fallback 与边界，再验证 PD 获益区间。保持等权公式，不用调权重、默认值或 P:D 比例人为制造 PD；新目标须另命名、单独对照 |
| B6 placement 与可复现性 | 稳定 hash 不保证均分，显式 8/8 map 也不保证工作量均衡；live 轨迹仍会分叉 | 保持 map/affinity 控制 placement，逐节点报告真实负载；严格 serving 对照另补 replay，见 4.3 |

等权时原始策略使用：

```text
S = (TTFT_PD − TTFT_local) / TTFT_PD
  − (TPOT_local − TPOT_PD) / TPOT_PD
有效 calibration 且未被 bypass：S > 0 选 local；S ≤ 0 选 PD。
```

因此，PD 出现的条件是 local 的相对 TPOT 代价足以抵消其相对 TTFT 收益，或 PD 连 TTFT 也更好。更高 QPS 不保证满足这一条件：P 排队或传输越拥堵，反而越可能偏向 local；D 正在密集 decode、追加 prefill 干扰较大而 P 有余量时，PD 更可能获益。增加 P:D 也不是必然解，且改变资源容量与校准适用域，本阶段保持 2:2。

该 score 不是预计节省的总秒数，也不直接保证吞吐或 SLO。非流式 Agent 要等完整 response 才能执行工具，后续必须同时报告 full-response/tool-ready 时间、TTFT/TPOT、队列、失败和 goodput，不能只看 P 使用率。

### 4.2 拓展适配：引入决策时可用的执行状态

| 问题 | 为什么原特征不够 | 备选方案与边界 |
|---|---|---|
| E1 实际 KV residency | 历史长度 H 不等于 P/D 可复用前缀 C_P/C_D；response cached_tokens 是事后结果 | 节点提供无副作用的 prefix/block 查询，或维护带时间戳的 cache 目录；估算只作低置信度 fallback。查询后还会发生 eviction，需执行时复核或有界 reservation，不能把快照当作命中承诺 |
| E2 队列、decode 干扰与滑动窗口 | 相同 arrival QPS 可对应不同排队工作量、长上下文 decode 和容量压力；事后 load max 不能作在线特征 | 第一版使用 B_D/K_D 与两端 Q_tok/Q_att 类别，分别记录实际 scheduled/blocked/pending。TTFT/ITL/QPS 窗口先作 profiling，不启用 AMPD 前置路由 |
| E3 单向数据面与正确路径成本 | 当前 producer→consumer 路径没有 D→P 历史回传。P 可能复用自己保留的历史，也可能因多轮 local 而远落后于 D | 在现有 connector 上建模 P/D 各自未命中量、排队、计算和实际传输；不假定回 P 只计算 observation。双向传输列为独立系统方案，需要 pin/merge/布局与并发保障 |
| E4 目标、预测误差与 fallback | 平均 TTFT/TPOT 不能反映所有 ITL 尖峰或对其他请求的干扰；过时指标会选错路 | 第一版固定原 score/权重，表外、近零 score 与过时状态须明确回退。容量检查不等于保证准入，Remote 仍需 D 保存 KV；最终 Agent 目标不能直接等同 TTFT 最小化 |
| E5 cache loss 与 TTL 收益边界 | prefix 未命中不一定是容量驱逐，保护更多 KV 也可能挤压其他请求 | 先补 token-LCP、实际 block eviction、后续重算与活跃 preemption 证据；区分自然命中与额外保护收益，不从 loss 数量直接推断 TTL 效果 |
| E6 TTL 移植与路由耦合 | 公开固定阈值 pin 不等于论文完整估计器；保护价值取决于回访路径 | 当前只定义状态和生命周期接口，不 pin/unpin；固定 TTL、论文式效用、优先级及 route-aware TTL 均后置独立验证 |

AMPD 在线路由可以不使用当前最终 output 长度，但它并非完全免预测：仍需 prefill/传输成本及新鲜节点状态。原 Algorithm 1 先查 P 的 TTFT 余量，再查 D 的 ITL 余量，最后比较成本；不能将其概括为“每次都计算成本取最小”。我们固定候选 P/D、保留单向 connector 的方案只能称为 AMPD-inspired。

TTL 决策发生在 response 完成、等待工具期间，路由发生在下一请求到达时。保护 D cache 可能减少下一轮 local 重算；长工具 gap、过多 pin 或 decode 瓶颈也可能使 TTL 无收益甚至变慢。Continuum 公开代码缺少论文完整估计器，固定阈值 pin 与论文效用策略必须分别命名。TTL 到期是解除保护，不等于立即 eviction，更不是 offload。

### 4.3 对照实验仍需控制的因素

稳定 map 已解决跨 run 的 placement 漂移；是否强制样本数均分属于实验设计选择，不是 PPD 正确性的必要条件。报告继续按节点列 episodes、requests、tokens、KV 与 queue，避免把 8/8 当作负载均衡。

Live Agent 继续用于 profiling 和生态验证。固定 temperature/sample/map 仍不能固定生成轨迹；严格 serving 对照需要明确 completion 工作量、retry 和依赖关系的 request/tool-gap replay。同 episode 的 closed-loop gap 回放与固定绝对 arrival 的 open-loop 压力测试不能混称。

当前 knee/stress 工作点可用于机制验证，但换模型、样本、工具行为、上下文或 routing contract 后需要重新检查。所有预测只使用决策时可见的信息；本轮实际 completion、下一轮 cached tokens、最终 episode 步数和事后 queue/load max 只作分析标签。

## 五、后续工作与阶段状态

### 阶段 1：冻结 Agent contract（已完成）

Token/reasoning/tool/trace/Modal contract 已完成并冻结在 uni-agent `449f80f4383b` 与当前 YAML。后续 serving A/B 不改变 ReAct policy；新协议缺陷单独修复和验收。

### 阶段 2：冻结 matched routing 与 retry contract（已完成）

稳定 routing key、strict frozen map、logical/physical retry identity 和 fingerprint 已完成 CPU/live 验收。保留默认稳定 hash 与显式 balanced map 的区别；正常路由维持 D affinity，不把跨 D 迁移混入 Dynamic。

### 阶段 3：确定 Static 工作点并复现原始 Dynamic（已完成）

已完成当前 workload 的 Static c=12/c=14/c=16 与原始 Dynamic c=14。原始 Dynamic 本次动作等价于 x=1；无需在相同旧表上为纯输入替换重复完整 baseline。详细结果与限制见 [实验记录](./agent-ppd-experiment-log.md)。

### 阶段 4：冻结请求级 profiling/协议语义（已完成现有契约）

P/D phases、TTFT/preemption/tokens、ORCA/load、retry、decision source/cell、append 和窗口 arrival 已进入 artifact。此处“完成”不包括决策前 residency API、实时逐迭代 ITL、纯 NCCL bytes/time、block eviction 生命周期和 replay；这些是阶段 5/6 明确新增的能力。

### 阶段 5：KV/负载状态条件化的 Dynamic 查表（语义与观测 CPU 验证完成，原生验收/策略待实施）

维持 2P:2D、固定 P/D affinity、模型、Agent contract、前 16 条样本与 map。当前完成旧表可达性审计和问题/备选方案整理，未实现 output predictor、窗口驱动决策或 state-aware router。

第一版输入为 (N,U_P,U_D)、D 的 (B_D,K_D)，以及 P/D 各自的 (Q_tok,Q_att)。H/A 是逻辑历史与新增，C_i 才是节点有效前缀；U_i=N−C_i。对可执行/执行中的背景 prefill 及去重 pending，用 R_r=N_r−E_r 汇总 Q_tok=ΣR_r 与 Q_att=Σ[E_rR_r+R_r(R_r+1)/2]；blocked 与待决策本请求不混入该集合。这些是工作量而非毫秒，E_r 不能直接取尚未完成的 scheduled 进度。

2026-10-22 已完成第一步“冻结输入与结果语义”：新增 [规范契约](/root/vllm-ppd/docs/stage5_state_semantics_contract.md)、[CPU 参考函数](/root/vllm-ppd/ppd/optimizer/state_semantics.py) 和 [单元测试](/root/vllm-ppd/tests/test_state_semantics.py)。冻结 H/A 前缀证明、有效 C<N、decode 同步边界与 L=N+G−1、E≤S≤T、物理 pending 去重，以及 candidate Turn 2/首 attempt/非 warmup 的请求加权统计。缺失值用 null，不当成零。

CPU 验收为新语义 46 项、原 calibration 契约 4 项 unittest 通过；扩展 pytest 回归未执行（当前环境缺 pytest），没有安装依赖。Agent/服务配置、first16 map 和原决策文件指纹不变。该验收不包含真实引擎状态导出或新路由性能。

测量来源特别分离：旧 score 使用 client-stream 首文本 TTFT/TPOT；D engine TTFT 不含 P 阶段；poller 的旧 `tpot` 别名是 ITL 事件直方图，不是 request-mean TPOT。当前非流式精确 route-wide 首生成 token 仍未观测，不能用于新路径胜负标签。参考函数未被 proxy/engine 导入，不说明状态采集已实现。

后续按以下依赖推进，分桶边界、回退细则及实验点尚待确认：

1. **5A 状态观测（实现与 CPU 验证已完成，GPU 验收未完成）。** 默认关闭的 `/v1/ppd/observe` 经 owner-thread utility 导出状态；独立 block 就绪记录处理登记/完成/回收，缺证据返回未知。Pending 账本已独立测试，尚未挂入 proxy 派发链路，需核实原生 request ID 和终止 ACK 映射。开销已做 CPU 假负载测量；真实长前缀、NCCL、IPC 与 off/on serving 对照需要开卡。见 [实现说明](/root/vllm-ppd/docs/stage5_readonly_observer.md) 和 [实验记录第九章](./agent-ppd-experiment-log.md)。
2. **5B 配对状态校准。** 选择真实长 context、不同 P/D 复用和 decode/积压背景，恢复可比起点后测两路径。沿用 TTFT/TPOT 口径并保留失败；旧均值不能重新命名为新状态标签。
3. **5C 新表与 shadow。** 保留原 score 和 w_ttft=w_tpot=1，输出 Local/Remote，同时保存实测指标、波动与支持范围。验证类别、fallback、协议可达性，不再在三个策略家族中择一。
4. **5D 有限 live 与消融。** 只启用状态查表；单独验收 bypass 与 table 的关系，记录完整响应、episode、队列/ITL、失败和质量。当前不加 AMPD 门控或 TTL。
5. **5E 严格 serving 比较。** 明确 replay 的生成工作量和工具间隔语义后，比较端到端延迟、goodput、重算与不确定性；保留 Static x=1、原始 Dynamic 和必要的强制 PD 对照。

双向传输、P:D sizing、D migration、admission、offload、队列重排不在第一版中一起修改。原始 Dynamic 与静态基线保留，新方案单独命名；若新场景仍全部 Local，允许作为结论，不人为提高 PD 占比。详细定义见 [阶段 5 当前方案](./stage5-state-aware-routing-and-ttl-design.md)。

### 阶段 6：TTL 及与 Dynamic 的组合（后置，当前仅留接口）

本轮仅约定有效前缀、保护容量、epoch 和 request/tool/episode 生命周期接口，不实施 pin/unpin，不改变缓存策略。以下保留为后续候选计划，不与当前状态查表同时启用；将来可先在 Static x=1 独立验证，再在路由与 TTL 各自验收后研究组合。

1. 补齐 episode/tool-gap 生命周期与可验证的 cache loss 事件；返回 gap 包含 Modal/RPC/客户端开销，不能只用 shell 执行时间。
2. 接入有界 pin/unpin、共享 block 引用、传输保护和取消/结束清理；expiry 解除保护，不强制删除仍可自然命中的 cache。
3. 分开比较不加 TTL、固定 TTL、Continuum-style utility，并隔离回访优先级的贡献。公开仓库的固定阈值实现不冒称论文完整版。
4. 用当前 healthy/knee/stress 工作点检查收益和代价：减少真实重算是否足以抵消 pin 引起的排队、preemption、ITL 或其他 episode 的损失。
5. 再研究 route-aware retention：缓存保护应取决于它能改变哪条路径的成本，而不是把 gap 预测数值同时塞进 Dynamic 与 TTL。

最终组合需要回答“在同等工作量下是否更快、更少超时且不伤害其他请求”，不能只报告 retention 上升或 eviction 下降。当前证据不足以宣称 PPD+TTL 优于 co-located Continuum，也不足以宣称此前论文已经否定这种组合。
