# 阶段 5：Agent-compatible Dynamic 的问题与备选方案

问题梳理基准日期：2026-09-19。当前方向说明更新：2026-10-21。状态：理论方案与源码核对；在线新策略尚未实现，本次未启动 GPU 实验。

当前第一版（2026-10-21）已选择保留 PPD 的实测 TTFT/TPOT 评分与现有权重，以 (N,U_P,U_D)、D 的 (B_D,K_D) 和两端 prefill 积压 (Q_tok,Q_att) 重新定义查表场景。在线仍输出 Local/Remote，表可同时保存两边指标与动作。AMPD 前置 TTFT/ITL 判断后置；TTL 仅留接口。F/W/X 精细预测、output predictor 和“TTFT 优先”均不是第一版前提。完整变量推导与校准边界见 [阶段 5 当前方案](./stage5-state-aware-routing-and-ttl-design.md)。本文其余候选池保留为研究过程，不代表当前同时实施。

本文按基础适配和拓展适配整理现有问题、备选解法及其前提。基础适配使 live Agent 的特征与 calibration 含义一致；拓展适配处理“相同输入特征，执行条件却不同”的情况。已完成工作见 [handoff](./agent-to-ppd-handoff.md)，运行记录、原始数据位置和 CPU 审计命令见 [experiment log](./agent-ppd-experiment-log.md)。下面的候选设计不等于已实施计划，也不代表已有性能收益。

## 一、当前判断与不变的实验边界

原始 Dynamic 已经跑通，但当前 calibration 不能给出第二种动作。在 `2P_2D`、`w_ttft=w_tpot=1`、`default_use_ppd=true` 下，72 个实际 matched pair 的 score 全为正；另 108 个缺失 pair 使用默认 PPD，合计 180 个 lookup cell 全部选 D-local。Huge-context 分支继承 large 的符号或默认值，关闭 short bypass 后的 4,320 组 CPU 输入扫描也全部选 PPD。这是既有审计结果，详见实验记录第八章。

因此，修正 output/QPS/context 可以改变选中的 cell，却不能在这张表上改变动作。新的有效路由需要先证明：在当前数据面和资源比例下，存在可重复的 PD 获益条件。找不到这种条件时，保留 x=1 是可接受的研究结论；P 空闲本身不构成策略失败。

最初的适配保持以下边界：

- 固定模型、BF16、Agent policy、reasoning 回放、tool schema、生成预算、样本与 routing map。
- 保持 2P:2D；正常多轮请求仍在同一 D decode，首先连配对 P 也保持不变。
- 动作只有“固定 D 做缺失 prompt 的 prefill”与“固定 P prefill 后传回固定 D”。
- 暂不混入 P:D sizing、D migration、admission、offload、双向 KV 传输或队列重排。
- 原始 Dynamic 保留为复现 baseline；新特征、新校准、新目标或新状态规则分别命名，不回改原始结果。

代码依据以复现版本 PPD `11dccb5d64d9`、uni-agent `449f80f4383b`、Continuum `316a58794a6f` 为锚点。论文依据为上传的 PPD、AMPD、Continuum、PBKV PDF；下文只保留支持具体方案的引用，不再设置 prior-art 阅读路径。

## 二、先统一变量、动作与信息时刻

### 2.1 Token 数、缓存数与工作量

| 符号/字段 | 含义 | 当前可知程度 |
|---|---|---|
| N / prompt_tokens | 本轮完整序列化 prompt，含 system、tools schema、历史 reasoning/tool calls/observations 及本轮输入 | response 有真实计数；决策前完整模板计数仍需对齐 |
| H / context_tokens | 本轮逻辑历史，而非节点上驻留的 KV 数量 | proxy 历史特征仍含字符近似 |
| A / append_tokens | 相对上一轮生成后的逻辑新输入，通常为 tool observations 或新 user message | 当前按最新 suffix 的 tokenizer/template 计数 |
| Dynamic 的 input_tokens | 传给决策器的 append 特征 A，不是整轮 N | 不可与 API usage 中的总输入长度混称 |
| G / completion_tokens | 本轮实际生成 tokens，包含 thinking、工具调用序列化及正文，不能只数 assistant.content | 完成后可知；路由前只能预测 |
| G_max | 请求允许生成的上限，即 max_tokens/max_completion_tokens | 请求到达时可知，不等于 G 的预测 |
| C_D、C_P | 候选节点此刻匹配且可复用的前缀 tokens | 当前缺可靠的决策前逐请求接口 |
| U_D、U_P | 各候选节点需要计算的 prompt 工作量 | 可从 N−C_i 起步，但需计入 block 对齐、末 token 重算与实际调度语义 |
| X_actual | 当前 connector 的实际 KV 传输成本 | HTTP 阶段有记录，纯传输 bytes/time 尚不完整 |

