# Agent 修复阶段总结与 PPD/TTL 改动计划

更新时间：2026-09-08

## 结论

阶段 2（实验路由契约）、阶段 4（请求级 profiling/协议语义）和原始 Dynamic 的 c=14 live 验收均已完成。Dynamic run 的 16/16 traces、245/245 physical rows 和 frozen-map fingerprint 全部完整；16 个 Turn 1 为 `pd`，225 个 Turn 2+ 全部为 `ppd_dynamic`，`ppd_mode_enabled=true`、affinity miss=0、`ppd_direct=0`。4 次 required→auto retry 均复用原 logical identity 并恢复成功。至此原始 PPD Dynamic 流程已经复现，而不是“代码没跑起来”。

最重要的结果是：原始 Dynamic 在该 live Agent workload 上 225/225 次选择 PPD，策略上完全退化为 Static `x=1`。其中 191/225（84.9%）由 `<512` token short-append bypass 直接决定，31/225（13.8%）把 huge context 外推到 large calibration，只有 3/225（1.3%）直接命中同 context class 的 calibration；所有请求又都因 lifetime QPS≤0.254 而落到 0.5 档。requested output 恒为 16384，而后续实际 output p50/p90/max 仅 721/1839/4166，导致 213/225（94.7%）被归为 `very_long_gen`。因此退化来自原始特征/策略契约，而不是 append tokenizer 或最后一条 `role=tool` 适配失败。

Dynamic 的 retention<90% 只有 1/222（0.45%），低于 Static c=14 的 15/264（5.7%）；但这不能解释为 Dynamic 消除了驱逐。Dynamic 同时只有 241 vs 284 个 logical requests、5.81M vs 7.30M prompt tokens、293.7K vs 359.9K completion tokens，KV p90/max 也只有 76.2%/86.8% vs 96.8%/100%。唯一 loss 发生在 KV=54.4%、waiting=0、queue≈0，更像 prefix-divergence/孤立候选，而非容量驱逐。原始 Dynamic 实际做出了和 Static 相同的 PPD 动作，只是本次 live trajectory 更轻。

两轮不能做严格性能因果比较：16 个相同 sample 只有 1 个 step 数相同，绝对 step 差 p50/p90/max 为 4.5/20/48；Dynamic 有 8 个 sample 步数更少、7 个更多。Dynamic wall 虽从 1473.3 s 降到 1216.6 s（-17.4%），但 logical requests、prompt tokens 和失败生成时间分别下降 15.1%、20.4% 和 41.2%。两轮均为 1/16 resolved，且 resolved 的都是 `astropy-14309`，说明这一次最终质量相同，但不能把墙钟/cache 差异归因于 Dynamic。下一阶段应把原始 Dynamic 作为已完成的 reproduction baseline，进入 Agent-compatible Dynamic ablation；TTL 则继续用 c=14 knee 和 c=16 stress，并在归因前补 block eviction/offload/restore 或 token-LCP 证据。

当前代码状态：

- uni-agent 分支：fix/agent-tool-contract
- uni-agent 最新提交：449f80f4383b Add stable routing identity to agent profiles
- vllm-ppd 分支：fix/agent-tool-contract
- vllm-ppd contract 提交：ddffde8fcc9b Freeze matched routing and request profiling contracts
- vllm-ppd 最新提交：11dccb5d6 Fix small-sample percentiles and TTFT detection
- uni-agent 比 GitHub origin 超前 1 个本地提交，vllm-ppd 超前 2 个，均尚未 push
- 两个代码仓库的 agentic-ppd-baseline-v1 tag 均保持不变

阶段 3 已完成 c=2 gate、deterministic 8/8 Static c=12 healthy、c=14 knee、c=16 stress，以及同配置原始 Dynamic c=14。下一步进入阶段 5：逐项验证 output 预测、QPS/压力信号、short-append bypass 和 huge-context calibration 的 Agent 适配；原始 baseline 不再重复运行。

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

### P0：matched routing 已修复并通过 Static/Dynamic live 验收

旧 Static c=16 的 11/5 节点倾斜来自三项混杂：state hash 包含 run/epoch、Python `hash()` 进程随机性，以及 ZMQ 注册顺序。该历史 run 保留为 routing-skew stress baseline，但不再作为 matched Static/Dynamic 比较依据。

当前实现已经：

- 分离 stack-local state key 与跨配置稳定的 `routing_key=instance_id|rollout=N`；
- 使用显式 seed 的 SHA-256 和排序后的 server addresses；
- checked in 前 16 条任务的 8/8 frozen mapping；
- 用 strict 模式对缺 key、缺 role 和 slot 越界 fail closed；
- 在每个 timing row 与 `/status` 中记录 fingerprint、slot、assignment source 和候选节点。

