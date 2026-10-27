# Agentic PPD：Mac 本地项目接手总结

整理日期：2026-10-27。

用途：用户准备在 Mac + VS Code 中建立本地项目，用本文接续研究和开发。本文汇总此前长对话、当前仓库文档与迁移验收结果，不要求先导入 Codex 历史数据库。本文不是原始逐条聊天记录；涉及实现时仍应核对对应代码，涉及实验数值时仍应使用原始 artifact。

路径约定：本地工作目录建议为 `~/Projects/agentic-workspace`，四个仓库作为同级目录。本文应放在 `agentic-ppd/docs/`。服务器历史路径以 `/root/...` 表示，不可直接当作 Mac 路径执行。

## 1. 接手时首先需要知道的状态

研究目标是在多轮工具调用 Agent 场景下改进 PPD serving：先理解真实请求、KV 可复用性和节点负载，再改路由，最后研究与 TTL 的协同。不要把提升 P 利用率、提高 Remote 占比或减少 eviction 数量本身当作最终目标。

已完成：Agent 与工具契约修复；固定 placement 和重试身份；请求级 profiling；原始 Static/Dynamic 复现；当前 workload 的 c=12/14/16 工作点分析；新状态变量的语义冻结；默认关闭的节点只读观测实现及 CPU 验证。

尚未完成：只读观测的原生 GPU/IPC 联合验收；production proxy 的 pending 身份对账；新状态查表的校准和路由实现；严格 request/tool-gap replay；TTL pin/unpin。

当前第一版研究路线已经选定：保留 PPD 的实测 TTFT/TPOT 评分，改用实际 KV 前缀和节点工作量匹配 calibration。AMPD 前置路由和精细延迟预测是后续备选；TTL 暂时只保留接口。最终主指标与约束阈值尚未冻结，不能宣称“优先优化 TTFT”已经达成共识。

最新基础设施进展是 AutoDL → Vast 的单卡迁移。新卡上模型和工具调用 smoke 已通过，但没有在新硬件重跑原来四卡 2P:2D 实验，也没有完成新 observer 的 GPU 验收。旧 c=14 knee 和旧 calibration 不能直接视为新硬件的有效性能依据。

用户现在选择在 Mac 建本地项目，以详细总结接续；Mac 已经有自己的 Codex 历史。不要默认覆盖、合并或切换其 `~/.codex`。

## 2. 仓库、版本与文档位置

| 仓库 | 作用 | 当前开发分支 | 本次核对的提交 |
|---|---|---|---|
| `Stark-X6/agentic-ppd` | 总体文档、实验管理、版本清单 | `main` | `5d11504b5fb36a255a96e48a1361bc5c30e81c3f` |
| `Stark-X6/vllm-ppd` | proxy、路由、benchmark、profiling、vendored vLLM | `fix/agent-tool-contract` | `c0b7a0aec5b3bbcf0523cae6e92d4af0f41513dc` |
| `Stark-X6/uni-agent` | ReAct、工具、Modal、请求和 Agent trace | `fix/agent-tool-contract` | `449f80f4383b1acf446bf5b7a6e2afce56093813` |
| `Hanchenli/vllm-continuum` | TTL 相关参考实现 | `main` | `316a58794a6ff86b216e579b74fd56ed0c5a911f` |

`agentic-ppd` 没有把另外两个组件作为源码目录或子模块收进去。三个开发仓库保持独立 Git 历史，不要把 uni-agent 复制进 vllm-ppd。

`vllm-ppd/vllm-source/` 已作为普通 tracked 文件纳入仓库；正常 clone 会带上修改过的 vLLM 源码。不要另外安装 upstream vLLM 后误以为运行的是本 fork。

`uni-agent` 有递归子模块，迁移验收时为：

- `verl`：`1ae945592754cbeb1350cbe092fe6117070fd4c7`。
- `verl/recipe`：`e7f889574b8301cc0f0fc1d57c6d67f31ffeb689`。

现有 `agentic-ppd-baseline-v1` 是更早的冻结基线；`manifests/baseline-v1.yaml` 中的 PPD/Agent commit 不等于当前修复后版本。不要移动旧 tag。新实验应记录每个仓库的实际 commit、dirty diff、配置和 calibration 指纹。

优先阅读以下已存在的文档：

1. [当前交接与阶段状态](./agent-to-ppd-handoff.md)。
2. [实验记录与数字](./agent-ppd-experiment-log.md)。
3. [阶段 5 当前设计](./stage5-state-aware-routing-and-ttl-design.md)。
4. [早期问题与备选方案](./stage5-agent-compatible-dynamic-theory.md)：其中一些路线已后置，不能全部当作当前任务。
5. [Dynamic 通信与调用流程](./dynamic-inference-call-flow.md)。
6. [输入与结果语义 v1](../../vllm-ppd/docs/stage5_state_semantics_contract.md)。
7. [只读观察器实现与验收边界](../../vllm-ppd/docs/stage5_readonly_observer.md)。
8. [现有 serving contract](../../vllm-ppd/docs/agent_serving_contract.md)。