本轮 H 通常已经包含上一轮 assistant 的 reasoning、正文和 tool calls；新工具返回才进入 A。因此，两次完整 prompt 的长度差通常还含上一轮 G，不能直接当成 A。单独对 suffix 使用 chat template 也可能额外计入模板边界。只有按一致的完整 token 序列划分，才能使用 N=H+A。判断前缀能否复用要看 token-LCP 和 block hash；不能由“上一轮 prompt+completion”直接保证完全命中。

“只算 observation 的 KV”成立的条件是：候选节点已有可复用历史，且前缀一致。即使如此，新 tokens 的 attention 仍要读取长历史 KV，prefill 时间不只取决于 A。D cache 丢失或 P 长期未参与时，U_i 可以远大于 A。

### 2.2 可观测信息何时能用于决策

| 时刻 | 可用信息 | 决策/更新 | 不可倒用的事后信息 |
|---|---|---|---|
| 本轮请求到达 | 完整 messages、过去 completion/gap、当时节点快照 | prefill 路由 | 本轮 G、最终 queue time、response cached_tokens、运行期间 load max |
| 本轮生成完成 | 实际 G、finish reason、已生成 tool calls | TTL/返回间隔预测；更新 output 模型 | 尚未执行工具的真实耗时、episode 最终步数 |
| 下一轮到达 | 真实返回 gap、新 observations | 更新 gap 模型、重新路由 | 不得回填成上一次决策已经知道的数据 |
| episode 结束 | submit/context_limit/max_steps/error 等状态 | 结束保护并清理引用 | 不能依靠“出现 tool call”无限期假定还会回来 |

ORCA response 指标通常晚于本次路由；若用最近一次结果作候选输入，必须记录 sample age。事后 profiling 字段丰富，不等于已具备在线状态接口。

## 三、基础适配问题

### B1. Output feature 混用了“生成上限”与“预计生成长度”

**现状。** Live proxy 使用请求中的 `max_tokens`/`max_completion_tokens`，较大的安全上限会把多数短回复分入长输出 workload。ShareGPT benchmark 的做法也不构成线上预测器：它对已有记录的 `gpt_text` 计数，再将该数设为生成上限并使用 `ignore_eos=True` 控制工作量。它知道的是数据集中参考回答的长度，不是事先知道新回答将生成多少 tokens。[记录提取](/root/vllm-ppd/scripts/benchmark/sharegpt_benchmark.py:244)、[请求构造](/root/vllm-ppd/scripts/benchmark/sharegpt_benchmark.py:398)。

**备选方案。**

1. 保留 G_max，另设 `predicted_output_tokens`。先比较上一次 G、近期 rolling median、EWMA 等低成本估计；按模型/生成设置及可见任务阶段分组，冷启动回退全局历史。P90 表示保守分位数，不能当成期望长度，median 也不等于均值。
2. 若粗估计确实影响动作，再考虑回归或长度分桶预测，可用当前 prompt/append、过去生成和已知工具返回类型等特征。不能使用本轮尚未生成的工具调用或未来 step 信息；按 episode/时间切分，防止同一轨迹泄漏。
3. 借鉴 AMPD，改用 queue/prefill/传输成本和 decode 延迟保护，在线路由不显式依赖本轮最终 G。这会替换原始 workload 查表策略，而不是直接从原表删除 output 维。

预测服务于路由，不应同步缩小真实 `max_tokens`，否则改变了 Agent 行为。`finish_reason=length` 是截断观测，不能作为自然结束长度标签；失败生成、retry 和成功返回也要分开。

评价重点是分类/动作变化、预测误差引起的错路由成本与校准覆盖，不能只看 token MAE。Actual-G oracle 只可做事后敏感性分析；当前全正旧表上，即使用 oracle 也改变不了动作，因此暂不优先训练复杂模型。

### B2. QPS 的到达对象、时间窗口与 concurrency 不一致