c=2 live smoke 验证了 frozen-map slot 与后续 conversation affinity；完整 c=16 又验证了 16/16 首轮、354/354 后续 affinity 和 372/372 个 physical rows 的 fingerprint。c=12 验证了 16/16 首轮、291/291 后续 affinity 和 308/308 个 physical rows；Static c=14 验证了 16/16 首轮、268/268 后续 affinity 和 289/289 个 physical rows；Dynamic c=14 又验证了 16/16 首轮、225/225 后续 affinity 和 245/245 个 physical rows。四个正式 run 合计 12 次自动 retry，全部复用原 turn、route 和节点。后续 ablation/TTL 必须继续使用同一 mapping 和 fingerprint。

最新结果同时验证了限制：8/8 只冻结 episode 数量和身份，不保证两个节点最终 request 数、token 数或 wall time 相等。c=16 的 8200/8201 分别收到 232/138 个逻辑请求和 6.75M/3.56M prompt tokens，c=12 则是 162/145 和 4.67M/3.75M。因此 matched 报告必须继续列每节点 episodes、requests、prompt/completion tokens、KV、waiting 和 preemption，不能只写“8/8 balanced”。

QPS 6～20 仍无 clean calibration；不过现有 Agent run 的 lifetime QPS 均映射到 0.5 档，所以它不影响当前原始 Dynamic 路径是否可执行。它仍限制更高流量场景的结论外推。

### P1：Dynamic 的特征不适配 live Agent

#### 1. output_tokens 使用请求上限

当前 proxy 使用 max_tokens 或 max_completion_tokens 作为期望输出。Agent 请求上限通常为 16384，但真实 completion 可能只有几十到几百 token，因此 workload 分类会严重偏离真实情况。

Dynamic c=14 已量化这个偏差：225 个后续请求的 requested output 全是 16384，而重试成功后实际 output p50/p90/max 为 721/1839/4166，中位数高估 22.7 倍；213/225（94.7%）请求因此被归为 `very_long_gen`，其余 12 个为 `mid_bal`，完全没有 paste 类。候选改进是根据最近若干轮真实 completion 建立 rolling median、p90 或简单预测器，同时保留 requested_max_tokens 供对照。

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

复现阶段已保留原定义并完成验证：Dynamic c=14 后续请求的 lifetime QPS p50/p90/max 为 0.233/0.251/0.254，全部映射到最低 0.5 calibration 档；同一批请求观测到的 5 秒 QPS p50/p90/max 为 0.4/0.6/1.0。下一阶段才把 sliding-window arrival、waiting/KV 或 queue pressure 作为单变量 ablation。

#### 3. 小于 512 tokens 的 short-append bypass

当前 input_tokens < 512 时直接选择 PPD。Dynamic c=14 中 191/225（84.9%）个后续 append 触发这一 bypass；实测 append p50/p90/max 为 74/1370/30707 tokens，说明并非所有工具输出都短，但绝大多数轮次根本没有查询 calibration。其余 34 个长 append 也全部选择 PPD。

后续应分别测试：

- 原始阈值 512；
- 阈值 128 或 256；
- 完全关闭 bypass；
- 根据实测 append 分布选择阈值。

#### 4. Huge context 外推

原始 calibration 的 large context 只覆盖到约 4096 tokens，而 live Agent 可以达到 155K。Dynamic c=14 的 225 个后续请求中有 222 个被分类为 huge context；31 个非 bypass 请求通过 `huge_context_extrapolated_from_large` 使用 large cell，只有 3 个仍在 large class 并直接命中 calibration。两个被选中的 cell 都是 QPS=0.5，score 只有两个固定正值 0.588/0.648，因此 34/34 仍选 PPD。

后续可增加 agent-derived context calibration；论文原始复现已经在不修改策略的前提下完成。

#### 5. Offline workload 与 Agent workload 不一致

原始 matched benchmark 是固定两轮、固定输入输出、没有工具间隔；真实 Agent 是多轮、有 reasoning、有动态 tool observation 和 Modal 延迟。

因此 offline calibration 可以验证 Dynamic 流程，却不能直接证明其策略适合 live Agent。

### P2：缺少 TTL-aware cache policy

c=16 中与 KV≈100%、waiting、queue 和 preemption 共现的 loss 主要表现为下一轮重新计算；当前没有明确的 KV offload 层，也没有根据 Agent 预计返回时间进行 TTL-aware eviction。但 cached-token retention 只观测“前缀没有命中”，不能单独证明“容量驱逐”；c=12 的两个低压 loss candidate 还可能来自 reasoning/tool-call 重渲染后的 token-prefix 不连续。

