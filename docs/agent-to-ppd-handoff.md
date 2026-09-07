# Agent 修复阶段总结与 PPD/TTL 改动计划

更新时间：2026-09-07

## 结论

阶段 2（实验路由契约）和阶段 4（请求级 profiling/协议语义）已通过 c=2 与完整 c=16 的真实 GPU 验收。最新 Static `x=1` run 使用 frozen map 将 16 个 episode 严格分为 8/8；370 个逻辑请求、372 个物理 attempt 均保持相同 map fingerprint，16 次 Turn 1 为 `ppd`，354 次 Turn 2+ 为 `ppd_direct`。两次 required-tool 失败均以同一 client request、turn、route 和节点重试成功，没有再污染 turn counter。

c=16 应正式定义为 stress，而不是 knee：353 个可判断 transition 中 123 个 retention 低于 90%（34.8%），其中 90 个低于 50%，共有 58 次 request-level preemption。尽管 episode 数是 8/8，8200/8201 最终承载 232/138 个逻辑请求和 6.72M/3.56M cumulative prompt tokens；122/123 个 loss 与全部 58 次 preemption 都落在 8200。“身份均衡”已复现，但 live Agent 的轨迹长度仍造成明显的“工作量不均衡”。

请求阶段数据把瓶颈进一步定位到 queue 与 prefix recompute/prefill。cache-loss 请求的 queue/prefill/TTFT/LLM p50 为 6.04/3.62/12.25/36.76 s，retained 请求为约 0/0.134/0.176/15.31 s；前序工具时间反而是 1.39/1.44 s。当前 loss 与 KV 满载、waiting、排队和 preemption 共现，不是长工具调用导致。下一步应保持 16 个 sample 和同一 mapping，只把并发降到 12，寻找 knee 后再跑 matched Dynamic。

当前代码状态：

- uni-agent 分支：fix/agent-tool-contract
- uni-agent 最新提交：449f80f4383b Add stable routing identity to agent profiles
- vllm-ppd 分支：fix/agent-tool-contract
- vllm-ppd contract 提交：ddffde8fcc9b Freeze matched routing and request profiling contracts
- vllm-ppd 最新提交：11dccb5d6 Fix small-sample percentiles and TTFT detection
- uni-agent 比 GitHub origin 超前 1 个本地提交，vllm-ppd 超前 2 个，均尚未 push
- 两个代码仓库的 agentic-ppd-baseline-v1 tag 均保持不变

阶段 3 已完成 c=2 gate 和 deterministic 8/8 Static c=16 stress；尚未执行的是 16 episodes、concurrency=12 的 Static knee 定标及随后 matched Dynamic。

## 本次完成：阶段 2 与阶段 4

### 阶段 2：实验路由契约已冻结

现在明确分离了两类 identity：

- `run_id + conversation_id + PPD_PROXY_EPOCH` 只负责当前 stack 中的 turn state 和 cache affinity；
- `routing_key=instance_id|rollout=N` 只负责跨 Static/Dynamic matched run 的实验分配。

Agent 从 SWE-bench resolved task 的 `metadata.instance_id` 生成 routing key，经 ContextVar 和 `X-PPD-Routing-Key` 传给 proxy。Proxy 默认按数值端口排序节点，并使用带显式 seed 的 SHA-256 对 P、D 等 role 独立选 slot，不再依赖 Python 进程随机 `hash()` 或 ZMQ 注册顺序。`legacy_hash` 仍可显式复现旧的 Python hash + registry insertion order，但不得用于 matched 结果。

前 16 条 SWE-bench Verified 的冻结映射位于：

`/root/vllm-ppd/configs/routing/swebench_verified_first16_balanced_2x2.json`

它已从 `/root/data/swe_agent/swe_bench_verified.parquet` 重新生成并逐字比对，source SHA-256 一致，P 与 D/pD 都是严格 8/8。正式 paired run 必须设置 `PPD_ROUTING_MAP_STRICT=true`；缺 key、缺 role 或 slot 越界会 fail closed。每个 proxy timing row 和 `/status` 都记录 contract version、seed、map fingerprint、候选列表、slot 与 assignment source。

重试协议也被冻结：同一 Agent step 的 transport retry 和 required→auto retry 复用同一个 `client_request_id`，proxy 只在首个物理请求递增 logical turn 并做一次路由/Dynamic 决策；后续物理尝试复用 turn、地址、route、decision features、QPS snapshot 与 calibration cell。统计明确拆成 logical/physical/retry requests。

### 阶段 4：请求级 profiling/协议语义已补齐

P 节点的 prefill-only 请求现在保留与 D 请求相同的 messages 和完整 tool schemas，但固定为 `tool_choice=none`、`max_tokens=1`、`stream=false`。当前 vendored vLLM 保持 `exclude_tools_when_tool_choice_none=false`，因此 prompt/KV 前缀不因工具 schema 被删而改变，同时避免 1-token prefill 结果进入 required-tool 严格 JSON 解析。

非流式 vLLM response 新增 `ppd_request_metrics` schema v1：

- first queued→first scheduled 的 queue time；
- first scheduled→first token 的 prefill elapsed；
- first token→last token 的 decode elapsed；
- inference elapsed、request TTFT 和 TTFT residual；
- request-level scheduler preemptions；
- prompt/cached/uncached/completion/requested-max tokens。

`prefill_elapsed` 是包含 prompt 计算/重算、stall 和 preemption 的阶段墙钟，不冒充纯 GPU kernel time。`uncached=prompt-cached` 还包含合法的新 append，因此 analyzer 另用相邻 Agent step 计算 `previous_reusable_tokens`、`lost_reusable_tokens` 和 `cache_retention`；retry 的 warm cache 不参与 retention 证据。

Proxy schema v4 对每个物理请求记录 P、D 两次 HTTP 边界、两侧 engine metrics、ORCA response waiting/KV、requested/actual output、routing/decision metadata、精确定义的 lifetime decision QPS 和仅观测用 1/5/10/60 秒 window QPS。失败请求也保留同一完整字段集。

Analyzer 会在 run 目录生成：

- `request_profile.jsonl`：无损完整字段；
- `request_profile.csv`：稳定分析列；
- `analysis.txt`：per-physical-request phase 表和 run-wide p50/p90/max。

每行分别 join P 与 D/pD 的 enter/max/exit running、waiting、KV、GPU 和 preemption counter delta；同时区分 proxy 当时真正 overlap 的其他 LLM 请求，与整个 Agent 生命周期仍活跃、但可能正在跑工具的同节点 episodes。

Dynamic 算法结果未改。新增 `decision_source`、requested/selected QPS、`selected_calibration_cell` 和 calibration availability。策略输入仍严格保持原实现的 `prior logical requests / max(proxy uptime, 1s)`；窗口 QPS 只记录、不参与决策，retry 不计入 logical QPS。

完整字段和边界定义见：

`/root/vllm-ppd/docs/agent_serving_contract.md`

### 实现时的 CPU 验收

实现阶段没有启动或重启 GPU stack，完成了：