文档分工：handoff 更新当前结论、计划完成情况和边界；experiment log 保存各轮实验数据与解释。不要再把每轮实验全文堆回 handoff。早期文档中的 `/root/...` 绝对链接在 Mac 上需要按仓库位置定位。

## 3. 实验系统与基本概念

主要链路：SWE-bench task → uni-agent ReAct loop → PPD proxy → vLLM fork → 模型 → tool call → Modal sandbox → tool observation → 下一轮 LLM 请求。

原四卡环境为 A800-80GB ×4。P 在 GPU0/1、HTTP 8100/8101；D/pD 在 GPU2/3、HTTP 8200/8201；proxy 10001；控制面 ZMQ 30001；KV 数据面为 P2pNcclConnector。端口是历史约定，实际运行要读 config。

一个 episode 是对一个 SWE-bench task 的完整尝试；rollout 是一次具体运行轨迹，同 task 可有多个 rollout。一个 step 通常包含一次逻辑 LLM 请求及随后工具执行；重试会增加 physical attempts，但不应增加 logical turn。

`total episodes=16, concurrency=12` 表示总共跑 16 条、最多同时 12 条，并非顺序跑完一个再跑下一个。episode 可能正在等待工具，此时并不占一个实际 decode 序列，因此 episode concurrency、HTTP in-flight、scheduler running 和 decode batch size 不能互换。

当前单 episode 内多个工具调用按顺序执行。跨 episode 可以并发。用户怀疑的几十秒 grep/view 不能仅靠命令类别解释，必须拆 Modal 生命周期、RPC、执行、输出传输等边界。

当前数据是 SWE-bench Verified，迁移后 parquet 为 500 条；后期固定取前 16 条。早期交接中的“验证集”称呼不应替代对具体 parquet、split 和 sample IDs 的确认。

## 4. 模型与配置的演变

早期用过 Qwen3-Coder-30B-A3B、Qwen3-32B。Qwen3-32B 阶段 run9 是 3 个并行 rollout，27/50/34 steps，耗时约 889.7/692.4/1030.2 秒，分别按 context_limit/max_steps/context_limit 结束。它说明链路可运行，不代表 task 被解决，也不是当前模型的性能基线。

后来为研究 thinking 工作负载，切换到当前模型 `Qwen/Qwen3-30B-A3B-Thinking-2507`。用户要求 BF16，没有采用 FP8。曾讨论接近 256K/250K 的窗口；最终已验收的冻结值是 155648，不要把早期愿望写成已实现配置。

当前对照契约：

| 参数 | 值与含义 |
|---|---|
| model | Qwen3-30B-A3B-Thinking-2507 |
| dtype | BF16 |
| max_context_len / server max_model_len | 155648，单请求输入加生成的上下文约束 |
| max_total_tokens | 131072，整个 episode 累计 completion 预算，不重复累计历史 prompt |
| max_tokens_per_turn | 16384，单次生成上限，不是预测生成长度 |
| temperature | 0 |
| replay_reasoning | true |
| tools | 五个结构化 editor 工具及现有其他工具 |

模型上限、服务端窗口、单次生成上限和 episode 累计预算是不同量。历史上扩大这些限制才允许多 step，不意味着任何参数越大越好。

MoE 或 dense 本身不足以决定 KV 占用；应查看 attention 层数、KV heads、head dimension、dtype 和实际分配。换模型、reasoning 回放或上下文后，原 concurrency 工作点必须重新检查。

## 5. Agent 已完成的修复

### 5.1 长对话和截断

早期因 observation 累积导致 prompt 超过窗口、vLLM 返回 400。Agent 已增加预算检查和干净的 context_limit 退出；预检查仍含字符估算，成功请求后用真实 usage 校正。这是保留的技术债。

关键 tools-400 根因是 `tool_choice=required` 在生成结束后对输出做严格 JSON 解析：输出被 max_tokens 截断时，`TypeAdapter(list[FunctionDefinition]).validate_json(content)` 抛错，经上层转为 HTTP 400。它不是 tools 请求字段被 proxy 字符串化。对照实验用 max_tokens=1/512、直连/proxy、热/冷缓存逐步定位了生成后解析路径。

客户端对识别到的该类失败以 tool_choice=auto 重试；ReAct 再用 json.loads 丢弃无效 arguments。同一步复用 client_request_id。失败 generation 的时间是真实成本，不能因重试成功就从统计中删除。

### 5.2 Reasoning

assistant content 为空不代表模型没有生成内容：reasoning 和 tool_calls 可以占主要输出。trace 分别保存这些字段。

当前 serving 对照冻结 replay_reasoning=true。回放策略由 Agent、模型模板和 API 能力共同约束，不是“vLLM 自动把所有 thinking 塞进下一轮”。省略 reasoning 会改变下一轮 token 序列，可能使其之后的旧 KV 不再属于可复用连续前缀。物理显存 block 不连续与逻辑 token 前缀不连续是两回事。

prefix cache 内可能还保留请求结束后的 blocks，直到回收策略处理；省略 reasoning 不等于立即显式 evict 那部分 blocks。TTL 保护有价值的对象必须以实际下一轮可复用前缀为依据，不能保护所有历史输出。

