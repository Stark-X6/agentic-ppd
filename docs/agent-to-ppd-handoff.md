# Agent 修复阶段总结与 PPD/TTL 改动计划

更新时间：2026-09-06

## 结论

严格 editor contract 的 live smoke 和正式 Static PPD `x=1`、16 episodes、concurrency=16 实验均已完成。Agent 数据契约可以冻结：正式 run 的 297 个 response 全部是 `think+tool`，281/281 个可回放 transition 精确带回 reasoning；322/322 个工具调用成功，旧 editor 的 `insert` 缺字段错误降为 0。

正式 c=16 确实产生了清晰 cache pressure：280 个可判断 transition 中 140 个 retention 低于 90%，其中 131 个低于 50%。但这些 loss 几乎全部集中在一个节点。conversation affinity 将 16 个 episode 分成 8200 上 11 个、8201 上 5 个；8200 为 139/171 loss、110 次 preemption，8201 仅为 1/109 loss、0 次 preemption。因此，这次 run 是有效的“路由倾斜/单节点压力”基线，却不能单独证明均衡负载下 c=16 就是合适的 knee。

工具端已不是瓶颈：工具时延 p50/p90/max 为 1.70/3.32/10.57 s，LLM 时延为 41.0/111.5/755.0 s；按 episode 累加时间，LLM 占 95.0%，工具占 3.8%，toolbox setup 占 1.1%。cache loss 后下一轮 LLM p50 为 62.3 s，retention≥90% 时为 13.5 s；两组前一轮工具时间 p50 分别为 1.71 s 和 1.81 s。当前 loss 更接近 KV 满载、排队和 preemption，而不是“工具执行太久”。

在重跑 matched Dynamic 前，需要先固定 sample→node 映射。当前 proxy 的节点选择使用 Python `hash()`，conversation hash 又包含每次不同的 `run_id`/epoch，节点列表还依赖注册顺序；所以重启后即便使用相同 16 个样本，也不能复现本次 11/5 分配。若直接比较 wall time，路由倾斜会成为比 Dynamic 策略更大的混杂变量。

当前代码状态：

- uni-agent 分支：fix/agent-tool-contract
- uni-agent 最新提交：4f25cdf Split editor into strict operation-specific tools
- vllm-ppd 分支：fix/agent-tool-contract
- vllm-ppd 最新提交：ac30818d0 Use v2 analyzer for run-scoped Agent profiles
- 两个代码仓库的 agentic-ppd-baseline-v1 tag 均保持不变

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

### 已完成的 live smoke 与正式验收

修正 append/reasoning 后的 Dynamic c=2 smoke 位于：

`/root/vllm-ppd/logs/runs/dynamic_smoke_corrected_flat_c2_20260906T044159Z`

它验证了 Turn 1 `pd`、Turn 2+ `ppd_dynamic`、精确 append tokenizer、reasoning replay 和三源 join；2/2 episode 自然结束，没有 timeout。

editor 拆分后的 Static c=2 live smoke 位于：

`/root/vllm-ppd/logs/runs/static_x1_editor_split_smoke_c2_20260906T073603Z`

验证结果：

- 2/2 episode 正常结束；
- 33/33 工具调用成功，无 `format_error`；
- 31/31 相邻 transition 的 reasoning 精确回放；
- 路由为 2 次 Turn 1 `ppd` 和 31 次 Turn 2+ `ppd_direct`；
- 小规模下没有 cache loss。

正式 Static c=16 run 又覆盖了 322 次工具调用，包括 193 次 replace、50 次 view、13 次 create、9 次 insert 和 4 次 undo，全部成功。它不仅验证 schema 能被模型接受，也验证五个 wrapper 在真实多轮任务中的数据面行为。

因此当前 Agent 状态应定义为：reasoning、trace、token、Modal 边界和 editor contract 均已 live 验收，可以冻结用于 serving 对照实验。

### 非阻塞技术债

- Agent 的 context-limit 预检查仍使用 observation 字符数近似；请求成功后才由 vLLM 返回的真实 prompt_tokens 校正。
- Trace 为复现目的保存每轮完整请求历史，因此文件大小接近二次增长。
- 语义错误工具调用仍需要模型或 Agent policy 改进。
- Modal 抖动只能拆分和统计，无法由 Agent 代码彻底消除。

这些技术债不阻塞 serving profiling。后续若没有发现新的 trace 契约错误，不再改变 Agent commit；所有 Static、Dynamic 和 TTL 对照都固定使用当前 Agent 版本。

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

