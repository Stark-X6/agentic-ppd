# Agent 修复阶段总结与 PPD/TTL 改动计划

更新时间：2026-09-06

## 结论

小规模 live smoke 和修正 calibration loader 后的原始 Dynamic c=36 baseline 已完成。结果证明 reasoning replay、精确 append 计数、三源 profiling join 和 Dynamic 路由链路都能工作；原始 Dynamic 在本次 live Agent workload 上仍然 100% 选择 PPD，行为退化为静态 `x=1`。

本轮也得到此前没有出现过的强 cache-pressure 证据：553 个可判断 transition 中有 379 个 retention 低于 90%，其中 366 个低于 50%。cache loss 与接近满载的 KV、排队以及长 LLM 延迟明显相关。这使 TTL 成为有依据的后续方向，但当前数据也表明“只按工具执行时间设置 TTL”不够，因为不少 cache loss 发生在极短工具间隔之后。

c=36 中曾出现 215 次缺少 `new_str` 的本地 `format_error`，集中在 6 个 episode，并使 5 个 episode 达到 `max_steps=50`。这些错误没有调用 Modal，却制造了额外 LLM 请求和人为 cache 压力。editor 模型可见接口现已拆成五个具有无条件必填字段的工具；CPU contract smoke 已通过，下一步只需先做小规模 live smoke，确认模型端行为，再建立同一 Agent commit 下的静态对照。

当前代码状态：

- uni-agent 分支：fix/agent-tool-contract
- uni-agent 本地最新提交：4f25cdf Split editor into strict operation-specific tools
- vllm-ppd 分支：fix/agent-tool-contract
- vllm-ppd 本地最新提交：f40722674 Classify split editor tools in profiling
- 两个仓库的 agentic-ppd-baseline-v1 tag 均保持不变

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

CPU contract smoke 已验证：五个 schema 的 required 集合、缺少 `insert_text.new_str` 时不发生 sandbox I/O、view/create、replace/insert、跨 wrapper 两次 undo 和新插入行号语义。当前环境未安装 pytest，所以本轮使用独立 async smoke 加 `py_compile`/`git diff --check`；仍需一个 c=2 live smoke 验证模型实际选择和 arguments。

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

### 已完成的 live smoke

修正后的 c=2 smoke 位于：

`/root/vllm-ppd/logs/runs/dynamic_smoke_corrected_flat_c2_20260906T044159Z`

验证结果：

- 2/2 episode 自然结束，无 timeout 和请求错误；
- 18/18 step 均有 reasoning；
- 16/16 可回放 transition 与下一轮实际 request 完全一致；
- 18/18 工具调用成功，15/15 editor 调用含 path；
- Turn 1 为 2 次 `pd`，Turn 2+ 为 16 次 `ppd_dynamic`；
- 18/18 proxy row 使用精确 tokenizer，且 `ppd_mode_enabled=true`；
- 没有误注册为 `ppd_direct`。

因此 reasoning replay、role=tool append、精确 tokenizer、cached-token retention 计算和 Dynamic 路由链路均已验证。

该 c=2 smoke 发生在 editor 拆分之前，而且没有触发 `insert`。现在直接 CPU contract smoke 已覆盖五个新接口；后续 live smoke 的任务是验证模型是否能稳定选择新工具并生成完整 arguments，而不是再次依赖自然探索来覆盖全部操作。

### 非阻塞技术债

- Agent 的 context-limit 预检查仍使用 observation 字符数近似；请求成功后才由 vLLM 返回的真实 prompt_tokens 校正。
- Trace 为复现目的保存每轮完整请求历史，因此文件大小接近二次增长。
- 语义错误工具调用仍需要模型或 Agent policy 改进。
- Modal 抖动只能拆分和统计，无法由 Agent 代码彻底消除。

因此当前 Agent 状态应定义为：reasoning、trace、token、Modal 边界和 editor 的代码级契约已验收；editor split 还差一个小规模 live smoke，之后即可冻结用于新的 matched serving 实验。

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

### P0：原始 Dynamic baseline 已重跑，但同版本静态对照缺失

QPS≤4 的 72 对 matched 数据已完整加载，corrected Dynamic c=36 也已完成。558 次 Turn 2+ 物理决策全部选择 PPD，因此“原始 Dynamic 在该 live Agent workload 上退化为静态 `x=1`”已经得到验证。

不过，这次 run 使用了新的 reasoning replay、精确 append tokenizer 和 editor schema；旧的 static x=1 c=36 不是同一 Agent contract。两者的上下文、生成轨迹和节点负载不匹配，不能用旧静态结果做严格性能归因。若要比较 Static、Dynamic 和 TTL，仍需用修好 editor 接口后的同一 commit 重跑 static x=1。