- 全部改动和测试文件 `py_compile`；
- 4 个 shell 脚本 `bash -n`；
- 两仓库 `git diff --check`；
- Quart 模拟 P+D 请求与 required→auto retry，验证同 turn/同节点、P 请求 contract、logical/physical 统计和阶段字段；
- vLLM response model/phase calculation 真实导入 smoke；
- Agent routing ContextVar/header/response metric 回传 smoke；
- frozen map 从原 parquet 重建并逐字一致（16 条，8/8）；
- 用旧 Dynamic c=2 run 的临时副本验证 analyzer 向后兼容，18 个物理请求成功生成 18 行 JSONL 和 18 行数据 CSV。

基础 Python 环境没有安装 pytest，因此未运行 pytest runner；等价核心断言已由独立 CPU harness 执行。随后完成的真实 GPU 验收如下。

### 2026-09-07 Stage 3 Static c=2 live contract smoke

正式产物位于：

`/root/vllm-ppd/logs/runs/stage3_contract_static_c2_20260907T0255Z`

实验参数为 Qwen3-30B-A3B-Thinking-2507、BF16、2P_2pD、Static `x=1`、前 2 条 SWE-bench Verified、2 workers、concurrency=2、temperature=0、155648 context、131072 episode completion budget 和 16384 per-turn max output。workload exit code 为 0，2/2 episode 都以 `finished` 结束，无 timeout/error；Agent driver wall 为 457.1 s，包含 wrapper 收尾的 manifest wall 为 465.6 s。两个样本均为 WA，因此该 run 只验收 serving/trace contract，不作为能力结果。

路由和请求完整性：

| 项目 | 结果 |
|---|---:|
| logical / physical requests | 30 / 30 |
| retry | 0 |
| HTTP 200 | 30/30 |
| Turn 1 | 2 次 `ppd` |
| Turn 2+ | 28 次 `ppd_direct` |
| frozen-map 首轮分配 | 8100→8200 与 8101→8201，各 1 episode |
| 后续 conversation affinity | 28/28 命中原 pD |
| map fingerprint | `0d2f3a9d50231b114bd1f963ccf0f1264b673c224804af1cbac154d8eaef459c` |

两条 Turn 1 都使用 `same_prompt_tool_choice_none_max_tokens_1_v1`，P 和 D 均返回 200。首轮 D 侧 cached 分别为 2210/2211 和 2597/2598；这是 P→pD 外部 KV transfer 的结果，不是预热命中。Turn 2+ 则全部在固定 pD 上做本地 prefix-cache 命中，没有误路由到另一个节点。

请求级数据契约覆盖率：

| 字段族 | 覆盖 |
|---|---:|
| schema v4 + exact episode/request join | 30/30 |
| D engine phases/tokens/preemption | 30/30 |
| D ORCA response load + metrics window | 30/30 |
| P engine/ORCA/metrics window（Turn 1） | 2/2 |
| Agent step `ppd_request_metrics` | 30/30 |

这说明阶段 4 的核心字段不再只是 CPU mock：非流式请求也能获得真实 engine TTFT。D 侧 queue/prefill/decode/TTFT/HTTP 的 p50 分别为 0.000016/0.116/8.83/0.147/9.07 s，p90 分别为 0.0023/0.153/29.48/0.232/29.68 s。没有 request-level preemption，request window 的 decode waiting max 为 1，KV usage max 为 27.0%。

28 个可判断的相邻 transition 中，retention min/p50/p90/max 为 97.09%/99.53%/99.74%/99.80%，没有低于 90% 的 cache-loss event。`lost_reusable_tokens` 的 33～751 token 小缺口在这种低 KV/低 queue 条件下不能直接叫 eviction；它还包含 block 对齐和从 Agent transcript 重建 reusable prefix 的估计误差。这个 c=2 结果仅证明 contract 正常，不能用于评估 TTL 收益。

Thinking 和 output 语义也得到验证：30/30 response 都有非空 reasoning 和 tool calls，assistant `content` 为空是该 thinking 模型的正常表示；28/28 个后续请求标记 reasoning replay，且 last role 都是 `tool`。真实 completion tokens 的 p50/p90/max 为 888/3150/7950，明显小于 30/30 请求都使用的 16384 上限，再次证明原始 Dynamic 把 `max_tokens` 当 output feature 会系统性高估生成长度。

工具层共 31 次调用。RPC/handler 层全部返回 `status=ok`，工具时长最高 6.33 s；但 observation 语义上有 24 次 `replace_text` 返回 `No replacement was performed`，另有 1 次错误路径。第二个 episode 因此在同一编辑动作上循环并最终提交错误答案。这里必须区分：

- `status=ok` 当前只表示工具 handler/RPC 正常返回；
- 它不表示文件修改成功或任务解法正确；
- schema split 已消除缺必需参数的 `format_error`，但无法阻止模型生成不存在的 `old_str` 或反复执行相同失败动作。

这不是 PPD serving 故障，也不妨碍本次 contract gate；不过 analyzer 以后应单列 `semantic_outcome`/`mutation_applied`，避免把 handler success 写成“工具调用成功”。若要保持当前 serving baseline 不变，可先在派生分析中标注，不改变 ReAct loop 行为。

Modal 分解还捕获到两个 episode 的 toolbox setup 分别为 41.54 s 和 11.96 s。较慢者主要是一次 `exec:tmux` 25.14 s，其中 submit 约 23.99 s；这属于 sandbox/RPC 初始化长尾，已与 episode 内工具执行、LLM 时间分开，不应归因给 grep、editor 或 LLM server。

Analyzer 在验收时修正了两个展示问题并重新生成本 run 的 `analysis.txt`/`request_profile.*`：小样本 p90 改为线性插值；TTFT warning 改为识别新的 `decode_engine_ttft_s`。对应提交为 vllm-ppd `11dccb5d6`。

## 一、Agent 侧已经完成的修改

### 1. Token 和上下文语义

Agent 现在明确区分：

- max_total_tokens=131072：整个 episode 累计生成的 completion tokens，不重复累计每轮 prompt。
- max_tokens_per_turn=16384：单次模型请求的生成上限。
- max_context_len=155648：模型实际上下文窗口。
- 接近上下文或 generation budget 上限时干净退出，不再发送必然导致 vLLM context overflow 的请求。

当前 PPD Agent 配置使用 Qwen3-30B-A3B-Thinking-2507、BF16、temperature=0。

### 2. Thinking/reasoning 的记录与回放

Thinking 模型的 assistant content 可能为空，因为思考过程位于 reasoning 或 reasoning_content 字段。

当前 Agent 分别记录：

- content
- reasoning
- tool_calls
- prompt_tokens
- completion_tokens
- cached_tokens
- finish_reason

PPD 配置启用了 replay_reasoning=true。下一轮请求会带回上一轮 reasoning，从而：

- 保留 Agent 的计划和推理上下文；
- 使下一轮输入前缀与上一轮真实生成序列一致；
- 让 prefix cache、cache loss 和 TTL retention 分析具有正确语义。

### 3. 截断 tool call 的容错

已经处理两层问题：

1. vLLM 在 tool_choice=required 下严格解析被 max_tokens 截断的 JSON 并返回 HTTP 400时，客户端使用 tool_choice=auto 重试。
2. 如果 Hermes parser 仍返回不完整 arguments，ReAct 层使用 json.loads 过滤非法 tool call。

这避免了一个被截断的工具调用导致整个 episode 崩溃。

### 4. Agent/tool schema