即使回放 reasoning，也不能保证跨轮 token 前缀完全相同；工具 schema、模板、assistant 边界和最后一个 token 都会影响复用。需要 token-LCP 和 block 证据。

### 5.3 Tool schema 和 Modal

PPD 默认模型工具列表将旧 flat str_replace_editor 拆为 view_file、create_file、replace_text、insert_text、undo_edit。参数进入 Modal 前由统一 Pydantic schema 校验，绑定同 sandbox 时共享 undo history。replace_text 的 new_str 即便为空也必须显式给出；insert_line=0 表示开头，1 表示第一行之后。

工具正常返回只说明 handler/RPC 成功，不说明文件确实改变或任务解决。重复 view、old_str 不匹配等仍可能来自模型轨迹；semantic_outcome 尚未统一覆盖全部工具。

由于原云环境不能使用 Docker，任务执行采用 Modal sandbox。已补生命周期和 RPC 事件，但云端抖动无法由 Agent 代码完全消除。镜像首次构建约几十秒不能直接归因到每次 grep，也不能用简单 app log 推导所有工具等待。

### 5.4 Trace 与可复现性

trace 保存完整请求 attempts、原始 response、reasoning、tool calls、tokens、各阶段时间、工具参数/返回、Modal events、退出原因及关联 ID。完整历史重复保存会使文件体积接近二次增长，是当前有意保留的可审计性与体积权衡。

temperature=0 不保证完整轨迹确定：GPU batching、数值差异、重试、工具返回和 Modal 时序仍会分叉。live Agent 用于真实工作负载 profiling；严格 serving 因果比较仍需 request/tool-gap replay。不能只看同样样本/config 就认定两轮生成工作量相同。

## 6. PPD、路由与 profiling 已完成的工作

### 6.1 两条路径

Local：请求在固定 D/pD 上做缺失 prompt 的 prefill，然后继续 decode。可利用 D 已有前缀，省 P 排队和 P→D 传输，但追加 prefill 可能拖慢该 D 上其他 decode。

Remote：在固定 P 上 prefill，经 P→D 传输后回到原 D decode。P 可以利用自己保留的前缀；若 P 落后于 D，可能要重算大量历史。当前没有 D→P 的历史回传，因此不能假设 Remote 只算新 observation；也不能一律假设 P 全量重算。两条路径都需要 D 有最终 KV 容量。

正常多轮会话保持 P/D affinity；Dynamic 决定 prefill 在哪里做，不是每轮换一个 D。原始 Static x=1：Turn1 为 ppd，Turn2+ 为 ppd_direct。原始 Dynamic：使用 2P_2D + ENABLE_PPD_MODE=true，Turn1 为 pd，Turn2+ 应记录 ppd_dynamic，不能误注册成 ppd_direct。

### 6.2 Append 与 calibration loader

旧 proxy 低估工具 observation；已按最近 assistant 后的 suffix 结合模型 tokenizer/chat template 计数，支持最后一条 role=tool。实际 suffix 比字符估算更长，某些 observation 可达三万 tokens，并非所有工具响应都短。

这个修复仍不等于“完整 prompt 的精确历史/新增边界已全部解决”：独立 suffix 渲染的可加性需要进一步验证。旧 context_tokens 仍有字符估算，不能直接改名为新规范的 N/H/A。

Benchmark writer 和 calibration loader 的小数 QPS 命名已对齐；兼容旧整数名，并校验文件内部 config/workload/QPS。缺失 coverage 必须显式报告。

### 6.3 Placement 和 retry

run_id + conversation_id + proxy epoch 用于单 run 状态与 affinity；routing_key=instance_id|rollout=N 用于跨 run 的稳定实验分配。proxy 使用 seed + SHA-256 和排序后的节点地址，不依赖 Python hash 或节点注册顺序。

固定前16条的 map 位于 `vllm-ppd/configs/routing/swebench_verified_first16_balanced_2x2.json`，正式 matched run 使用 strict map。它保证总 episode 8/8 分配，不保证请求数、token 数或瞬时负载相等。

同一 logical request 的重试复用 turn、节点、route、特征与 calibration cell；物理 attempt 仍独立计时。代理累计到达数和窗口 QPS必须区分 logical/physical。

### 6.4 请求级 profiling

P prefill-only 请求保留完整 messages/tools，tool_choice=none、max_tokens=1、stream=false；vLLM 不因 tool_choice=none 排除工具 schema，从而保持 prompt 前缀并避开 required 输出解析。

vLLM response 的 ppd_request_metrics 提供 queue、prefill elapsed、decode elapsed、TTFT/residual、preemption 和 tokens。prefill elapsed 是阶段墙钟，包含 stall/重算等，不是纯 kernel 时间；P/D HTTP 时长也不是纯 NCCL 时间。

Proxy schema v4记录 logical/physical 身份、P/D HTTP 边界、engine metrics、ORCA/load、decision source/cell、requested/actual output、窗口 QPS 等。Analyzer 生成 request_profile.jsonl、request_profile.csv、analysis.txt，并关联 metrics 和 Agent trace。

