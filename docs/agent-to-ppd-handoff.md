# Agent 修复阶段总结与 PPD/TTL 改动计划

更新时间：2026-09-09

## 结论

Agent 与 PPD 的 baseline contract 已经冻结：Agent 侧的 token 预算、thinking/reasoning 回放、结构化工具、截断重试和 trace 语义已稳定；PPD 侧的精确 append、calibration loader、matched routing、logical retry 以及请求级 profiling 也已完成。后续 serving 对照实验应固定现有模型、Agent/PPD commits、样本集合、routing map、temperature 和 token 参数，不再同时修改 Agent 行为。

原始 PPD Dynamic 的 offline calibration 和 live Agent 执行链路都已复现。最新视角不是“Dynamic 已降低 cache loss”，而是：原始 Dynamic 在当前 live Agent workload 上退化为 Static `x=1`，根因集中在 requested-max output、lifetime-average QPS、short-append bypass 和 huge-context 外推这四类特征/策略契约。Static 与 Dynamic 的 live trajectory 又会明显分叉，因此两轮只能作为 config-matched 生态验证，不能把 wall time、KV 压力或 cache-loss 差异直接归因于 serving policy。

当前 Static 基线已经确定三个工作点：c=12 是 healthy control，c=14 是 operational knee，c=16 是 stress control。现有 cached-token retention 能发现“可复用前缀没有命中”，但还不能区分容量驱逐、prefix divergence、block 对齐、offload 或 restore；在评价 TTL 前必须补 token-LCP 或 block-level eviction/offload/restore/recompute 证据。

下一阶段是 Agent-compatible Dynamic 的单变量 ablation，依次处理 output 预测、压力/QPS 信号、short-append bypass 和 Agent context calibration；之后再集成 Continuum TTL。严格 serving 因果主表最终应使用录制 request body 与 tool gap 的 replay，live Agent 保留为端到端真实性和质量验证。

详细的 CPU/live 验收、正式 run 数据、分位数、cache-loss 表和可复现性限制统一记录在 [Agent × PPD 实验记录](./agent-ppd-experiment-log.md)；请求字段定义见 `/root/vllm-ppd/docs/agent_serving_contract.md`。

当前代码状态：

- uni-agent 分支：`fix/agent-tool-contract`
- uni-agent 最新提交：`449f80f4383b Add stable routing identity to agent profiles`
- vllm-ppd 分支：`fix/agent-tool-contract`
- vllm-ppd contract 提交：`ddffde8fcc9b Freeze matched routing and request profiling contracts`
- vllm-ppd 最新提交：`11dccb5d6 Fix small-sample percentiles and TTFT detection`
- uni-agent 比 GitHub origin 超前 1 个本地提交，vllm-ppd 超前 2 个，均尚未 push
- 两个代码仓库的 `agentic-ppd-baseline-v1` tag 均保持不变

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

PPD 配置启用 `replay_reasoning=true`。下一轮请求会带回上一轮 reasoning，从而同时保留 Agent 计划、还原真实生成前缀，并使 prefix-cache、cache-loss 和 TTL retention 具有一致语义。

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

Append 使用与模型一致的 tokenizer 和 chat template 计数，解决 Agent trace 与 proxy 对工具增量长度统计不一致的问题。当前还记录：

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

## 四、PPD 当前的核心问题

### P0：matched routing 已解决，严格 serving 因果仍缺 request replay

稳定 routing map 解决了跨配置 episode 身份与节点分配不可复现的问题，但 8/8 episode assignment 不保证两节点的 request 数、token 数、瞬时 resident KV 或 wall time 相等。即使固定 `temperature=0`，live Agent 仍会因 batching、数值差异、失败重试和工具 observation 走出不同 step 数与上下文。

因此后续报告必须继续按节点列 episodes、logical/physical requests、prompt/completion tokens、KV、waiting、queue 和 preemption。Live Static/Dynamic/TTL 只做 config-matched 生态验证；严格 speedup、cache-retention 和 recompute 因果比较需要录制相同 request body、arrival/tool gap 后 replay。

### P1：原始 Dynamic 特征不适配 live Agent

详细观测分布见 [实验记录第七章](./agent-ppd-experiment-log.md)。当前需要拆成四个单变量问题。

#### 1. output_tokens 使用请求上限

Proxy 使用 `max_tokens`/`max_completion_tokens` 作为预期输出，而 Agent 的 per-turn 上限远大于多数真实 completion，使 workload class 系统性偏向长生成。候选改法是用近期真实 completion 的 rolling median/p90 或简单预测器，同时保留 requested max 作为对照字段。

#### 2. QPS 使用 lifetime average

原始策略使用 stack 启动以来的累计 logical requests 除以累计运行时间。工具执行和 Modal 等待会持续稀释该值，它既不是瞬时 arrival rate，也不是 scheduler pressure。