正式 c=36 run 暴露出旧 `str_replace_editor` 的结构性问题：模型只看到一个 flat schema，其中只有 `command` 和 `path` 能被无条件列入 `required`；`insert_line`、`new_str` 等只能靠自然语言描述和本地 validator 表达。直接发布 `oneOf` 又会触发当前 Qwen 模型漏掉顶层 `path`。因此现已采用操作拆分方案。

PPD 的模型可见工具改为：

| tool | 无条件必需参数 | 后端操作 |
|---|---|---|
| view_file | path | view |
| create_file | path, file_text | create |
| replace_text | path, old_str, new_str | str_replace |
| insert_text | path, insert_line, new_str | insert |
| undo_edit | path | undo_edit |

这里 `replace_text.new_str` 即使做删除也必须显式传递空字符串，避免“漏字段”和“有意删除”无法区分。五个工具使用独立、普通、无 `oneOf` 的 JSON Schema；Toolbox 在 Modal I/O 前使用同一个 Pydantic model 校验 arguments。

实现仍复用原 editor 数据面，并做了两项兼容处理：

- 保留 `str_replace_editor` 注册，旧配置仍可运行，但 PPD 默认配置不再把它发给模型；
- 五个 wrapper 绑定同一个 sandbox 时共享 undo history，因此 `replace_text`/`insert_text` 后可由 `undo_edit` 正确撤销。

同时修复了旧实现的 `insert_line` off-by-one：`0` 表示文件开头，`1` 表示第一行之后，与工具描述一致。ReAct 默认工具列表和 `task_config_ppd.yaml` 已切到五工具接口；analyzer 会把新名称归一回 `editor:view`、`editor:str_replace` 等稳定分类，历史和新实验仍能放在同一报表口径下。

CPU contract smoke 已验证：五个 schema 的 required 集合、缺少 `insert_text.new_str` 时不发生 sandbox I/O、view/create、replace/insert、跨 wrapper 两次 undo 和新插入行号语义。随后两次 Static c=2 live smoke 都证明模型能生成满足 schema 的新工具参数；2026-09-07 的最新 smoke 也表明 schema 合法不等于编辑语义成功，模型仍可能反复提交不存在的 `old_str`。

以下问题仍属于模型语义错误，schema 无法提前判断：

- old_str 在目标文件中不存在；
- 重复 view 同一个文件；
- grep 搜索目标错误；
- 修改方案本身不正确。

### 5. Profiling 数据契约

每轮 Agent trace 已包含：

- 完整 request attempts；
- 原始 response body；
- assistant content、reasoning 和 tool calls；
- prompt、completion 和 cached tokens；
- LLM 起止时间与总耗时；
- 每个工具的 arguments、observation、状态和耗时；
- Modal sandbox lifecycle/RPC events；
- run、episode、step 和 request 关联 ID；
- episode 退出原因。

多个 tool call 当前在单个 episode 内顺序执行，因此不会由一个 episode 同时制造多个工具 CPU 峰值。

### 6. 可复现性设置

当前使用 temperature=0，可以显著降低采样随机性，但不能完全消除：

- GPU batching 和调度差异；
- Modal 环境与 RPC 抖动；
- 文件系统或依赖状态差异；
- 模型在长上下文中的数值差异。

## 二、Agent 侧验收结果与保留的技术债

### 已完成的 live smoke 与正式验收

修正 append/reasoning 后的 Dynamic c=2 smoke 位于：

`/root/vllm-ppd/logs/runs/dynamic_smoke_corrected_flat_c2_20260906T044159Z`

它验证了 Turn 1 `pd`、Turn 2+ `ppd_dynamic`、精确 append tokenizer、reasoning replay 和三源 join；2/2 episode 自然结束，没有 timeout。

editor 拆分后的 Static c=2 live smoke 位于：

`/root/vllm-ppd/logs/runs/static_x1_editor_split_smoke_c2_20260906T073603Z`

验证结果：

- 2/2 episode 正常结束；
- 33/33 工具 handler 正常返回，无 `format_error`；
- 31/31 相邻 transition 的 reasoning 精确回放；
- 路由为 2 次 Turn 1 `ppd` 和 31 次 Turn 2+ `ppd_direct`；
- 小规模下没有 cache loss。

正式 Static c=16 run 又覆盖了 322 次工具调用，包括 193 次 replace、50 次 view、13 次 create、9 次 insert 和 4 次 undo，全部无 schema/handler error。它验证了 schema 能被模型接受以及五个 wrapper 的真实数据面，但不保证每次编辑都实际命中目标文本。

因此当前 Agent 状态应定义为：reasoning、trace、token、Modal 边界和 editor 参数 contract 均已 live 验收，可以冻结用于 serving 对照实验；工具的“handler 状态”和“语义结果”仍需分栏解释。

### 非阻塞技术债

- Agent 的 context-limit 预检查仍使用 observation 字符数近似；请求成功后才由 vLLM 返回的真实 prompt_tokens 校正。
- Trace 为复现目的保存每轮完整请求历史，因此文件大小接近二次增长。
- 语义错误工具调用仍需要模型或 Agent policy 改进；当前 `status=ok` 仅代表 handler 返回，后续应另记 `semantic_outcome`/`mutation_applied`。
- Modal 抖动只能拆分和统计，无法由 Agent 代码彻底消除。

这些技术债不阻塞 serving profiling。后续若没有发现新的 trace 契约错误，不再改变 Agent 行为；所有 Static、Dynamic 和 TTL 对照固定使用 uni-agent `449f80f4383b` 及当前 YAML。该提交只增加 stable routing identity 和服务器指标透传，不改变 ReAct policy/tool 行为。

## 三、PPD 侧已经完成的修复

### 1. Tool append 精确计数

Proxy 现在以最近一个 assistant message 为边界：

    system/user/assistant/reasoning/tool_calls
                    |
                    +-- reusable context

    最新 tool observation 或 user message
                    |
                    +-- current append

append 使用与模型一致的 tokenizer 和 chat template 计数，解决了 Agent trace 与 proxy 对 Tool append 长度统计不一致的问题。

同时记录：

- feature_token_estimator
- append_tokens_fallback_est
- append_message_roles
- feature_tokenizer_path/error
- last_message_role
- assistant_reasoning_replayed

相关提交：

- 669ac51c6 Tokenize live-agent append suffixes for Dynamic PPD
- ceb035cec Allow disabling append tokenizer explicitly

### 2. Calibration 文件名与 loader 契约

旧代码可能由 benchmark 保存：

    2P_2D_small_tiny_1.0.json

而 loader 查找：

    2P_2D_small_tiny_1.json

这使 clean calibration 中 QPS≤4 的 72 对 matched 数据只加载了 18 对，其余点静默使用 default_use_ppd=true。

当前修复包括：

- benchmark writer 统一使用小数格式；
- loader 优先读取规范文件名；
- 向后兼容旧整数文件名；
- 校验文件内部 config、workload 和 qps；
- 输出 matched calibration coverage。

真实目录验证结果：

    修复前：18 / 72 matched pairs
    修复后：72 / 72 matched pairs
    完整 lookup table：180 entries

4 个回归测试通过。

## 四、PPD 当前的核心问题

### P0：matched routing 已修复并通过 Static live 验收