Cache-loss 表比较 previous reusable tokens 与下一请求 cached tokens，给出 lost tokens、retention、同节点其他 episode/LLM 请求、queue/KV/LLM 延迟。retention 低只能证明前缀未命中候选，不能单独证明容量驱逐、offload 或 TTL 收益。

旧评分的 client-stream TTFT/TPOT、节点 engine TTFT，以及 poller 的 ITL 事件直方图不是同一统计量。D engine TTFT 不含 P 阶段。旧名 tpot 不可直接当作 request-mean TPOT；见语义契约。

## 7. 最关键的实验结论

### 7.1 Offline 与旧 Dynamic 全 Local

最初计划复现原始 360 点 matched offline benchmark。后来因耗时，先不测 QPS>10，进一步只保留 QPS≤4，并对污染点 clean rerun。最终 clean 目录是：

`/root/vllm-ppd/results/comprehensive_qwen3_thinking_bf16_155648_matched_clean_qps_le4`

144 个单策略结果对应 72 个 PD/PPD matched pair；历史上用 13 个 clean rerun 覆盖污染点。这个 144 不等于 144 个 matched pair。

CPU 可达性审计得到：

| 项目 | 结果 |
|---|---:|
| 实际 matched pairs | 72 |
| lookup cells | 180 = 2 context × 9 workload × 10 QPS |
| 实测 score 范围 | 0.0040941662 ～ 0.8661177467 |
| 实测 score≤0 | 0 |
| 缺失 pair、默认 PPD | 108 cells |
| 全表 Local / Remote | 180 / 0 |
| 关闭 short bypass 后扫描 | 4320 组特征，仍全 Local |

因此，单独改 output/QPS 输入或关闭 short bypass，旧表仍没有 Remote 动作。不能再用相同旧表反复跑昂贵 A/B 来证明新特征价值。该结果也不证明 x=1 在所有硬件、context、复用状态和负载下永远最优。

旧表有短历史锚点和 Agent 长上下文覆盖不足；high-QPS 缺失结果不能被默认值冒充成实测。曾有离线点成功率偏低、重启缓解：需区分单点失败、跨点残留/拥塞和缓存/请求状态，不能仅凭重启恢复认定唯一根因，也不能忽略失败样本后比较均值。

### 7.2 同 first16、同 map 的 Static 工作点

以下数字来自原四卡模型与配置，不是 Vast 单卡测量。retention 分母是可比较 transitions。

| 指标 | c=12 healthy | c=14 operational knee | c=16 stress |
|---|---:|---:|---:|
| logical / physical requests | 307 / 308 | 284 / 289 | 370 / 372 |
| retention<90% | 2/290，0.7% | 15/264，5.7% | 123/353，34.8% |
| retention<50% | 0 | 2 | 90 |
| KV max p90 / max | 79.8% / 88.7% | 96.8% / 100% | 99.5% / 100% |
| request preemptions | 0 | 2 | 58 |
| queue p90 | 0.014 s | 0.022 s | 8.988 s |
| prefill p90 | 0.196 s | 0.535 s | 7.645 s |
| TTFT p90 | 0.260 s | 0.631 s | 16.216 s |

c=14 是按本次预设压力标准选的 operational knee：开始出现有容量压力证据的 loss，但尚未发生 c=16 的持续排队和 preemption 崩塌。它不是数学上证明的最优点或通用并发上限。

早期曾逐步尝试 c=12/16/24/32/36、不同 episode 数和模型；reasoning 回放、editor 和路由契约变化后，早期数字不宜直接作为当前 paired baseline。后期将总 episode 固定16，再调 concurrency，才形成以上工作点。

### 7.3 原始 Dynamic c=14

关键 run：`/root/vllm-ppd/logs/runs/stage3_dynamic_original_c14_20260908T091501Z`。

- 16/16 traces，241 logical requests、245 attempts，4次重试均恢复。
- Turn1：16次 pd；Turn2+：225次 ppd_dynamic，实际225次全 Local、0次 Remote，无 ppd_direct 误标。
- short bypass 191次，占84.9%；其余34次查表仍选择 Local。
- append p50/p90/max = 74/1370/30707 tokens。
- requested output p50=16384，实际 completion p50=721，说明上限不能当预测。
- 225个后续请求最后一条消息均为 tool，suffix tokenizer 没有报错；多条 observation 也被包含。
- 222/225 被归为 huge context，却外推到旧 large calibration；所有选中 QPS 都是0.5。

Static c=14 的 Agent wall 为1473.3s，Dynamic 为1216.6s，但 Dynamic prompt-token sum 少20.4%、completion sum少18.4%，且动作仍全 Local。两轮都是1/16 resolved。不能把17.4%的wall下降称作Dynamic路由收益。

Dynamic仅有1/222的低retention事件，发生时KV压力较低、queue近零，没有充分容量驱逐证据。Static/Dynamic差异首先反映live trajectory分叉；相同map和temperature不能保证相同工作量。

## 8. 论文讨论与当前算法方向

### 8.1 已达成的取舍