当前 profiling 已同时记录 lifetime QPS、1/5/10/60 秒 window arrival、active episodes、waiting、running、queue 和 KV。下一阶段分别测试 sliding-window arrival 与 queue/KV pressure；在单变量实验前不改变原始定义。

#### 3. short-append bypass

原始策略在 append `<512` tokens 时直接选择 PPD，大量短工具 observation 因而完全绕过 calibration。后续依次比较原阈值、128/256 和关闭 bypass，并用实际 append 分布解释结果。

#### 4. Huge context 外推与 offline mismatch

原始 calibration 的最大 context class 远小于长程 live Agent context；huge 请求只能外推到 large cell。与此同时，offline benchmark 是固定两轮、固定输入输出且没有工具间隔，真实 Agent 则包含 reasoning、动态 observation 和不规则 think time。

Offline calibration 足以验证原始 Dynamic 流程，却不能证明其 cell/score 适合 live Agent。后续先保持原表作为 reproduction baseline，再增加 Agent-derived context/output calibration；两者必须分别报告。

### P2：缺少 TTL-aware cache policy 与可验证事件

当前系统没有显式 KV offload 层，也没有按 Agent 预计返回时间管理 cache block。`cached_tokens` retention 只能说明下轮没有命中全部可复用前缀，不能单独证明容量 eviction，更不能判断数据是重算、offload 还是 restore。

集成 Continuum 前需要：

- 给 conversation/cache block 增加 TTL 或预计复用时间；
- 由 Agent/proxy 传递和记录 tool gap/next-arrival 预测；
- scheduler 在内存压力下优先驱逐已过期或低复用价值 KV；
- 保留 PPD connector 与 NCCL KV transfer；
- 增加 token-LCP 或 block-level eviction、offload、restore、recompute 事件；
- 将 prefix divergence、alignment loss 与容量 eviction 分开统计。

Static c=14 knee 用于观察策略是否能在刚出现容量压力时减少 loss；c=16 stress 用于验证 TTL 在高压下的稳定性与代价。没有上述事件证据前，不把 retention 改善直接写成 TTL 减少 eviction。

## 五、后续工作与阶段状态

### 阶段 1：冻结 Agent contract（已完成）

Token/reasoning/tool/trace/Modal contract 已完成并冻结在 uni-agent `449f80f4383b` 与当前 YAML。后续 serving A/B 不再改变 ReAct policy。

### 阶段 2：冻结 matched routing 与 retry contract（已完成）

稳定 routing key、strict frozen map、logical/physical retry identity 和 fingerprint 已实现并完成 CPU/live 验收。历史 `legacy_hash` 结果只保留作诊断样本。

### 阶段 3：确定 Static 工作点并复现原始 Dynamic（已完成）

Static c=12 healthy、c=14 operational knee、c=16 stress 以及同配置原始 Dynamic c=14 均已完成。原始 Dynamic 在 live Agent 上等价退化为 `x=1`，不再重复该 baseline。完整数据与限制见 [实验记录](./agent-ppd-experiment-log.md)。

### 阶段 4：冻结请求级 profiling/协议语义（已完成）

P/D request phases、TTFT/preemption/tokens、ORCA/load windows、logical retry、decision source/cell、exact append、lifetime/window QPS 和 retention 字段均已进入 schema 化 artifact。当前证据是 request-level elapsed 与 lost reusable prefix，不冒充纯 kernel recompute 或 block eviction。

### 阶段 5：Agent-compatible Dynamic ablation（下一步）

保持模型、Agent commit、前 16 条 sample、frozen map 和选定 concurrency 不变，一次只修改一个变量：

1. 用历史真实 completion 预测 output tokens；
2. lifetime QPS 改为 sliding-window arrival 或 queue/KV pressure；
3. 调整或关闭 512-token bypass；
4. 增加 Agent-derived context calibration；
5. 最后才组合已经单独验证有效的特征。

每轮同时报告策略选择比例、实际 workload、每节点压力、失败重试、quality 和 trajectory 差异。若仍使用 live Agent，只能称为生态 A/B；严格性能主表使用 request/tool-gap replay。

### 阶段 6：集成 TTL

在 Static PPD 上先单独验证 Continuum TTL，再与 Agent-compatible Dynamic 组合：

| 配置 | 用途 |
|---|---|
| 2P_2D，`x=0` | PD 基线 |
| 2P_2pD，`x=1` | Static PPD 基线 |
| 原始 Dynamic | 论文复现 baseline |
| Agent-compatible Dynamic | Agent 特征适配 |
| Static PPD + TTL | 隔离 TTL 收益与代价 |
| Dynamic + TTL | 最终组合方案 |

建议工作流：

    Agent contract frozen
        -> deterministic matched routing
        -> Static c=12 healthy; c=14 knee; c=16 stress
        -> original Dynamic reproduction complete
        -> Agent-compatible Dynamic single-variable ablations
        -> request/tool-gap replay for causal comparison
        -> Static PPD + TTL
        -> Dynamic + TTL