旧 Static c=16 的 11/5 节点倾斜来自三项混杂：state hash 包含 run/epoch、Python `hash()` 进程随机性，以及 ZMQ 注册顺序。该历史 run 保留为 routing-skew stress baseline，但不再作为 matched Static/Dynamic 比较依据。

当前实现已经：

- 分离 stack-local state key 与跨配置稳定的 `routing_key=instance_id|rollout=N`；
- 使用显式 seed 的 SHA-256 和排序后的 server addresses；
- checked in 前 16 条任务的 8/8 frozen mapping；
- 用 strict 模式对缺 key、缺 role 和 slot 越界 fail closed；
- 在每个 timing row 与 `/status` 中记录 fingerprint、slot、assignment source 和候选节点。

c=2 live smoke 验证了 frozen-map slot 与后续 conversation affinity；完整 c=16 又验证了 16/16 首轮、354/354 后续 affinity 和 372/372 个 physical rows 的 fingerprint。两次自动 retry 也都复用原 turn、route 和节点。后续 Static/Dynamic 必须继续使用同一 mapping 和 fingerprint。

最新结果同时验证了限制：8/8 只冻结 episode 数量和身份，不保证两个节点最终 request 数、token 数或 wall time 相等。c=16 的 8200/8201 分别收到 232/138 个逻辑请求和 6.72M/3.56M prompt tokens。因此 matched 报告必须继续列每节点 episodes、requests、prompt/completion tokens、KV、waiting 和 preemption，不能只写“8/8 balanced”。

QPS 6～20 仍无 clean calibration；不过现有 Agent run 的 lifetime QPS 均映射到 0.5 档，所以它不影响当前原始 Dynamic 路径是否可执行。它仍限制更高流量场景的结论外推。

### P1：Dynamic 的特征不适配 live Agent

#### 1. output_tokens 使用请求上限

当前 proxy 使用 max_tokens 或 max_completion_tokens 作为期望输出。Agent 请求上限通常为 16384，但真实 completion 可能只有几十到几百 token，因此 workload 分类会严重偏离真实情况。

候选改进是根据最近若干轮真实 completion 建立 rolling median、p90 或简单预测器，同时保留 requested_max_tokens 供对照。

#### 2. QPS 是 lifetime average

当前 QPS 定义为：

    stack 启动以来的累计请求数 / 累计运行时间

工具执行和 Modal 等待会稀释这个数值。它既不是瞬时 arrival rate，也不是 LLM scheduler 的实际压力。

应先同时记录：

- lifetime average QPS；
- 最近 1、5、10 秒 sliding-window arrival rate；
- active episodes；
- waiting requests；
- 每节点 running requests 和 KV usage。

复现阶段先保留原定义，不直接改变决策。

#### 3. 小于 512 tokens 的 short-append bypass

当前 input_tokens < 512 时直接选择 PPD。大量 Agent tool observation 都落在这个范围，因此请求绕过 calibration lookup，Dynamic 容易退化为 x=1。

后续应分别测试：

- 原始阈值 512；
- 阈值 128 或 256；
- 完全关闭 bypass；
- 根据实测 append 分布选择阈值。

#### 4. Huge context 外推

原始 calibration 的 large context 只覆盖到约 4096 tokens，而 live Agent 可以达到 155K。当前 huge context 主要沿用 large 的趋势外推，属于明显的分布外决策。

后续可增加 agent-derived context calibration，但论文原始复现阶段先不修改。

#### 5. Offline workload 与 Agent workload 不一致

原始 matched benchmark 是固定两轮、固定输入输出、没有工具间隔；真实 Agent 是多轮、有 reasoning、有动态 tool observation 和 Modal 延迟。

因此 offline calibration 可以验证 Dynamic 流程，却不能直接证明其策略适合 live Agent。

### P2：缺少 TTL-aware cache policy

当前 PPD cache loss 后主要是 KV 被驱逐、下一轮重新计算，没有明确的 KV offload 层，也没有根据 Agent 预计返回时间进行 TTL-aware eviction。

将 Continuum 集成进来至少需要：

- 给 conversation 或 cache blocks 增加 TTL/预计复用时间；
- 由 Agent 或 proxy 传递预计工具间隔；
- scheduler 在内存压力下优先驱逐已过期或低复用价值 KV；
- 保留 PPD connector 和 NCCL KV transfer；
- 分别记录 eviction、offload、restore 和 recompute，不能全部归类为 cache miss。

## 五、2026-09-06 corrected Dynamic 实验结果

### 1. 实验与数据完整性

正式 run：

`/root/vllm-ppd/logs/runs/dynamic_original_corrected_c36_20260906T045214Z`

主要参数保持原始 Dynamic baseline：Qwen3-30B-A3B-Thinking-2507、BF16、2P_2D、`ENABLE_PPD_MODE=true`、155648 context、131072 episode completion budget、16384 per turn、temperature=0、36 workers、36 concurrency、SWE-bench 前 36 条。

| 指标 | 结果 |
|---|---:|
| traces | 36/36 |
| driver wall | 2608.7 s（43 分 29 秒） |
| trace 覆盖窗口 | 2596.3 s |
| termination | 31 finished / 5 max_steps |
| resolved / WA / timeout | 7 / 29 / 0 |
| logical LLM steps | 591 |
| physical proxy attempts | 594 |
| tool calls | 611 |
| cumulative prompt tokens | 14,009,720 |
| completion tokens | 702,189 |

`cumulative prompt tokens` 会重复计算每轮携带的历史，仅表示服务端实际处理的请求 token 总量，不是 36 个对话的唯一文本长度。36 个 episode 最终上下文 token 之和为 1,062,019。

所有 591 个 response 都是 `think+tool`；555/555 个相邻 transition 的 reasoning 在下一轮 request 中逐字回放成功。594/594 个 proxy attempt 使用 `hf_suffix_chat_template_v1` 精确 append estimator。

完整产物：

- `analysis_v2.txt`：逐 episode、逐 step、工具与 cache-loss 表；
- `aggregate_summary.json`：run-wide 机器可读聚合；
- `server_logs/`：本次 2P_2D 四节点和 proxy 日志快照；
- `metrics.csv`、`proxy_timing.jsonl`、36 个 schema-v3 Agent trace。

### 2. Dynamic 决策结果

| 路由/决策 | 次数 |
|---|---:|
| Turn 1 `pd` | 36 |
| Turn 2+ `ppd_dynamic` physical attempts | 558 |
| Turn 2+ 选择 PPD | 558 |
| Turn 2+ 选择 PD | 0 |
| `ppd_direct` | 0 |

558 次物理决策包括 555 个真实相邻 Agent transition 和 3 次 tool-choice 自动重试。原始 Dynamic 再次 100% 选择 PPD，退化为 `x=1`。

推断的决策来源为：

| decision source | 次数 | 占 Turn 2+ |
|---|---:|---:|
| `short_append_bypass` | 482 | 86.4% |
| `huge_context_extrapolation` | 58 | 10.4% |
| `calibration_lookup` | 18 | 3.2% |

精确 append 的 p50/p90/max 为 53/1261/9833 tokens。proxy 的 context 估计 p50/p90/max 为 24440.5/39286/46943 tokens；这里 context 仍是 chars/4 近似，append 才是模型 tokenizer 的精确值。