PPD 的优势是用实测路径结果包含多种干扰，不必先精确分解 decode 干扰模型。但旧 workload/context/QPS 不足以区分真实 Agent 执行条件，尤其缺候选节点可复用前缀和当时的 decode/prefill 工作量。

AMPD 提供在线负载反馈与等待/传输建模的借鉴，其路由涉及 P 的 TTFT 余量、D 的 ITL 余量及后续成本比较；不是完全忽略 D 上 decode。它支持的双向 KV 传输会改变成本路径，本项目不能直接套用其执行假设。

用户与学长决定不依赖当前请求最终 output 长度预测；这是一项设计约束，并非“任何模型/应用中输出长度都不可能预测”的已证命题。滑动窗口仍可用于流量与延迟观测，但 QPS 不等于 GPU 负载。

当前第一版保留 PPD 实测查表。先不启用“P有TTFT余量就Remote，否则D有ITL余量就Local”等AMPD门控，也不实现精确F/W/X分解模型。保持固定2P:2D、固定D affinity，不同时改变资源比例或增加D迁移。

等权原评分为：

```text
S = (TTFT_PD - TTFT_Local) / TTFT_PD
  - (TPOT_Local - TPOT_PD) / TPOT_PD

有效 calibration 且未被 bypass 时：S>0 选 Local，S≤0 选 Remote/PD。
```

提高QPS不保证选择PD：P队列/传输压力可能使Remote更差；D密集decode而P空闲、且Remote重算可接受时，PD才更有希望。增加P:D改变资源与校准域，不作为当前路由调优手段。

最终评价需包含完整response/tool-ready、episode时间、累计LLM等待、ITL尾部、超时/goodput及任务质量。原score是第一版控制变量，不等价于最小化Agent完成时间。

### 8.2 规范输入

| 符号 | 含义 |
|---|---|
| N | 实际完整prompt token序列长度，含tools schema、模板及reasoning策略 |
| H | 同一完整序列中可证明的逻辑历史/setup边界 |
| A | 边界可证明时的N−H；否则H/A=null |
| C_P、C_D | 对本次序列匹配、ready、连续、符合引擎规则的有效前缀 |
| U_P、U_D | N−C_P、N−C_D |
| B_D | 当前具备继续decode条件的序列数，而不是len(running) |
| K_D | 上述同一集合中已有可用历史KV位置数之和 |
| Q_tok,P/D | 每个节点已接下且可执行/执行中的剩余prefill/recompute tokens |
| Q_att,P/D | 相应剩余attention query–key配对工作量 |

(N,U_P,U_D) 已隐含两端缓存历史长度。H/A用于解释逻辑历史与新增，不另设一个冗余查表轴。相同U但不同N的prefill成本仍可能不同。

普通生成请求考虑末token和block对齐后满足C<N、U≥1。未知状态用null，不冒充冷缓存0，也不能把hash登记当成GPU写入已完成。

在同步、非speculative、普通n=1的边界，某decode请求已接受G≥1个生成token时，已有KV位置数L=N+G−1；最后一个sampled token尚未作为下一步输入计算KV。K_D是逻辑读历史规模，不是物理显存去重占用。

积压中的每条工作区分E_ready、S_scheduled和T_target：

```text
0 ≤ E ≤ S ≤ T
inflight = S - E
unscheduled = T - S
remaining R = T - E

Q_tok = ΣR
Q_att = Σ[E*R + R*(R+1)/2]
```

初始prefill的T=N；preemption恢复可能包括已经生成但丢失KV的tokens，T可以大于原prompt。不能用调度时已提前增加的num_computed_tokens直接当E。Q_tok包含in-flight，不能重复加。blocked、等待远端KV、当前待路由请求不混入本地可计算积压。

Q_att是工作量代理，不是时间公式。它帮助区分相同剩余tokens在不同历史长度下的attention量，不证明prefill延迟严格服从某个二次函数。

### 8.3 最近尚未定案的讨论

用户担心新增轴导致offline点数爆炸，且prefill积压估计不够准确。当前没有最终确定分桶边界、采样点和总预算。

已讨论的候选包括：把两端积压压成少量状态档；只测trace中常见状态与路径边界；用近期到达token工作量/未完成请求数作低成本代理；把积压或TTFT/SLO差距放在查表外作为惩罚或在线修正。不能把这些候选写成已经实现或已经选定。

若路由仍使用offline测得的结果，但通过当前状态选cell，本身就具有在线状态感知；再加在线惩罚会改变实际决策规则，应单独命名和消融。不能仅凭“有窗口”就称其与原PPD或AMPD相同。

新calibration不应把所有原始字段做笛卡尔积，也不能把旧均值重新贴上新状态标签。需要同一背景、起始KV状态、当前请求的Local/Remote配对测量，报告失败与波动。具体简化方案仍是下一阶段的讨论任务。

## 9. 只读状态观测已经做到哪里

核心缓存索引、请求队列和调度元数据位于CPU侧，因此可以设计无需读取GPU KV tensor的查询。但是CPU侧可观测不等于零开销，也不等于任意线程读取都一致。