**现状。** `record_request_arrival` 为原策略保留“此前 logical requests / proxy uptime，分母至少 1 秒”；已实现的 1/5/10/60 秒滑动窗口只用于观测。[实现](/root/vllm-ppd/ppd/comprehensive_proxy.py:775)。

原 offline benchmark 按 conversations/s 生成到达，每个 conversation 固定两轮，第一轮结束立即进入第二轮，没有工具 gap；live 则每个 step 产生一个 logical LLM request。即便每个 offline conversation 有两轮，有限窗口中的 requests/s 也不能机械除以二变回 conversations/s。[负载生成](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:578)、[两轮调用](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:539)。

需要同时区分：

- episode concurrency：仍在进行的 episode 数，可能在等工具；
- in-flight requests：已提交、尚未结束的 LLM 请求数；
- engine running/waiting：节点实际调度或等待的请求；
- arrival rate：单位时间内进入系统的新请求数。

在近似稳态的 closed-loop 中，可用 λ≈C/(LLM 时间+工具/客户端 gap) 理解关系。LLM 堵塞会让下一轮更晚到达，arrival QPS 可能下降而拥堵加重。Little 定律也需要一致的系统边界和稳态条件；有限样本的启动/排空阶段不能据此精确反推负载。

**备选方案。**

1. 建立统一窗口字段：到达对象、logical/physical、全局/每节点、时间戳、窗口宽度、分母、样本数和指标年龄。Logical retry 不重复算新 turn，但 physical attempts 必须反映真实服务器工作量；窗口初期分母与空窗口行为要明确。
2. 继续使用 lookup 时，先统一 offline/online 到达单位，再重新提取或采集相应 calibration。修成 window request QPS 后直接喂给旧 conversation-QPS cell，只修好了时效性，没有修好含义。
3. 状态感知方案将 arrival rate 作为辅助信号，主要看队列工作量、decode 干扰与 KV 状态；不能把 waiting=3 或 concurrency=16 直接填进 QPS cell。