lifetime QPS 的 p50/max 只有 0.279/0.312，558/558 全部映射到 calibration 的 0.5 档。请求用 16384 作为 output feature，而真实 Turn 2+ completion 的 p50/p90/max 只有 660.5/2199/10685。按请求上限分类得到 531 次 `very_long_gen` 和 27 次 `mid_bal`；改用事后真实 output 后则分散到多种 workload。由此确认 max-token feature 和 lifetime-QPS 定义均不适配闭环 Agent。

### 3. Cache loss、节点压力与延迟

retention 定义为：

`next_cached_tokens / previous(prompt_tokens + replayed completion_tokens)`

| 指标 | 结果 |
|---|---:|
| 可判断 transitions | 553 |
| retention < 90% | 379（68.5%） |
| retention < 50% | 366（66.2%） |
| cached tokens 恰为 992 的 loss | 334/379 |
| 涉及 episodes | 36/36 |
| event-level lost-token 总和 | 8,095,621 |

lost-token 总和会在同一前缀被反复驱逐时重复计数，应解释为“跨 transition 的重算暴露量”，不是唯一 KV 容量，也不能直接换算成节省的字节或时间。

cache loss 时的中位观测为：同节点另有 14 个活跃 episode、waiting=6、KV usage=99.7%。loss 后下一轮 LLM latency p50=70.6 s，而 retention≥90% 时 p50=11.6 s。这是很强的相关性证据，但不是严格因果分解；端到端 LLM 时间同时包含排队、prefill/recompute 和 generation。

| 解码节点 | episode | requests | loss events | preemptions | active-window KV p50/p90 | waiting>0 | waiting p50/max |
|---|---:|---:|---:|---:|---:|---:|---:|
| 8200 | 16 | 196 | 89 | 53 | 89.2% / 97.8% | 48.4% | 0 / 7 |
| 8201 | 20 | 398 | 290 | 254 | 90.3% / 97.9% | 87.7% | 5 / 10 |

16/20 的 episode 数差异并不大，但 8201 恰好承载了更长的轨迹，request 数达到 8200 的两倍。固定 conversation affinity 使最初看似温和的哈希偏差被 episode 长度放大。两个节点 active window 内 GPU utilization p50 都为 96%，说明 c=36 已把 decode 侧推入持续压力区。

每步 LLM latency p50/p90/p99 为 51.4/146.6/357.7 s，最大逻辑 step 为 931.9 s。最大三个逻辑 step 都包含一次严格 tool-call JSON 解析失败及自动重试，所以不能把 931.9 s 当成单次物理生成延迟；最长失败物理 attempt 为 656.6 s。

与旧的 `dynamic_live_c36_20260829T0016Z` 只能做诊断性比较：旧 run 的 prompt p50 为 10309，本轮为 24684；旧 run 有 34 次 cache loss，本轮为 379 次。两轮的 Dynamic 都选择 PPD，差异主要说明当前 reasoning replay 和不同 Agent 轨迹显著提高了常驻上下文与 cache 压力，不能解释成 Dynamic 算法本身退化了。

### 4. Tool/Modal 与 TTL 含义

所有 episode 累加时间中，LLM 占 97.0%，工具执行占 1.9%，toolbox setup 占 1.0%。有效工具调用的代表性分布为：

| operation | count | p50 | p90 | max |
|---|---:|---:|---:|---:|
| editor:str_replace | 176 | 1.68 s | 2.93 s | 4.04 s |
| editor:view | 91 | 2.10 s | 2.75 s | 4.52 s |
| shell:python | 53 | 3.75 s | 7.03 s | 8.08 s |
| shell:pytest | 5 | 3.19 s | 3.75 s | 5.57 s |
| shell:grep | 3 | 3.01 s | 3.01 s | 3.03 s |

Modal sandbox create p50/max 为 0.854/0.964 s，toolbox setup p50/max 为 11.2/29.8 s。本轮没有早期实验中几十秒的简单 grep/view，工具层不是主要 wall-clock 瓶颈。

发生 cache loss 的 transition 中，工具间隔 p50/p90 为 1.64/3.77 s；109/379 的间隔小于 0.1 s，135/379 小于 1 s。小于 0.1 s 的一部分来自本地 format error，但排除含 format error 的 6 个 episode 后，其余 30/30 episode 仍有 239 次 loss、累计 4,479,439 event-level lost tokens。

因此 TTL 的目标不能只写成“长工具执行时保留 KV”：

- 需要覆盖工具返回后请求已经到达、但仍在 scheduler queue 中等待的时间；
- TTL 或 priority 应结合 recent arrival、waiting、active episode 和 KV pressure；
- 需要分别记录 eviction、offload、restore 和 recompute，当前 `cached_tokens` 只能证明本地命中丢失；
- 应在 clean Agent contract 下比较 Static PPD、Static PPD+TTL、Dynamic 和 Dynamic+TTL。

### 5. Agent 污染项与 PPD 侧新问题

611 次工具调用中有 215 次本地 `format_error`，全部是 `editor:insert` 缺少 `new_str`，占 35.2%。它们集中在 6 个 episode；其中 5 个 episode 运行满 50 steps，共产生 214 次错误，另 1 个 episode 单次错误后恢复。错误调用在 Modal 之前被拦截，单次约几毫秒，但后续重复 LLM 生成显著增加负载。

排除这 6 个 episode 后，30 个 clean episode 仍有 239 次 cache loss，因此 eviction 现象是真实 serving 问题；但总 wall、request 数、节点失衡和 19.4% resolved 均受到 Agent loop 污染，不能作为最终算法对比数字。

另有两个 PPD 数据契约问题：

1. 3 个 tool-choice retry 均成功恢复，但 proxy 把同一 `client_request_id` 的 retry 当作新 turn，turn counter 分别从 1→2、3→4、4→5。未来应以 run/episode/client_request_id 做幂等路由，retry 复用原 logical turn 和 server assignment。
2. 36/36 个 Turn 1 的 producer 内部 `prefill_status=500`，外部 decode 最终均成功。proxy 的 prefill 子请求把 `max_tokens` 强制设为 1，却保留 `tool_choice=required`；producer 因而在 KV 已产生/转移后继续严格解析必然截断的工具 JSON。当前是“副作用成功、HTTP 语义失败”，应让 prefill-only 请求跳过工具输出解析。

此外，本轮使用非流式请求，逐请求 TTFT 不可用。`t_total_s` 足以分析整体延迟和 cache-loss 相关性，但若后续要复现论文 TTFT 指标，需要 streaming 或服务端 request-level TTFT 数据。

### 6. 可得出的结论和不能得出的结论

可以得出：

- calibration loader 修正后，原始 Dynamic 的 Agent 路径可运行；
- 本 workload 上原始 Dynamic 仍完全退化为 `x=1`；
- reasoning 被真实回放后，上下文显著增长，c=36 会产生普遍而严重的本地 KV loss；
- cache loss 与 KV 满载、waiting、preemption 和较高 LLM latency 同时出现；
- 工具/Modal 不是本轮主要性能瓶颈；
- TTL 值得测试，但工具时长不是充分的 TTL 特征。

不能得出：