已实现默认关闭的 `/v1/ppd/observe`，开关 `VLLM_PPD_OBSERVER_ENABLED=1`；经API→AsyncLLM→EngineCore utility在owner-thread边界读取。维护独立block ready证明，处理登记、完成、取消、抢占代次和回收，不增加引用计数或touch LRU。

目前只支持明确的同步、单engine、TP=PP=DP=DCP=PCP=1、单full-attention KV group、无speculative/LoRA/多模态，connector限无connector或当前P2pNcclConnector。不支持的布局返回unavailable，不能为了得到数字隐式改变原服务调度模式。

API提供节点epoch、快照序号、覆盖标志、C/U、B/K、Q、blocked/unknown、ready/scheduled/target，以及原生request ID的presence。token-only probe标明canonical_chat_verified=false；还没有完成每条chat请求的精确渲染/身份自动对齐。

PendingLedger作为独立组件完成CPU测试，但尚未接入production proxy。必须验证X-Request-Id到原生engine request ID的映射、物理attempt去重、终止ACK和tombstone。请求从active snapshot消失不等于已完成；timeout也不等于engine停止。

CPU共138项测试通过：语义46、observer47、mock API19、pending22、calibration4。以下是假scheduler/cache与替代hash的微基准，不含native hasher、HTTP、IPC或GPU：

| 背景请求数 / probe tokens | state-only中位 | state+probe中位 | state+probe p95 |
|---|---:|---:|---:|
| 16 / 4097 | 0.060ms | 1.716ms | 1.745ms |
| 64 / 32769 | 0.185ms | 13.744ms | 68.878ms |
| 256 / 131073 | 0.736ms | 106.783ms | 161.351ms |

长prefix probe成本不可忽略，10Hz接口上限不是推荐轮询频率。需要真实off/on对照，再考虑复用hash、减少频率或缓存查询结果。

当前未实现TTL动作，观察器不会pin任何block。既有TTL能力字段仅表明disabled。状态快照也不是命中承诺：查询与执行之间仍可能被驱逐。

## 10. TTL 的研究边界

TTL希望利用Agent工具间隔保护下一次可能复用的KV；它也会减少可回收容量，导致其他请求排队或preemption。应比较额外减少的真实重算和额外阻止回收的容量×时间，而不是只追求retention更高。

TTL到期应解除保护，不等于立即evict，也不等于offload。共享block、传输引用、取消、episode结束、节点epoch和超时都需要可靠清理。

P和D的缓存时间轴不同：P往往较早完成prefill，D还会产生长reasoning/output。保护哪一端、保留多久，要结合下一轮路径选择；路由依赖缓存、TTL又改变缓存，存在自我强化和振荡风险。

当前没有实现完整Continuum论文策略；公开仓库中的固定阈值pin与论文完整效用估计器必须区分。也没有足够文献证据可笼统写“PPD/AMPD已经证明TTL移植到PD必然退化”：AMPD中的co-located Continuum对照不等于这项实验。

后续顺序建议：先在Static x=1独立验证TTL，再在路由与TTL各自验收后组合。固定TTL、Continuum-style效用、优先级和route-aware retention应作为可区分方案。

## 11. 重要代码入口

| 主题 | 仓库内路径 |
|---|---|
| Proxy与输入特征、路由 | `vllm-ppd/ppd/comprehensive_proxy.py` |
| 原始查表与score | `vllm-ppd/ppd/optimizer/ppd_decision_engine.py` |
| Matched offline benchmark | `vllm-ppd/scripts/benchmark/comprehensive_benchmark.py` |
| 新状态参考函数 | `vllm-ppd/ppd/optimizer/state_semantics.py` |
| Pending账本 | `vllm-ppd/ppd/optimizer/observation_pending.py` |
| 节点观察器 | `vllm-ppd/vllm-source/vllm/v1/core/ppd_observer.py` |
| 只读HTTP接口 | `vllm-ppd/vllm-source/vllm/entrypoints/serve/ppd/api_router.py` |
| Scheduler | `vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py` |
| KV管理 | `vllm-ppd/vllm-source/vllm/v1/core/kv_cache_manager.py`、`block_pool.py` |
| EngineCore | `vllm-ppd/vllm-source/vllm/v1/engine/core.py` |
| NCCL connector | `vllm-ppd/vllm-source/vllm/distributed/kv_transfer/kv_connector/v1/p2p/p2p_nccl_connector.py` |
| 当前分析器 | `vllm-ppd/scripts/profile_analyze_v2.py`；旧 `profile_analyze.py` 也保留 |
| 服务配置与启动 | `vllm-ppd/scripts/server/config.sh`、`start_2P_2D.sh`、`start_2P_2pD.sh` |
| Agent循环/请求 | `uni-agent/uni_agent/agents/react/agent.py`、`model.py` |
| Editor、shell | `uni-agent/uni_agent/tools/edit_file.py`、`shell.py` |
| Modal | `uni-agent/uni_agent/sandbox/modal.py` |
| Task/评测 | `uni-agent/uni_agent/tasks/swe_bench/` |
| Agent配置与并行入口 | `uni-agent/examples/inference/task_config_ppd.yaml`、`parallel_infer_api.py` |