**共同使用滑动窗口，不代表共同使用窗口 QPS。** AMPD 使用每节点近期 TTFT/ITL 统计，论文默认 10 秒；Continuum 的估计包含历史回访排队等待，工具返回分布又是另一类历史样本。可以共享采集框架，但分别定义指标。窗口平滑与瞬时队列互补；无完成样本不表示零负载，长窗口会滞后，短窗口则容易受突发影响。[AMPD §3–4](https://arxiv.org/abs/2602.14516)、[Continuum §4–5](https://arxiv.org/abs/2511.02230)。

### B3. Short-append bypass 把逻辑小输入等同于低本地成本

**现状。** `append<512` 直接返回 D-local，早于 lookup。当前 append 已包含工具返回内容，但小 A 不能证明 C_D≈H：D miss 后可能要重算长历史，即使 D 命中也可能因已有 decode 而产生明显 ITL 干扰。[在线分支](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:428)。

**备选方案。**

- 保留 512 作原始 baseline；关闭或降低阈值作为启发式敏感性对照。128/256 不是已验证最佳值。
- 以实际 U_D 或预测本地成本判定是否可安全 bypass；对长历史 attention、D queue/ITL 和状态不确定性增加保护。
- 若已有足够便宜的二路径成本比较，取消独立 short shortcut，避免在比较之前就屏蔽候选 PD。

后两项依赖 E1/E2。B3 与 lookup 是串联规则：表全正时关 bypass 仍全部 local；bypass 未解除时也可能看不到表变化。消融应先确认动作可区分，再讨论收益。

### B4. 短 context calibration 无法代表长 Agent 历史

**现状。** 原校准历史锚点为首轮 `128+128=256` 与 `1024+1024=2048` tokens；4096 是分类边界，不是已实测的最大历史锚点。Huge-context 继承 large 的 score 符号或默认值，没有建模长度继续增加后的成本。[分类](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:70)、[huge 分支](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:398)。

长 H 有两种不同后果：D 命中时仍有长 attention 成本；P 缺历史时则可能有大段重算。不能仅凭“context 更长”断言一定偏向 P 或一定退化为 x=1。

**备选方案。**

1. 从真实 Agent 的 H/A/G 分布选择代表性长 context 点，做 matched calibration；明确每个 cell 的支持范围、样本量和 cache 初态。只改 `small/large/huge` 阈值不构成校准。
2. 用连续或分段的 prefill 模型表达 F(C_i,U_i;节点状态)，减少高维分类表的稀疏问题。代价是要验证预测误差、外推和背景 decode 下的准确性。
3. 分层方案：旧表只处理原支持域，长 context 或状态异常走显式 fallback/新模型，并记录来源。

全 prompt 的模板、tools schema、reasoning 回放和 token-LCP 先保持一致，再讨论长 context。两轮、无工具间隔的 offline workload 也不能复制多轮后 P stale、D cache 竞争等状态；原表仍保留用于论文流程复现，新增 Agent-derived calibration 另存版本，不覆盖旧数据。

### B5. 全 PPD lookup 的动作可达性与优化目标

等权 score 为：

```text
A_T = (TTFT_PD − TTFT_local) / TTFT_PD
B_T = (TPOT_local − TPOT_PD) / TPOT_PD
S = A_T − B_T
有效数据、没有提前 bypass 时：S > 0 → local；S ≤ 0 → PD。
```

因此，只要 local 的相对 TPOT 代价 B_T 不小于其相对 TTFT 收益 A_T，就会选 PD；若 PD 连 TTFT 也更快，更容易满足条件。当前最小 S≈0.004094，已接近边界，但尚缺重复测量来判断边界是否稳定。72 个正 score 是测得条件下的结论，不是 x=1 的普适最优证明。[score 与失败处理](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:280)。

`success_rate` 在 comprehensive benchmark 中按汇总到的 turn 请求结果计算：successful_reqs/total_reqs×100，不是 conversation 修题成功率或 SWE-bench 准确率；loader 用 PD 的零成功或全零指标分支判定 PD 不可用。此次 72 个 pair 没有经该分支强制 PPD，108 个缺失 cell 则确实使用默认动作，不能把 180 个 cell 全当成实测证据。[统计代码](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:627)。

**QPS/P:D 的推论。** 更高 QPS 可能让 D-local append 更干扰 decode，也可能首先压垮 P/传输、使 local 更有利，所以不单调。更多 P 可以减少 P 排队，但固定总 GPU 数时又减少 D 的 decode/KV 容量；比例改变后原校准也不再适用。我们要研究固定 2:2 的路由，不把资源扩容当成路由收益。

**备选方案。** 先重复验证接近零的条件，并覆盖真实长历史与 P/D warm/miss 状态；确认存在 PD 收益后，再选择补 lookup 或换成本模型。保持 `w_ttft=w_tpot=1` 的原始 baseline，不用修改权重/默认值人为生成 PD。未测高 QPS 明确标 fallback，不能为消除全 PPD 而把缺失项改成 PD。

该 score 是相对 TTFT/TPOT 的加权和，不是直接估算请求总延迟、最大化吞吐或保证 SLO。对于非流式 Agent，工具要等完整 response，近似有：

```text
L_tool_ready ≈ TTFT + (G−1) × mean_TPOT，G≥1
```

这说明 G 对完整返回时间有价值，却也说明等权百分比 score 不等于节省多少秒。该近似不含 RPC 等额外时间，也没有计算本次 append 对其他请求的干扰。若改为 tool-ready 或 SLO-aware 目标，应作为新策略明示，不继续称为原公式复现。

### B6. 路由分配与 live 可复现性的边界

原始会话 hash 和注册顺序可能使跨 run 的节点分配变化；有限样本 hash 也不保证恰好均分。当前稳定 SHA-256/排序节点解决前者，显式 first16 map 控制 8/8 分配。稳定 hash 与强制均分是两件事，后者是配对实验控制，不是 PPD 正确性要求。

正常会话保持 P/D affinity，Dynamic 选择去 P prefill 或在已绑定 D prefill，并非每轮换一台 D decode。AMPD 也绑定 D，但可在多个 P 候选中选 remote；照搬其多 P 选择会改变我们当前固定配对的实验条件。[server selection](/root/vllm-ppd/ppd/comprehensive_proxy.py:814)。

备选研究若包括负载感知初始 placement、admission 或故障迁移，应另开对照轴。当前先固定 map，报告每节点实际 requests/tokens/queue/KV；8/8 episodes 不意味着瞬时均衡。Temperature=0 的 live 轨迹仍会分叉，稳定 map 不能替代 workload replay。

## 四、拓展适配问题

### E1. 路由前能否知道可复用 KV：可以增加查询，但快照不等于保证

**当前能力。** vLLM 在请求已进入节点后的调度阶段，根据 block hashes 找最长可复用前缀。`get_computed_blocks` 返回完整缓存块，并将最大命中限制为 prompt length−1 以重算末 token logits，必要时会影响整个末尾 block；真正分配时才 touch 引用以防驱逐。Proxy 当前没有等价的决策前接口。[查前缀](/root/vllm-ppd/vllm-source/vllm/v1/core/kv_cache_manager.py:164)、[分配与引用](/root/vllm-ppd/vllm-source/vllm/v1/core/kv_cache_manager.py:287)。

**备选方案。**

1. 节点提供只读 prefix query，输入与该节点一致的 token IDs/block hashes 及模型/cache namespace，返回 C_i、block 数、epoch、时间戳和版本。需独立实现无副作用查询，不能直接把会更新统计或影响生命周期的方法当只读 RPC。
2. 节点将 cache 插入/驱逐事件发布给目录，proxy 本地查询。它减少每轮 RPC，但存在事件滞后、共享前缀身份及目录大小问题，需要 freshness 和一致性检查。
3. 只根据历史命中、上次节点、KV 压力估计 residency，工程量较小；它只能给置信区间或保守 fallback，不能宣称准确知道可复用数。
4. 为选中的路径增加短租约/reservation，并在执行前复核。它比裸快照更可靠，却会占用显存并改变系统行为，必须限制总量、处理超时/取消；不宜为所有候选长期 pin。

实际 prefill 前可得到某一时刻的准确匹配数量，但从查询到执行之间其他请求仍可驱逐 cache。先做快照和执行时复核，还是引入 reservation，是准确性、开销与调度干扰之间的选择。

### E2. 同样 QPS/长度下，队列和 decode 干扰不同

原特征没有区分：D 空闲或已有多个长 decode，P 空闲或积压大 prefill，等待请求短或长，以及可分配 KV 是否充足。仅增加 conversation 数也无法表达这些状态。

**备选输入。** 每个候选节点记录 running/waiting、队列中的预计 token 工作量、当前任务剩余工作估计、decode batch/context、free/pinned KV、近期 TTFT/真实逐迭代 ITL 及统计年龄。Queue time 近似为前置工作量而非 waiting count 乘常数；chunked prefill、batching 和优先级会改变这一近似，必须针对当前 vLLM 调度校准。

AMPD 的原 Algorithm 1 顺序是：有 P 的窗口 TTFT 留有 SLO 余量则 remote；否则 D 的窗口 ITL 有余量则 local；都紧张时才比较 queue/prefill/传输成本。它不是每次都直接取成本最小，也不是仅使用窗口 QPS。[AMPD §4.1](https://arxiv.org/abs/2602.14516)。

可借鉴其窗口与成本分解，也可只加 decode 干扰保护；直接照搬 P-slack 优先会遗漏“P 最近很空，但当前请求需要重算很长历史”的风险。当前 request 平均 TPOT 不能替代迭代 ITL 尖峰，若缺该接口，方案与报告都应明确这一限制。

### E3. 单向 KV 数据面决定 remote 成本，不能直接搬 AMPD 公式

当前实现中 P 为 producer、D 为 consumer，PD 分支发送完整 request 给 P，再将 KV 送到绑定的 D；没有从 D 拉取历史 KV 到 P 的阶段。这是当前协议/connector 实现边界，不是 NCCL 物理上只能单向通信。[proxy PD 路径](/root/vllm-ppd/ppd/comprehensive_proxy.py:1328)、[connector](/root/vllm-ppd/vllm-source/vllm/distributed/kv_transfer/kv_connector/v1/p2p/p2p_nccl_connector.py:304)。

P 仍可能通过自己的 prefix cache 复用历史，不能一律称为全量重算。但如果十轮都在 D-local，P 即使保留首轮前缀，也缺少后续 reasoning、工具调用及 observations；这一轮 A=1K 不代表回 P 只算 1K。D miss 时 local 同样可能计算大量历史。

保留单向实现时，可将候选路径的首 token 墙钟成本写成：

```text
T_local_first ≈ Q_D + F_D(C_D, N−C_D; state_D) + R_local
T_PD_first    ≈ Q_P + F_P(C_P, N−C_P; state_P)
                 + X_actual(P→D) + R_D
```

Q 表示到可执行前的等待；F 表示给定前缀和未命中量下的 prefill；R 明确包含尚未计入的调度/首 token 阶段。实际打点要统一终点，不能一条算到 prefill 结束、一条算到首 token。计算、传输与调度可能重叠，HTTP elapsed 已覆盖的部分不能重复相加。

AMPD 的 remote 路径支持 D→P 历史 KV 与 P→D 增量 KV，其成本式基于这一能力。[AMPD §3–4](https://arxiv.org/abs/2602.14516)。我们的备选是：

- **保留单向传输：** 测量各自 C_i/U_i、真实发送 blocks/bytes、队列与干扰，建立本机成本模型。实现范围较小，但 P stale 时可能合理地长期 local。
- **新增双向传输：** 以历史搬运替代 P 重算，可能扩大 remote 获益区间；需要历史/增量合并、block 布局、临时 P 容量、pin 与并发安全。历史+增量的双向字节数不必然大于单向完整 prompt KV，不能仅按方向数判断开销。
- **增强 P 侧历史保留或同步：** 减少未来 P miss，但增加内存、带宽和预测成本，且未必会再次使用这些 KV。先不与路由一同引入。

无论哪条路径，最终仍在同一 D 保存 KV 并 decode。因此 remote prefill 可以减少 D 计算干扰，却不会自动解决 D 容量不足；admission、迁移、offload 属于另一问题。

### E4. 决策目标、状态过时和不确定性如何约束新策略

以下三个策略家族是早期候选池。2026-10-21 已收敛为保留原 score 的实际 KV/负载状态条件化查表；不是仅适配旧 output/QPS，也不要求精细组件模型或 AMPD 前置门控。下表保留用于解释其他备选的取舍。

| 候选 | 保留什么 | 新增什么 | 主要限制 |
|---|---|---|---|
| Agent-adapted lookup | 原等权 score 与离散表 | 同语义 output/rate、Agent 长度校准、显式 coverage | 状态被平均掉；表仍可能全部 local |
| Lookup + state guard | 支持域内的原表 | residency/queue/ITL 条件与异常 fallback | 必须说明状态何时覆盖原动作，避免不可解释叠加阈值 |
| AMPD-inspired 单向成本路由 | 固定 P/D、当前 connector | C_i、队列工作量、F/X 模型与 decode/SLO 保护 | 依赖新鲜状态和本机校准，不是 AMPD 原样复现 |

当前最终输出 G 难预测，不妨先针对 prefill/TTFT 与 decode 干扰决策，再把 G 作为 full-response 目标的可选扩展。另一方面，仅最小化当前请求 TTFT 也可能拖慢其他正在 decode 的请求；保护条件和最终评价必须覆盖这种外部影响。

对三类候选都需明确：支持域、snapshot age、估计误差、缺数据行为、fallback 原因以及决策开销。若两路径差异小于模型误差，可保持原动作或回退 local；这只是保守候选规则，不证明 local 在 D 过载时安全。容量硬约束要优先报告并交给明确的安全调度机制，不能伪装成性能偏好。

### E5. Cache loss 与 TTL：收益机会、文献边界和负收益条件

**先修正比较前提。** 所读 PPD 版本没有直接测“Continuum TTL 接入 PD 后退化”；其中 session expiry 也不能等同于 GPU block TTL。AMPD §7.2 的 vLLM-Continuum 是 co-located baseline：回访优先等机制改善排队/TTFT，但 ITL 未相应改善，因而未必提高联合 SLO attainment。该实验不能证明 TTL 在 PD 上必然失效，也不与我们观察到前缀未命中矛盾。[PPD](https://arxiv.org/abs/2603.13358)、[AMPD](https://arxiv.org/abs/2602.14516)。

**当前证据。** Retention 下降只说明下一轮没命中预计可复用前缀，可能来自容量驱逐、模板/前缀变化、block 对齐等；运行中 preemption 与工具间隔中空闲 cache 被替换也应分开。当前没有显式 KV offload 层，不能把 miss 解释为搬到 CPU。需要 token-LCP、实际 block hash 移除、后续 miss/重算与 pin/unpin 事件建立证据链。

vLLM 本来就可能自然保留已释放引用的 prefix cache，直到空间被再分配。TTL 的增量作用是保护本来会被挤掉的 cache，而不是从“完成就立即删除”变成“首次保留”。Expiry 解除保护，不等于立即删除 cache。[free_blocks](/root/vllm-ppd/vllm-source/vllm/v1/core/block_pool.py:384)、[实际移除 hash](/root/vllm-ppd/vllm-source/vllm/v1/core/block_pool.py:326)。

**为什么 PPD+TTL 可能有益。** 若工具很快返回、D 历史本来会被竞争请求驱逐，保护它可减少下一轮 local 重算及其 decode 干扰。纯 PD 仍可能因 P 缺历史而重算，单独保 D 不一定消除那部分成本；PPD 提供直接利用 D 历史的路径，TTL 提高该路径下次仍便宜的概率。这是机制上的互补假设，收益仍取决于自然命中率、返回间隔和显存机会成本。

**为什么也可能无效或更慢。**

- 自然 cache 已能保留，pin 没增加命中，却限制分配自由。
- 工具/Modal 间隔很长，保护占用过久，挤掉更多其他 episode 的有价值前缀。
- 系统主要受长 decode 限制，减少少量 prefill 不改善端到端时间。
- 返回后仍走不受该 D cache 保护获益的路径，或 TTL 在返回前就到期。
- Pin 导致 waiting、preemption、ITL 尖峰或优先级饥饿，损失超过自身省下的重算。

不能以“很多 cache loss”直接推出“TTL 有益”，更不能直接推出“组合优于 co-located Continuum”。

### E6. TTL 的备选移植方案与路由耦合

Continuum 在 response 完成后，根据返回间隔分布与缓存重建/回访等待收益选择保护时间。用其模型的简化形式表示：

```text
τ* = argmax_{τ≥0} F_gap(τ) × (R + η × T̄) − τ
```

这里 F_gap 是返回时间经验 CDF，R 为 cache 重建成本，T̄ 为论文估计的回访排队等待，η 体现 program 进度/剩余工作关系；不是将 TTL 设置为平均工具时间。其机会成本近似有适用条件，不能称为任意 GPU 负载下的全局最优。[Continuum §4](https://arxiv.org/abs/2511.02230)。

公开 [vllm-continuum README](/root/vllm-continuum/README.md:3) 明确不包含论文完整 estimation。`estimate_with_func.py` 用 markdown bash block/命令首词、历史均值和固定 2 秒阈值，不等于经验 CDF 的 argmax，也不直接适配我们的 structured editor tools。[parser 与估计](/root/vllm-continuum/vllm/v1/core/estimate_with_func.py:64)、[pin 规则](/root/vllm-continuum/vllm/v1/core/estimate_with_func.py:150)。

可分层验证：

1. **固定 TTL baseline。** 先接正确 pin/unpin 生命周期，设有界保护作机制对照；不冒称论文完整复现。
2. **Continuum-style utility。** 用已完成 gap 的经验分布、冷启动回退、重建与等待估计实现论文式选择；把 TTL 与回访优先级拆开测，避免把优先级收益全记给 cache。
3. **Route-aware TTL。** 候选价值定义为“缺 cache 时最优可行路径成本”减“有 cache 时最优可行路径成本”，例如 B_cache(s)=min_a T(a|miss)−min_a T(a|hit)。再考虑保护真正避免自然驱逐的概率和对其他请求的显存机会成本。若 T 已含排队，不能重复加一遍 ηT̄；这是待验证扩展，不是 Continuum 原公式或最优性定理。

Gap 应定义为下一请求 server arrival 减本轮 server finish，包含传回、客户端处理、顺序工具调用、Modal/RPC 和 observation 传输。Modal 时间可以拆分解释，却不能从“等待 cache 复用多久”的目标中删除。多个工具在本 Agent 内顺序执行，预测应对应总返回间隔；复合 shell 命令不能只按第一个词归类。

实现底线包括：logical retry 不重复创建生命周期、program-end 清理、共享 block 引用正确、返回并入队后的保护转移、传输中 block 不被复用，以及 pin budget/高压释放。TTL 到期、unpin、实际 eviction、offload、restore、recompute 分开记录。跨 vLLM 版本应移植必要机制，不能整体覆盖 scheduler。

PBKV 可补充 workflow 生命周期与共享 ownership 的设计：确定 retired 的 cache 与仅预测低价值的 cache 分开，保守预取不挤占活跃 cache。但多个独立 SWE-bench episode 不等于一个多 agent 分支图，暂不移植图预测器或 host prefetch。[PBKV](https://arxiv.org/abs/2605.06472)。

## 五、方案选择、验证顺序与报告边界

### 5.1 先完成基础语义，再验证决策机会

以下保留早期依赖框架，并同步当前查表选择；不表示现在启动实验。输入定义与是否后置的边界以当前方案文档为准。

| 子阶段 | 内容 | 判断标准 |
|---|---|---|
| 5A CPU/shadow | 全表动作可达性；N/H/A/C/U、B_D/K_D、Q_tok/Q_att；状态时间戳与 in-flight 语义 | 全 PPD 旧表可达性已查明；新字段接口尚未完成，不能宣称路由收益 |
| 5B 小规模机会验证 | 接近边界及真实 Agent 长历史/append，控制 P/D cache 与背景负载 | 找到超出噪声的 PD 获益条件；找不到允许保留 x=1 |
| 5C 状态查表 shadow | 已选保留原 score/权重的新状态表，验证匹配、支持范围、fallback 和开销 | 能解释何时选 PD、何时错选；不以混合比例本身验收 |
| 5D 有限 live | 仅启用有辨识度的改变，必要时检查 bypass×table 的交互 | 验证协议、状态新鲜度、失败/重试、质量和节点压力 |
| 5E 受控 serving 对照 | 相同工作量的 replay，保留 Static/原始 Dynamic/新策略 | 比较延迟、goodput、真实重算和不确定性，而非单次 wall time |

机会验证覆盖“D warm/miss × P warm/stale”的有意义状态及不同 decode 干扰，无需立即重跑整个原始 360 点或做全笛卡尔积。两条路径须重建相同 cache 初态，避免先跑的路径为后跑预热；生成工作量、背景 arrivals 和测量边界一致。代表性点由 trace 与边界证据选择，重复与运行顺序用于判断噪声，不能只凭一次 score 符号下结论。

如果扩大上下文后仍无 PD 收益，应先解释单向路径成本和 decode/容量瓶颈，不立即增大 P:D 或修改权重。负结果可以说明 Static x=1 是这一范围内的强基线。

### 5.2 Live 与 replay 各回答什么

Live 保留真实工具、Modal 和 Agent 质量，用于本阶段 profiling 很合适；temperature=0 仍不能固定 trajectory。Config-matched 结果可以描述现象，不能把不同输出/step 数的 wall time 差直接称为 serving speedup。

严格 replay 至少明确：request body/token 序列、completion 工作量、physical retry，以及同 episode 的依赖和工具 gap。等本轮结束再等待录制 gap 是 closed-loop；固定绝对 arrival 则是 open-loop 压力测试。只回放 body 而让生成长度任意变化，也没有固定全部服务工作量。固定长度且 ignore_eos 的测试标作 synthetic serving probe，不作为 live Agent 质量结果。

报告按节点展示动作与原因、校准/fallback、queue/ITL、TTFT/TPOT/full response、缓存命中/重算、失败重试和 episode 质量。c=12/14/16 只代表当前基线轨迹的 healthy/knee/stress 工作点，更换模型或负载后不能直接沿用结论。

### 5.3 TTL 单独验证后再组合，保留创新判断的边界

当前 TTL 只留接口，以下组合计划均后置。将来 Static x=1+TTL 可作为独立验证，Dynamic+TTL 要在两部分单独验收后再测。组合的核心问题是“保护 cache 如何改变下一轮两条路径的相对成本，以及为此让其他请求付出了什么”，不是把工具时间作为又一个无解释的分类维度。

相邻工作已经覆盖多项可借鉴机制：PPD 自身讨论 offline 分布漂移与在线状态互补；AMPD 提供队列/延迟感知路由；[Dynamo conditional disaggregation 提案](https://github.com/ai-dynamo/dynamo/issues/11514) 讨论未命中量、负载与 D-local 后 P 历史落后；[ThunderAgent](https://github.com/ThunderAgent-org/ThunderAgent) 的 admission/pause 处理 program 容量；[ConServe](https://arxiv.org/html/2606.01839v1) 可作为固定 D、多轮 local 的相关设计，但其模型、资源配置及 best-effort AMPD 对照不能直接外推到本机。这里只将它们作为机制边界，不将工程提案当作普适性能结论。

因此，“PPD 加实时队列”或“加一个 TTL 常数”本身不能直接宣称新颖。值得验证的方向是：在真实 Agent 返回间隔、可查询 residency 和明确数据面约束下，量化并控制路由与 cache 保护的相互影响。现有阅读尚不足以证明无人做过类似组合；最终贡献需要同负载、可归因的结果支持。