- Dynamic 比 static 更快或更慢，因为没有同一 Agent commit、同一轨迹的静态对照；
- 19.4% resolved 是 PPD 算法质量，因为 5 个 episode 被 schema loop 污染，live Agent 本来也不是严格 matched A/B；
- 70.6 s 全是 KV 重算开销，因为当前没有把 queue、recompute 和 decode 单独计时；
- cache miss 是 offload/restore，因为当前 baseline 没有 Continuum offload 路径，现有证据只支持“本地 cache 未保留并需要重新处理”。

### 7. c=36 后的并发定标假设及其更新

corrected Dynamic c=36 中 retention<90% 已达 68.5%，两个 decode 节点 active-window KV p90 约 98%，最重节点有 87.7% 的采样点 waiting>0 和 254 次 preemption。因此 c=36 只适合作为 overload/stress 上界，不适合作为唯一主对照点。

当时根据 reasoning replay 前后 prompt p50 从 10309 增至 24684，曾用 `36 × 10309 / 24684 ≈ 15.0` 粗略选择 c=16 作为首个校准点。这个选择已完成实测，但新结果表明“总并发 × 全局中位上下文”不足以定 knee：conversation affinity 将 c=16 分成局部 11/5，使一台节点严重 loss、另一台几乎无 loss。

因此旧的 5%/20% loss 选点阈值仍可作为工程准则，但必须在 deterministic、最好平衡的 sample→node mapping 下应用。当前 c=16 的结论和新的实验顺序见下一节及第七节。

## 六、2026-09-06 strict-editor Static x=1、matched16 c=16 结果

### 1. 实验配置与完整性

正式 run：

`/root/vllm-ppd/logs/runs/static_x1_editor_split_matched16_c16_20260906T123720Z`

参数为 Qwen3-30B-A3B-Thinking-2507、BF16、2P_2pD、静态 `x=1`、155648 context、131072 episode completion budget、16384 per turn、temperature=0、SWE-bench Verified 前 16 条、16 workers、concurrency=16。两个 pD 节点各有 156672-token KV capacity。

| 指标 | 结果 |
|---|---:|
| traces | 16/16 |
| driver wall | 1667.5 s（27 分 47 秒） |
| trace 覆盖窗口 | 1593.0 s |
| termination | 15 finished / 1 max_steps |
| resolved / WA / timeout | 0 / 16 / 0 |
| logical LLM steps | 297 |
| physical proxy attempts | 298 |
| tool calls | 322，全部成功 |
| cumulative prompt tokens | 7,522,657 |
| completion tokens | 381,785 |
| final-context token sum | 572,190 |

`cumulative prompt tokens` 会在每轮重复统计历史，只代表 server 实际收到的各请求 prompt 之和。16 条样本恰好全部来自 astropy；同样的前 16 条在旧 corrected Dynamic c=36 中也全部为 WA，因此 0/16 不是 editor split 已知回归，但这个连续切片不能代表完整 SWE-bench 质量。这里应把它作为 serving profile set，而不是模型能力评测。

run 目录现在包含 16 个 schema-v3 trace、`analysis.txt`、`aggregate_summary.json`、`metrics.csv`、`proxy_timing.jsonl` 和已固化的 `server_logs/`。

### 2. Agent、工具与 Modal

- 297/297 response 都是 `think+tool`；
- 281/281 个可验证的相邻 transition 精确回放 reasoning；
- 322/322 工具调用为 `ok`，`format_error=0`；
- 工具分布为 replace 193、view 50、shell python 24、grep 14、create 13、insert 9、undo 4、submit 15。

| operation | count | p50 | p90 | max |
|---|---:|---:|---:|---:|
| editor:str_replace | 193 | 1.65 s | 2.80 s | 5.46 s |
| editor:view | 50 | 2.08 s | 2.40 s | 4.36 s |
| editor:insert | 9 | 2.75 s | 4.37 s | 5.40 s |
| editor:create | 13 | 1.84 s | 2.35 s | 3.25 s |
| shell:python | 24 | 3.42 s | 6.08 s | 10.57 s |
| shell:grep | 14 | 2.46 s | 3.49 s | 3.98 s |

Modal 也没有早期的几十秒长尾：sandbox create p50/max 为 0.864/0.910 s，app lookup 为 0.899/1.090 s，fs read p50/p90/max 为 0.543/1.334/3.092 s，fs write为 0.880/1.358/1.989 s。

LLM 每步 p50/p90/p99 为 41.0/111.5/243.4 s；最大逻辑 step 755.0 s 来自一次 631.9 s 的严格 tool-call 解析失败，加 123.1 s `tool_choice=auto` 重试，不能当成单次物理推理。全 run 的 episode 累加时间中，LLM/工具/setup 分别占 95.0%/3.8%/1.1%。

### 3. 路由与仍存在的协议异常

| 路由 | 次数 |
|---|---:|
| Turn 1 `ppd` | 16 |
| Turn 2+ `ppd_direct` | 282 physical attempts |

297 个逻辑 request 对应 298 个 proxy row，因为有 1 个 tool-choice retry。首次物理 attempt 返回 500，重试成功；proxy 仍把它从 turn 4 记成 turn 5，说明 retry logical-turn 幂等性问题尚未修复。

此外，16/16 个 Turn 1 的 producer 子请求仍记录 `prefill_status=500`，但外部 decode 均成功。原因仍是 prefill-only 请求把生成上限设为 1，却保留 `tool_choice=required`：KV transfer 已发生，随后严格 JSON tool-call 解析失败。这不是本次 16 个 episode 的外部失败，但会污染错误率和日志语义，进入 PPD 修改阶段后应修复。

### 4. Cache loss 与节点倾斜

retention 定义保持为：

`next_cached_tokens / previous(prompt_tokens + replayed completion_tokens)`

281 个相邻 transition 中有 280 个可判断；tool-choice retry 的首次失败 response 没有 usage，因此按 analyzer 契约记为 unknown，未被错误当成 warm-cache 命中。

| 指标 | 结果 |
|---|---:|
| 可判断 transitions | 280 |
| retention < 90% | 140（50.0%） |
| retention < 50% | 131（46.8%） |
| 涉及 episodes | 12/16 |
| cached tokens=976 的 loss | 110/140 |
| event-level lost-token sum | 3,271,354 |

`lost-token sum` 是每个 transition 暴露的重复重算量；同一前缀可能被多次计数，不能解释成唯一 KV 容量或直接节省量。

| pD | episodes | requests | known/loss | active-window KV p50/p90/max | waiting>0 | waiting p50/max | preemptions |
|---|---:|---:|---:|---:|---:|---:|---:|
| 8200 | 11 | 184 | 171 / 139（81.3%） | 88.9% / 97.9% / 100% | 75.7% | 2 / 5 | 110 |
| 8201 | 5 | 114 | 109 / 1（0.9%） | 49.4% / 68.6% / 82.3% | 0.5% | 0 / 1 | 0 |

8200/8201 的 final-context token sum 分别为 376533/195657；它们不是同一时刻的 resident KV，不能直接与容量相除，但配合 11/5 同时启动、KV=100%、waiting 和 preemption，足以解释两节点完全不同的 cache 行为。8200 episode duration p50 为 1362 s，8201 为 593 s；该差异同时受任务轨迹影响，不能全部归因于 serving。

压力与 loss 的共现非常强：