原始benchmark是生成calibration的工具。Live Agent执行Dynamic时不需要后台再跑一遍benchmark；proxy加载已经生成的结果进行决策。

## 12. Vast 迁移和当前环境

2026-10-27已在新实例完成单卡迁移验证。连接参数是当时记录，不保证未来租约重建后仍相同：

```text
ssh -p 40691 root@89.25.223.118
```

Mac连接需要它自己的SSH公钥已获授权；此前自动迁移用的是AutoDL侧密钥，不能假设Mac已能登录，也无需把那把私钥复制过去。

硬件实测：RTX PRO 6000 Blackwell Max-Q Workstation Edition，97887MiB显存，compute capability12.0；驱动590.48.01，Ubuntu22.04.5，CUDA Toolkit12.8.93。cgroup约52.66 CPU、457GiB内存；不能把宿主机总384 CPU/1.2TiB当作实例配额。

环境：Python3.10.14、torch2.9.1+cu128、Triton3.5.1、vLLM fork0.13.0、transformers4.57.6、FlashInfer0.6.14、Modal1.5.3。原生扩展按sm120构建，实际CUDA和模型测试通过。

已知依赖例外：vendored CUDA requirements声明flashinfer-python==0.5.3，保留原实际运行版本0.6.14，因此最终pip check并非全绿。迁移文档中“当前pip check通过”的早期段落应按构建前阶段理解；最终以installed-after-build与pip-check-after-build结果为准。不要为消除提示擅自改版本或篡改metadata。

OpenCV改用4.11.0.86、numba0.61.2，均有迁移记录。保留verl源码用于外部API工作流，但未注册要求numpy2/transformers5的RL训练发行包，避免与serving依赖冲突。

模型按用户要求从ModelScope重新下载，路径仍为 `/root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507`，16个分片、约56.87GiB权重。五个配置/tokenizer/权重索引SHA一致；未宣称完成源与目标全部权重逐文件SHA对照。

单卡smoke：BF16、155648窗口、显存利用率0.90、max_num_seqs4、max_num_batched_tokens2048、enforce_eager、prefix caching、Hermes工具、deepseek_r1 reasoning。正常文本返回4；工具调用emit_result({value:4})可解析。窗口通过容量/启动检查，但未实发155648-token prompt。

模型监听127.0.0.1:8000，API base为http://127.0.0.1:8000/v1。8080已被Jupyter使用。若要在Mac访问模型应单独转发8000，不要误把8080当模型API。

这些是单卡迁移验收，不是四卡PPD性能结果。服务此前启动成功；接手后先查当前进程和健康状况，不应把本文当实时状态。

新实例记录位于 `/root/ppd-migration/`：deployment.md、validation-status.json、installed-after-build.txt、pip-check-after-build.txt、single-gpu-server.log、chat-smoke.json、tool-smoke.json及迁移manifest等。原AutoDL工具与证据在 `/root/vast-direct-setup/new-instance-20261027/`，不属于三个主要Git仓库。

源数据集与目标的500条parquet校验一致，SHA256：`8f702d9ed2945f077a07a61c0c95e95e57b4cc5ed4dbb7755eb23e7bcd455efa`。实验logs/results已迁，PDF在新实例 `/root/ppd-migration/papers`。Modal认证未自动迁移，跑live Agent前需要确认。

新实例Codex0.162.1已由用户完成设备登录；一次交换凭证400后重新登录成功。无显式代理的实际对话6.68秒返回CODEX_NETWORK_OK。这个结论只针对新实例当时的网络，不证明Mac或AutoDL网络也相同。

Codex历史迁到Vast后可读取；随后为Mac做过隔离快照和导入器，7个主会话全部分页读取通过。这些备份保留作档案。用户最新选择是本地新项目配合本文接手，尚未验证Mac导入，不要再默认执行数据库迁移。

## 13. Mac 完整克隆与本地项目操作