QPS 6～20 仍无 clean calibration，但本轮 lifetime QPS 最大只有 0.312，全部映射到 0.5 档，因此高 QPS calibration 缺失没有直接影响本轮决策。

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

### 7. reasoning replay 后的并发定位

应该降低主实验的 concurrency，并把 c=36 保留为 overload/stress 上界，而不是直接作为唯一 A/B 点。corrected c=36 中 retention<90% 已达 68.5%，两个 decode 节点 active-window KV p90 约 98%，8201 有 87.7% 的采样点 waiting>0，并累计 254 次 preemption；此时 queue saturation、节点长尾和 KV eviction 已高度耦合，TTL 即使有效也可能被整体过载掩盖。

reasoning replay 前的旧 c=36 run 与新 run 不可直接比较，但可用于粗略定标：prompt p50 从 10309 增至 24684，约为 2.39 倍。按“并发 × 中位上下文”保持相近常驻 token 压力估算：

`36 × 10309 / 24684 ≈ 15.0`

因此 c=16 是 editor 修复后最合理的首个校准点，而不是凭经验直接回到 c=12，也不应直接继续用 c=36。这个换算只是起点，因为轨迹长度、哈希分配、生成长度和工具间隔都变了。

建议固定 SWE-bench 前 36 条和所有模型参数，仅改变并发上限：先 c=16；若 retention<90% 少于约 5%，再试 c=20、必要时 c=24；若 c=16 已超过约 20% 且 waiting 明显，再补 c=12。最终保留两个点：

- knee/正常负载点：有可测 cache loss，但没有 timeout、持续排队或大规模 preemption，用于 Static/Dynamic 的主要对照；
- stress 点：cache loss 明显但尚未全面饱和，用于放大 TTL 效果；c=36 只作为过载参照，除非 editor 修复后的新 run 证明它已不再饱和。

阈值 5%/20% 是实验选点准则，不是论文算法参数。所有比较必须在 editor split 后重新采集；旧 c=36 的 379 次 loss 可以证明问题存在，却不能作为修复后算法的严格基线。

## 六、建议的后续顺序

### 阶段 1：验收 Agent editor contract

五工具拆分和 CPU contract smoke 已完成。先跑 c=2 live smoke，重点检查工具名、arguments、format_error、undo、reasoning replay 和 analyzer 分类；通过后冻结 Agent commit。

### 阶段 2：重新选择并发并建立同版本对照

固定前 36 个 SWE-bench episode，不因并发变化而改变样本集合。先以 static x=1、c=16 运行；根据 cache loss、waiting、preemption 和 timeout 决定是否升到 c=20/24 或降到 c=12。选定 knee 后，在完全相同的模型、Agent commit、样本、temperature 和 token 参数下分别重跑 static x=1 与原始 Dynamic。必要时另保留一个非饱和 stress 点用于 TTL；不要把旧的 polluted c=36 当严格性能对照。

### 阶段 3：增强决策 profiling，但不改变策略

每个请求增加：

- decision_source
- selected_context_class
- selected_workload
- selected_qps_point
- calibration_found
- requested_max_tokens
- previous_actual_completion
- lifetime_qps
- window_qps
- active_episodes
- waiting
- kv_usage
- queue_time / recompute-prefill time / decode time

同时修复 retry 的 logical-turn 幂等性和 prefill-only 内部 500。

### 阶段 4：Agent-compatible Dynamic ablation

一次只改变一个变量：

1. 使用历史 completion 预测 output tokens；
2. lifetime QPS 改为 sliding-window arrival rate 或 queue/KV pressure；
3. 调整或关闭 512-token bypass；
4. 增加真实 Agent context calibration；
5. 加入工具间隔、next-arrival 和 queue residency 特征。

### 阶段 5：集成 TTL

最终建议比较：

| 配置 | 用途 |
|---|---|
| 2P_2D，x=0 | PD 基线 |
| 2P_2pD，x=1 | 静态 PPD 基线 |
| 原始 Dynamic | 论文复现 |
| Agent-adapted Dynamic | Agent 特征适配 |
| Static PPD + TTL | 单独验证 TTL |
| Dynamic + TTL | 最终组合方案 |

所有配置应固定同一模型、Agent commit、SWE-bench episode 集合、temperature、concurrency 和 token 参数。性能 A/B 最好进一步使用已录制的 request/tool-timing replay；live Agent 则保留用于生态真实性验证。

推荐实际工作流为：

    c=2 live editor smoke
        -> static x=1 concurrency calibration (start at c=16)
        -> same-version Static/Dynamic at the selected knee
        -> decision-source and queue-time profiling
        -> Agent-compatible Dynamic ablations
        -> Static PPD + TTL
        -> Dynamic + TTL