- `waiting_max=0` 的 111 个 transition 中仅 1 个 loss；`waiting_max=3..5` 的 107 个 transition 全部 loss；
- 同节点窗口内另有 0～3 个 active episode 时 80/80 无 loss；另有 5～9 个时 72/72 loss；
- loss transition 的 KV usage max p50 为 99.2%、waiting max p50 为 4；retained transition 分别为 63.9% 和 0；
- loss 后 LLM p50/p90 为 62.3/115.5 s，retained 时为 13.5/64.1 s，median 相差 4.6 倍；
- loss 与 retained 两组的前序工具时间 p50 为 1.71/1.81 s，几乎相同。

这些是强相关证据，不是独立因果分解：当前 LLM time 同时包含 queue、prefix recompute/prefill 和 decode。但它已足以排除“长工具调用是主要驱逐原因”。

### 5. 对 concurrency 和 TTL 的含义

总 concurrency=16 实际变成局部 concurrency 约 11/5。本 run 因此同时给出一个过载节点和一个近似无 loss 节点，是很有价值的 TTL/调度诊断样本；但它没有定位均衡条件下的 knee。

每个 pD 只有 156672 tokens KV。当前 prompt p50 已是 25432，且一轮可再生成数千 token；从容量量级看，平均 8 个长对话/节点的 c=16 很可能已经是 stress 点，平均 6 个/节点的 c=12 更接近 knee 候选。这个估计不能替代固定映射后的实测，因为 episode 长度和到达间隔差异很大。

TTL policy 可以改变“内存紧张时保留谁”，所以当前 loss 形态支持继续测试 TTL；但仅设置过期时间不会创造显存。当 11 个长上下文同时争用 156672 tokens 时，要显著降低总重算还可能需要：均衡 admission、KV offload/restore、压缩，或降低本地并发。Continuum 路径若包含 offload，必须把 eviction、offload、restore 和 recompute 分开记录。

### 6. 本轮可以和不能得出的结论

可以得出：

- strict editor contract 已通过真实并发 workload，旧 format-error 污染被清除；
- Static `x=1` 的 cache loss 在局部并发和 KV 压力升高时出现，并伴随 waiting/preemption/LLM 长尾；
- 工具和 Modal 不是当前 wall-clock 瓶颈；
- c=16 可作为 routing-skew stress 样本，但不是已确认的均衡 knee。

不能得出：

- Static 比 Dynamic 快或慢，因为尚无相同 editor contract、稳定 sample→node mapping 的 Dynamic pair；
- c=16 在任意 hash seed 下都会有 50% loss；
- loss 后多出的约 49 s median 全是 KV recompute；
- TTL 单独即可消除 loss，或 0/16 score 是 serving 算法导致。

## 七、2026-09-07 frozen-map Static x=1、16 episodes、c=16 结果

### 1. 配置与完整性

正式 run：

`/root/vllm-ppd/logs/runs/stage3_balanced_static_c16_20260907T032957Z`

参数为 Qwen3-30B-A3B-Thinking-2507、BF16、2P_2pD、Static `x=1`、155648 context、131072 episode completion budget、16384 per-turn max output、temperature=0、SWE-bench Verified 前 16 条、16 episodes 和 concurrency=16。启动前清空了 KV/state，并固定使用与 c=2 相同的 strict 8/8 map；manifest 记录 vllm-ppd `11dccb5d6` 与 uni-agent `449f80f4383b`。

| 项目 | 结果 |
|---|---:|
| traces | 16/16 |
| Agent wall / manifest wall | 1741.9 / 1750.4 s（约 29 分钟） |
| termination | 14 finished / 2 max_steps |
| resolved / WA / timeout | 2 / 14 / 0 |
| logical requests / physical attempts | 370 / 372 |
| logical routes | 16 `ppd` / 354 `ppd_direct` |
| retry | 2，均成功恢复 |
| routing fingerprint rows | 372/372 一致 |
| frozen assignment | 8200 与 8201 各 8 episodes |

16/16 个 Turn 1 producer 请求都使用 `same_prompt_tool_choice_none_max_tokens_1_v1` 并返回 200，说明 prefill-only strict-parser 污染已修复。372/372 个 physical attempt 都能精确 join episode/request 和 metrics window；370 个成功 response 都有 engine metrics/TTFT，两个失败 attempt 没有 response metrics 属于预期。Agent 的 370 个最终成功 step 全部收到 `ppd_request_metrics`。

两次初始失败分别为：一次耗时 411.4 s 的 opaque HTTP 500，以及一次耗时 709.0 s 的已知 required-tool 截断 JSON HTTP 400。两者均以相同 `client_request_id`、logical turn、route 和 8200 节点用 `tool_choice=auto` 重试成功；所以它们扩大了逻辑 step 的 max latency，但没有增加 logical turn 或改变 affinity。opaque 500 的 server log 没有 traceback，当前不能进一步归因，后续应将 response body 和异常类型继续保留。

### 2. Episode 数均衡，但实际工作量仍不均衡

| pD | episodes | logical requests | prompt-token sum | completion sum | known/loss | retention<50% | KV max p50/p90 | waiting>0 | preemptions |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 8200 | 8 | 232 | 6,715,216 | 227,315 | 223 / 122（54.7%） | 90 | 95.1% / 99.95% | 156/232 | 58 |
| 8201 | 8 | 138 | 3,559,030 | 167,696 | 130 / 1（0.8%） | 0 | 77.2% / 93.7% | 12/138 | 0 |

全局 353 个可判断 transition 中有 123 个 retention<90%（34.8%），90 个低于 50%，涉及 8/16 个 episode；event-level lost reusable tokens 合计 2,686,318。该总和会重复计算同一前缀的多次丢失，只代表跨 transition 的重算暴露量，不是唯一 KV 容量。

这次 routing contract 本身工作正常：每个 sample 的节点身份可跨配置复现，episode 数也正好 8/8。但偶数槽位恰好产生了更多、更长的轨迹，使 8200 的 request 数比 8201 多 68%，prompt-token 工作量多 89%。这不是路由随机漂移，而是 live Agent trajectory imbalance。后续 matched A/B 可以公平复用这组身份，但不能把它描述成资源负载均衡。

### 3. Cache loss 时卡在哪里

以下均比较“当前请求相对上一轮 reusable prefix 已 loss”与 retention≥90% 的请求：

| 当前请求观测 | cache loss p50 | retained p50 | 解释 |
|---|---:|---:|---|
| previous tool time | 1.388 s | 1.442 s | loss 前工具并不更慢 |
| same-node other active episodes | 5 | 4 | 并发压力略高 |
| KV usage max | 96.7% | 77.6% | loss 与接近满载共现 |
| waiting max | 2 | 0 | loss 组进入排队 |
| engine queue | 6.042 s | ~0 s | 明显 scheduler queue |
| engine prefill elapsed | 3.616 s | 0.134 s | 约 27 倍，符合 prefix 重算/stall |
| engine TTFT | 12.253 s | 0.176 s | 约 70 倍 |
| engine decode | 22.067 s | 14.902 s | generation 也变慢，但差距较小 |
| Agent LLM wall | 36.755 s | 15.306 s | 中位数约 2.4 倍 |

loss 组 queue/prefill/TTFT 的 p90 分别为 12.35/12.29/22.21 s。这里 `prefill_elapsed` 包含重算、stall 和 preemption，不是纯 GPU kernel 时间；但结合大量 uncached tokens、58 次 preemption、KV≈100% 和 retention loss，已经能把主要附加瓶颈定位到“排队 + prefix 重新处理”，而不是工具、Modal 或单纯 decode。