### P0：静态基线已建立，但跨配置节点分配尚不可复现

严格 editor contract 下的 Static `x=1`、c=16、前 16 条 SWE-bench run 已完成，原始 Dynamic 则仍只有旧 editor contract 下的 c=36 结果。因此下一项算法实验仍是重跑相同样本的原始 Dynamic。

不过，正式 Static run 暴露出 matched A/B 更基础的数据契约问题。当前 `comprehensive_proxy.py`：

1. 用 `run_id | conversation_id | PPD_PROXY_EPOCH` 生成 conversation hash；Static 和 Dynamic 的 run_id 不同，因此 hash 本身不同；
2. 用 Python `hash(conv_hash) % len(servers)` 选节点；未设置 `PYTHONHASHSEED` 时，不同 proxy 进程的 hash secret 不同；
3. `prefill_instances`/`decode_instances` 从 ZMQ 注册字典直接转 list，索引对应的物理端口受注册先后影响。

这三点意味着重启 stack 后不能复现相同 sample→decode-node 映射。本次 16 个 episode 恰好分成 11/5；公平二项哈希下出现 11/5 或更极端分配的概率约为 21%，并非罕见异常，但在每节点只有 156672 KV tokens 时足以把一侧推入持续 preemption、另一侧保持几乎无 loss。

matched 实验前应把“状态隔离 key”和“稳定路由 key”分开：

- state key 继续包含 run_id/epoch，防止跨 run 继承 turn counter；
- routing key 使用固定 comparison set、稳定 episode identity 和显式 routing seed，经 `hashlib` 计算，不使用 Python `hash()`；
- 所有 server list 先按地址排序；
- proxy log 写入 routing key、bucket、seed、排序后的候选节点和最终 assignment；
- paired Static/Dynamic 使用同一份预先确定的 mapping。若要强制 8/8，应在实验前生成并冻结 mapping，而不是根据运行结果临时挑 seed。

这是实验可复现性修复，不改变 Dynamic 决策本身。保留一个兼容开关即可继续复现原始 hash 行为。另一种不改代码的办法是每个配置跑多个 routing seed 并报告置信区间，但在当前 GPU 成本和 n=16 下不划算。

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

## 七、建议的后续顺序

### 阶段 1：冻结 Agent contract（已完成）

CPU contract smoke、c=2 live smoke 和正式 c=16 均已通过。固定 uni-agent `4f25cdf` 及当前 YAML，不再把 Agent 改动混入 serving A/B。

### 阶段 2：修复并冻结实验路由契约

在不改变 PPD/Dynamic 决策的前提下：

1. 分离 state key 与 deterministic routing key；
2. 排序 server 地址，使用稳定 `hashlib` 和显式 routing seed；
3. 记录完整 mapping 元数据；
4. 预先冻结前 16 条在两台 decode 上的 assignment，并让 paired configs 共用。

原始 hash 行为保留开关，当前 c=16 run 保留为 pre-fix stress baseline。若坚持完全不改 proxy，则至少每个配置跑多个 hash seed，不能拿单次 11/5 与另一单次随机分配比较。

### 阶段 3：确定 balanced knee/stress，并跑 matched Dynamic

仍使用相同前 16 条、模型、BF16、temperature=0 和 token 参数。建议在固定 8/8 映射下先复测 Static c=16：

- 若 loss 仍高于约 20%且持续 waiting/preemption，把 c=16 定义为 stress，再测 c=12 作为 knee；
- 若 c=16 只有可测但不饱和的 loss，可直接作为 knee；
- 选定点后立刻用完全相同 mapping 跑原始 Dynamic 2P_2D。

原始 Dynamic 预计仍 100% 选择 PPD；这轮的目的，是在严格控制 routing 和 Agent contract 后验证它是否等价退化为 `x=1`，并量化 Dynamic decision overhead，而不是期待它自动解决 eviction。

### 阶段 4：补齐 PPD 请求级 profiling/协议语义

不改变决策，先补：

- retry 复用 logical turn 和 assignment；
- prefill-only 请求不触发 required-tool JSON 解析；
- queue time、recompute/prefill time、decode time和 request-level TTFT；
- decision_source、selected calibration cell、requested/actual output、lifetime/window QPS、active episodes、waiting 和 KV usage。

这些字段是判断 TTL 是否真正减少 recompute，而不只是改变排队顺序的前提。

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