这里推荐普通完整Git clone，不使用--depth、--filter或--single-branch。--branch只选择初始工作分支；它不会让普通clone变成浅克隆。默认其他分支保存为origin/*远端跟踪引用，不需要为每个分支都建立一个本地工作目录。

在Mac终端逐条执行，若目标目录已存在，应检查它，不要删掉重来：

```bash
mkdir -p ~/Projects/agentic-workspace
cd ~/Projects/agentic-workspace

git clone https://github.com/Stark-X6/agentic-ppd.git

git clone --branch fix/agent-tool-contract \
  https://github.com/Stark-X6/vllm-ppd.git

git clone --branch fix/agent-tool-contract --recurse-submodules \
  https://github.com/Stark-X6/uni-agent.git

git clone https://github.com/Hanchenli/vllm-continuum.git
```

若已按前面的建议clone了uni-agent，只需补：

```bash
git -C ~/Projects/agentic-workspace/uni-agent submodule update --init --recursive
```

不要加--remote；需要的是父仓库固定的子模块提交，不是各子模块最新分支。下载子模块源码不等于安装完整训练依赖。

克隆后核对：

```bash
cd ~/Projects/agentic-workspace
git -C agentic-ppd status --short --branch
git -C vllm-ppd status --short --branch
git -C uni-agent status --short --branch
git -C uni-agent submodule status --recursive
git -C vllm-ppd rev-parse --is-shallow-repository
git -C uni-agent rev-parse --is-shallow-repository
```

两个is-shallow结果应为false。不同时间远端可能前进，本文中的commit用于定位当前交接状态，不要求无条件回退新代码。若要复现实验，应明确checkout记录的commit，并保留新开发分支。

完整Git clone只涵盖远端仓库可达的版本历史和tracked内容；它不是服务器磁盘镜像。不会自动带上ignored数据、未提交改动、shell环境、conda、运行中的进程或Codex聊天数据库。一般开发不需要--mirror。子模块须另外递归初始化。

在VS Code使用File→Open Folder打开agentic-workspace父目录，四个仓库仍独立管理。在本地Codex项目中打开同一目录，并把本文作为接手材料。本文新生成后尚未提交/推送；应下载到Mac的agentic-ppd/docs/，再按本地Git流程管理。

本地编辑、SSH连接Vast、同步确定版本、启动/监控实验和取回结果的具体操作，见[Mac与Vast实验协作说明](./mac-vast-experiment-workflow.md)。该文档第11节提供可直接给本地AI的接手指令。

Mac负责编辑、读文档、轻量CPU测试和分析。不要在macOS上直接安装Linux CUDA版vLLM完整环境，也不要为了编辑代码启动GPU。某些测试仍有Linux/依赖要求，未在Mac验收的测试应明确标记。

## 14. 需要单独保存或迁移的数据

| 内容 | Git clone是否包含 | 建议 |
|---|---|---|
| PPD/Agent源码、tracked文档与测试 | 是 | 完整clone |
| uni-agent嵌套子模块 | 递归初始化后包含 | 使用父仓库固定版本 |
| Qwen模型权重 | 否 | 留在GPU端，必要时ModelScope重下 |
| 原始run logs、trace、metrics、calibration JSON | 通常否 | 单独备份；分析所需部分同步Mac |
| SWE-bench parquet | 不应假定包含 | 按manifest检查并单独迁移 |
| 论文PDF | 当前在数据盘 | 单独拷贝 |
| Python/CUDA已安装环境 | 否 | 保存依赖清单与构建脚本，在目标平台重建 |
| Codex聊天、认证、SSH私钥、Modal凭证 | 否 | 不入Git；按需要独立处理 |

论文原件包括PPD、Continuum、AMPD，以及prediction-based KV-cache management相关工作。源文件在AutoDL数据盘，新实例papers目录已有迁移副本。这里只延续此前已核对的研究结论，没有在本次重新审计每篇论文全部内容。

后续备份应保留有价值的run manifest、request profile、Agent trace、必要原始服务日志、metrics和校准结果，不能因已有summary就删掉所有原始证据。用户允许清理无用旧日志不等于允许删除唯一实验原件。

## 15. 下一步工作顺序与协作约定

不需要GPU即可继续：

1. 在Mac完成完整克隆，把本文与当前设计文档纳入本地项目。
2. 核对版本、主要代码入口和CPU依赖，确定哪些测试可本地运行。
3. 继续讨论新calibration的低成本设计，特别是prefill积压如何简化；冻结点数、分桶、配对条件和fallback之前不要自动启动大规模测试。
4. 整理可重复的部署/实验命令，明确代码和数据保存位置。新硬件性能校准计划单独制定。

需要GPU时先做小规模原生observer验收：冷/热前缀、取消/重置、decode分类、pending ID/ACK、IPC与off/on开销。单卡可做部分本地路径验证；P→D/NCCL与固定2:2实验需要相应多卡资源。完成后才进入新状态calibration、shadow和有限live实验。

TTL后置；严格serving因果比较仍需replay。模型、Agent policy、样本、map和资源比例变化应作为单独实验轴。

用户偏好：用中文沟通；解释变量与实际系统行为；先核对代码再下结论；可执行任务直接推进，避免反复确认已授权事项。长GPU实验用低频监控，用户经常让Claude Code负责监控；不要持续高频轮询消耗额度。用户明确要求停在某阶段时必须停下。

不得因为新实例有GPU就自动开完整benchmark；讨论设计不等于授权租新实例或开新付费服务。工作区原改动属于用户，提交保持单一意图。运行前检查实际状态，报告区分已实现、CPU验证、GPU验证和研究假设。

可给Mac本地Codex的首次指令：

> 这是延续中的Agentic PPD研究项目。请阅读agentic-ppd/docs/mac-local-project-handoff-2026-10-27.md，以它和当前handoff/阶段5设计为上下文，核对本地四仓库分支与提交。当前重点是完成本地接手并继续讨论状态条件化查表的offline校准简化方案；新状态语义与CPU观察器已实现，原生GPU/IPC验收、新calibration和TTL尚未完成。请区分最新选择与历史备选，不重复搭建已验收环境，不自动启动GPU实验。后续按我的具体任务推进。