c=16 因而是有效的 Static PPD+TTL stress 点。它能检验 TTL/priority/offload 是否减少重算，但固定 TTL 不会创造显存：同节点仍有约 5 个其他活跃 episode、KV 接近满载时，策略必须决定保留谁，或配合 offload/admission control。

### 4. Agent、thinking、工具与可复现性

- 370/370 response 都是非空 reasoning + tool call，assistant `content` 为空；354/354 个后续 logical request 标记 reasoning replay。
- prompt tokens p50/p90/max 为 28488/42075/54006，仍远低于 155648 context；本轮没有 context-limit 退出。
- completion tokens p50/p90/max 为 621/1678/15385，而每次请求上限都是 16384；原始 Dynamic 的 output feature 仍严重高估多数请求。
- 375 次工具调用的 handler 状态全部为 `ok`，耗时 p50/p90/max 为 1.44/2.52/4.44 s；toolbox setup p50/p90/max 为 11.01/15.78/21.50 s。
- episode 累加时间中，LLM/tool/setup 分别为 13324.7/579.7/191.8 s，占 94.4%/4.1%/1.4%。工具/Modal 不是主瓶颈。

Agent 语义污染仍很明显：262 次 `replace_text` 中至少 257 次 observation 为 `No replacement was performed`，另有 5 次错误路径；2 个 episode 运行到 max_steps，只有 14 次 submit。schema split 已消除缺字段/format error，却没有解决模型构造错误 `old_str` 和失败动作循环。工具统计必须继续把 handler success 与 semantic success 分开。

使用相同模型、前 16 条数据、editor/reasoning contract 和 temperature=0 的 pre-fix c=16 为 0/16 resolved，本轮为 2/16；两轮之间仍有 routing/profiling 代码和 batching/轨迹差异。greedy decoding 也会因执行条件分叉，所以这 12.5 percentage-point 差异不能归因于 PPD 性能改善；它再次说明 live Agent 适合生态 profiling，不适合作为严格 matched 的质量比较。

### 5. 本轮结论与下一步

可以得出：

- strict frozen routing、prefill-only contract、request metrics 和 retry 幂等性均通过完整 live workload；
- c=16 是明确的 cluster stress 点，且压力集中在实际 token/request 更重的 8200；
- cache loss 的附加延迟主要落在 queue、prefill/recompute 和 TTFT，而非前序 tool time；
- c=16 适合后续 Static PPD+TTL 压力对照，但不应作为唯一 knee。

下一步保持 sample 总数为 16、mapping 和所有模型参数不变，仅把 global concurrency 降为 12；不要把 `--limit` 改成 12，否则 sample set 会变化。若 c=12 的全局 loss 落到约 5%～20%且不再持续 queue/preempt，就将它冻结为 knee，并在同一配置下运行原始 Dynamic。若 c=12 仍明显过载，再测 c=8～10。c=16 的 matched Dynamic 可作为额外 stress 对照，但优先级低于确定 knee。

## 八、建议的后续顺序

### 阶段 1：冻结 Agent contract（已完成）

Agent 行为的 CPU contract smoke、c=2 live smoke 和正式 c=16 均已通过。后续 serving A/B 固定 uni-agent `449f80f4383b` 及当前 YAML；该新增提交只携带 matched-routing identity 与服务端指标透传。

### 阶段 2：修复并冻结实验路由契约（已完成并通过 Static live 验收）

实现位于 vllm-ppd `ddffde8fcc9b` 和 uni-agent `449f80f4383b`。2026-09-07 c=2 smoke 已确认 `/status.routing_contract` 的 path、fingerprint、`map_assignment_count=16`、`map_strict=true`，并验证 frozen-map 首轮分配与后续 affinity。`legacy_hash` 仅用于旧行为诊断；pre-fix c=16 run 继续保留为 routing-skew stress baseline。

### 阶段 3：确定 knee/stress，并跑 matched Dynamic（进行中）

c=2 contract gate 和 deterministic 8/8 Static c=16 已完成。c=16 的 global loss=34.8%、8200 loss=54.7%，并伴随 58 次 preemption、queue p90=9.0 s 和 TTFT p90=16.2 s，已冻结为 stress。

下一轮保持前 16 条和同一 map，设置 total episodes=16、global concurrency=12：

- 若 loss 约 5%～20%且没有持续 waiting/preemption，定义为 knee；
- 若仍高于 20%并持续排队，再测 concurrency=8～10；
- knee 确定后，用完全相同 sample、mapping、concurrency 和参数跑原始 Dynamic 2P_2D；
- c=16 Dynamic 只作为可选 stress pair，不替代 knee pair。

原始 Dynamic 预计仍 100% 选择 PPD；这轮的目的，是在严格控制 routing 和 Agent contract 后验证它是否等价退化为 `x=1`，并量化 Dynamic decision overhead，而不是期待它自动解决 eviction。

### 阶段 4：补齐 PPD 请求级 profiling/协议语义（核心 Static 路径已 live 冻结）

以上字段已在 vllm-ppd `ddffde8fcc9b` 实现：logical retry、prefill-only parse contract、P/D request phases、request TTFT/preemption/tokens、decision source/cell、requested/actual output、lifetime/window QPS、两类 active concurrency以及 P/D waiting/KV 均进入 schema 化 artifact。

c=2 与 c=16 live run 已确认成功 response 的 P/D engine/ORCA/load fields、非流式 TTFT 和等量 request-profile rows。c=16 还真实覆盖了两次 required→auto retry：同一 logical request 复用 turn、route 和节点，logical/physical 统计为 370/372。这里测得的是 request-level phase elapsed 和 lost reusable tokens，不把它们误称为纯 kernel recompute time。尚未覆盖的只剩 Dynamic 特有的 decision source/cell 组合；它将在 matched Dynamic run 中验收。

### 阶段 5：Agent-compatible Dynamic ablation

一次只改变一个变量：

1. 用历史真实 completion 预测 output tokens；
2. lifetime QPS 改为 sliding-window arrival 或 queue/KV pressure；
3. 调整或关闭 512-token short-append bypass；
4. 增加真实 Agent context calibration；
5. 加入 next-arrival、工具间隔和 queue residency 特征。

### 阶段 6：集成 TTL

最终建议比较：

| 配置 | 用途 |
|---|---|
| 2P_2D，x=0 | PD 基线 |
| 2P_2pD，x=1 | 静态 PPD 基线 |
| 原始 Dynamic | 论文复现 |
| Agent-adapted Dynamic | Agent 特征适配 |
| Static PPD + TTL | 单独验证 TTL |
| Dynamic + TTL | 最终组合方案 |

所有配置固定模型、Agent commit、sample set、temperature、routing mapping、concurrency 和 token 参数。性能主表最好使用录制的 request/tool-timing replay；live Agent 作为生态真实性与 end-to-end 验证。

推荐实际工作流：

    Agent contract frozen
        -> deterministic matched routing
        -> balanced static c=16; c=12 if needed
        -> original Dynamic at the same point(s)
        -> request phase / decision profiling
        -> Agent-compatible Dynamic ablations
        -> Static PPD + TTL
        -> Dynamic + TTL