Dynamic c=14 唯一的 retention<90% 事件发生在 KV max=54.4%、waiting=0、queue≈0，不能作为 TTL 可改善的容量驱逐证据；而 Static c=14/c=16 的压力簇仍是 TTL 实验目标。将 Continuum 集成进来至少需要：

- 给 conversation 或 cache blocks 增加 TTL/预计复用时间；
- 由 Agent 或 proxy 传递预计工具间隔；
- scheduler 在内存压力下优先驱逐已过期或低复用价值 KV；
- 保留 PPD connector 和 NCCL KV transfer；
- 分别记录 eviction、offload、restore 和 recompute，不能全部归类为 cache miss。

## 五至十、实验记录（已拆分）

2026-09-06 至 2026-09-08 的六轮 live Agent 实验结果已迁移至独立文档：

[Agent × PPD 实验记录](./agent-ppd-experiment-log.md)

新文档保留原第五至第十章的编号和完整内容，涵盖 corrected Dynamic c=36、strict-editor Static c=16、frozen-map Static c=16/c=12/c=14，以及 frozen-map 原始 Dynamic c=14。后续实验结果也统一追加到该实验记录；本 handoff 继续维护代码契约、当前问题和后续改动计划。

## 十一、建议的后续顺序

### 阶段 1：冻结 Agent contract（已完成）

Agent 行为的 CPU contract smoke、c=2 live smoke 和正式 c=16 均已通过。后续 serving A/B 固定 uni-agent `449f80f4383b` 及当前 YAML；该新增提交只携带 matched-routing identity 与服务端指标透传。

### 阶段 2：修复并冻结实验路由契约（已完成并通过 Static live 验收）

实现位于 vllm-ppd `ddffde8fcc9b` 和 uni-agent `449f80f4383b`。2026-09-07 c=2 smoke 已确认 `/status.routing_contract` 的 path、fingerprint、`map_assignment_count=16`、`map_strict=true`，并验证 frozen-map 首轮分配与后续 affinity。`legacy_hash` 仅用于旧行为诊断；pre-fix c=16 run 继续保留为 routing-skew stress baseline。

### 阶段 3：确定 knee/stress，并跑 matched Dynamic（已完成）

c=2 contract gate、deterministic 8/8 Static c=12 healthy、c=14 knee、c=16 stress，以及原始 Dynamic c=14 均已完成。Static 三点 global loss 为 0.7%/5.7%/34.8%；Dynamic c=14 的 225/225 Turn 2+ 全部选择 PPD，验证了原策略在 live Agent 上等价退化为 `x=1`。Dynamic run 的 routing、decision source/cell、tool-last-message、exact append 与 profiling contract 均通过，当前不补 c=13/c=15，也不重复原始 Dynamic。

Static/Dynamic 的 realized workload 明显分叉，因此本轮只作为 config-matched ecological reproduction；它不能提供严格的 serving-policy speedup。后续性能因果表需要 request/tool-gap replay。

### 阶段 4：补齐 PPD 请求级 profiling/协议语义（Static/Dynamic 均已 live 冻结）

以上字段已在 vllm-ppd `ddffde8fcc9b` 实现：logical retry、prefill-only parse contract、P/D request phases、request TTFT/preemption/tokens、decision source/cell、requested/actual output、lifetime/window QPS、两类 active concurrency以及 P/D waiting/KV 均进入 schema 化 artifact。

c=2、c=12、Static c=14、c=16 与 Dynamic c=14 live run 已确认成功 response 的 P/D engine/ORCA/load fields、非流式 TTFT 和等量 request-profile rows。c=12/Static c=14/c=16/Dynamic c=14 分别真实覆盖 1/5/2/4 次 required→auto retry，logical/physical 为 307/308、284/289、370/372 和 241/245；同一 logical request 始终复用 turn、route 和节点。Dynamic 特有的 `ppd_mode_enabled`、decision source/cell、exact append、last tool role 和 QPS snapshot 也已覆盖。这里测得的是 request-level phase elapsed 和 lost reusable tokens，不把它们误称为纯 kernel recompute time，也不把低 retention 单独视为 eviction 证明。

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
        -> static c=12 healthy; c=14 operational knee; c=16 stress
        -> original Dynamic at c=14: reproduced, degenerates to x=1
        -> request phase / decision profiling: frozen
        -> Agent-compatible Dynamic ablations
        -> Static PPD + TTL
        -> Dynamic + TTL
