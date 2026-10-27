# 阶段 5：KV 与负载感知的 PPD 查表方案及 TTL 接口

初次记录：2026-10-10。最近整理：2026-10-27。

研究顺序更新（2026-10-27）：先执行 [假设验证与改进空间实验设计](./stage5-hypothesis-validation-experiment-design.md)。本文保留变量推导与此前候选方案，不把高维查表、窗口积压修正或 TTL 视为已证明必要的组件；是否实现以及采用哪些输入，由小规模配对实验、真实状态频率和观测开销决定。字段语义与 baseline 不变，尚未开展本轮 GPU 实验。

状态：第一步语义冻结已完成；[规范契约](/root/vllm-ppd/docs/stage5_state_semantics_contract.md) 是字段边界与统计口径的准据。第二步已实现默认关闭的节点观测接口、就绪记录和独立 pending 账本并通过 CPU 校验，原生 GPU/IPC 验收及 proxy 自动身份映射仍待完成，详见 [只读观测实现](/root/vllm-ppd/docs/stage5_readonly_observer.md)。新查表与校准未实施，冻结配置不变，未启动 GPU 实验。

关联文档：[阶段 5 早期问题与备选方案](./stage5-agent-compatible-dynamic-theory.md)、[项目 handoff](./agent-to-ppd-handoff.md)、[实验记录](./agent-ppd-experiment-log.md)。

## 1. 当前确定的第一版范围

截至 2026-10-21，第一版明确为：**保留 PPD 用离线实测 TTFT/TPOT 选择 Local/Remote 的方法，改用实际 KV 前缀和当前节点工作量匹配测试场景。** 不把精确预测 decode 干扰、分解所有阶段耗时或引入在线反馈控制器作为前提。

| 项目 | 第一版处理 |
|---|---|
| 路由动作 | 固定 D 做缺失 prompt 的 prefill；或固定 P prefill 后经现有 P→D 连接器回到同一 D decode |
| 请求工作量 | 完整 prompt N、两端有效前缀 C_P/C_D，派生 U_P/U_D；不依赖未来 output 长度 |
| Decode 背景 | 以可参与 decode 的序列数 B_D 和上下文总和 K_D 作轻量描述 |
| Prefill 积压 | P/D 分别记录尚未完成的 token 工作与 attention 工作，归为少量状态类别 |
| 决策规则 | 保留 PPD 的 TTFT/TPOT 相对变化评分与现有权重；先隔离输入改进的效果 |
| AMPD 前置路由 | 后续备选；第一版不启用 P-TTFT/D-ITL 门控或 normal/protect 模式 |
| TTL | 仅预留状态与生命周期接口，不执行 pin/unpin，不改变缓存回收策略 |

### 1.1 Local/Remote 的收益和代价

Local 利用 D 上已有的有效前缀，绕过 P 排队和 P→D 传输；代价是在 D 上增加 prefill，可能拖慢已有 decode。Remote 将主要 prefill 计算移到 P，但 P 可能缺少 D 已有的历史，需要额外重算，并支付排队和传输成本。两条路径最终都在 D 保存上下文并 decode，Remote 不能自动解决 D 容量不足。

Append prefill 的收益取决于实际缓存：逻辑 observation 很短，不保证 D 只需计算这段 observation。P 也可能命中自身前缀，因此不能一律称为全量重算。当前 PPD 没有 D→P 历史读取，P→D 也不保证 append-only 传输，详见第七章。

### 1.2 评分规则与最终目标分开

此前“正常时优先降低本请求 TTFT”的建议没有被共同确认为最终目标，现不作为第一版原则。非流式 Agent 通常要等待完整 tool call，TTFT 下降而生成变慢，任务仍可能更晚完成。

目前确定的是改进 Agent serving 效率；固定资源与任务质量下，降低 episode 完成时间是候选主目标，尚未冻结唯一标量指标和约束阈值。后续评价仍需同时检查完整响应/tool-ready 时间、累计 LLM 等待及超时，分离 Modal 工具抖动。第一版固定原评分和权重是为了控制变量，不等于证明该 score 已经等价于最小化 episode 时间或保证 SLO。

### 1.3 不变的边界

这里的准确研究约束是“不依赖输出长度预测”，并非已经证明输出长度在任何场景下都不可预测。早期文档中的 output predictor 仍属于历史备选方案，不再作为本次方向的必要模块。

当前宜继续保持固定模型、Agent 行为、P:D 比例与 D 亲和性；第一版只比较 D-local 与固定 P prefill 后回到固定 D。暂不同时引入 D migration、P:D sizing、offload 或 D→P 传输。目标不是提高 P 的利用率或人为增加 PD 选择比例，而是减少系统实际延迟与资源浪费。

原始 Dynamic 及其离线结果保留为 baseline。此前讨论的 F/W/X 成本模型、细粒度混合 batch 干扰预测，以及路由与 TTL 联合优化，保留为后续备选；本次没有将其纳入第一版实施范围。

## 2. 将旧 workload/QPS 改为请求工作量与节点状态

### 2.1 workload 应当与候选节点相关

按实际 chat template、tools schema 和 reasoning 回放规则得到完整 token 序列后，区分：

| 符号 | 含义 | 第一版用途 |
|---|---|---|
| H | 本轮保留的逻辑历史长度 | 解释历史增长和历史重算，保留日志 |
| A | 本轮逻辑新增输入，含新消息/observation 及相应模板标记 | 解释 append，保留日志 |
| N | 完整 prompt 长度，按同一 token 序列切分时 N=H+A | 查表原始输入 |
| C_i | 节点 i 当前匹配、有效且 ready 的连续前缀 | 查询并记录，不能由逻辑 H 直接推定 |
| U_i | 节点 i 需要计算的后缀长度 | 查表原始输入，U_i=N−C_i |

若 C_i≤H，则 U_i=(H−C_i)+A，分别对应缺失历史与逻辑新增输入。重复请求等情况下若 C_i 超过 H，仍使用一般式 N−C_i。长度必须按本次完整序列计算，不能将各消息字符数或单独 tokenize 的长度直接相加。历史改写后，也不能假定本轮是上一轮生成序列的纯追加。

对每个候选节点，基本关系为：

```text
U_i = N - C_i
```

实际使用时还要处理 block 对齐、末 token 重算等约束。至少记录：

```text
N, C_P, C_D, U_P, U_D
```

v1 进一步冻结：H 取本次完整序列中已证实的历史/setup 边界，不用跨轮 LCP 替代；不能证明边界时 H/A=null。C 是已按引擎规则修正的有效 ready 前缀，普通新生成请求满足 C<N，U≥1；未知 C/U 保持 null。旧 proxy 的估算 context/suffix 字段不能直接改名升级为这些值。具体规则见规范契约第三、四节。

同一请求在不同节点上的 workload 不一样：

- D 保存了完整历史时，可能只需计算新 observation。
- P 只保存较早的历史时，可能还需要重算上一轮 reasoning、tool call 等内容。
- D 发生驱逐后，逻辑 append 很短也可能对应很长的实际 prefill。

因此，`append_tokens` 只是逻辑新增输入，不等于候选节点的实际待计算 tokens。前缀应按实际 token 序列和有效 block 链判断，不能由上一轮 `prompt_tokens + completion_tokens` 直接推定。

U_i 相同也不代表耗时相同：向 100K 历史追加 1K tokens，与向 4K 历史追加 1K tokens，需要访问的 attention 历史不同。但 (N,U_i) 已隐含 C_i=N−U_i，因此两路径的紧凑输入 (N,U_P,U_D) 没有忽略这部分历史。给定 N/C_i 后，逻辑 H 不额外增加标准 prefill 的主要算术规模信息，暂不再设为独立查表轴。

`max_tokens` 保留为长度上限和安全约束，不作为普通路径成本中的预计生成长度。已完成请求的真实 completion 可以用于事后分析和在线统计，但不能倒用为该请求路由时已知的信息。

### 2.2 不把 QPS 改名为 GPU 负载

QPS 继续表示单位时间的请求到达数，明确 logical/physical、全局/节点、窗口大小和时间戳。滑动窗口 QPS 可描述流量趋势，但替代旧查表 QPS 维度的应是结构化节点状态。

| 状态 | 建议记录什么 | 为什么 QPS/session 总数不够 |
|---|---|---|
| Prefill 积压 | 待计算 tokens、对应上下文长度、估计剩余 prefill 工作量 | 多个短请求可能比一个长请求轻 |
| 等待情况 | 队列长度、最老等待时间、等待原因 | 等计算、等 KV 空间、等传输不能混为一谈 |
| Decode 负载 | 实际 decode 请求数、各请求当前上下文长度、近期迭代耗时/ITL | 相同请求数不代表相同访存压力 |
| KV 容量压力 | 正在使用的 blocks、可回收缓存、TTL 额外保护的 blocks、可分配余量 | 缓存内容仍在与缓存不能回收是两回事 |
| 请求级缓存状态 | 各候选节点的有效前缀长度、ready/inflight、保护状态 | 节点总体 hit rate 无法回答本请求命中多少 |
| 传输压力 | 待传输字节、正在进行的传输、近期传输耗时 | P 空闲不代表 P→D 路径空闲 |
| Agent 生命周期 | 请求 LLM、执行工具、结束等阶段及相应 session 数 | 工具中的 session 不消耗 decode 算力，但可能占用被 pin 的 KV |

需要区分 episode concurrency、HTTP in-flight、scheduler running 和实际 decode 序列数。vLLM 的 running 集合可能包含正在执行 chunked prefill 的请求，不能直接等同于 decode batch。

KV 物理容量按唯一 blocks 统计；多个序列共享前缀时不能重复计算物理占用。用于估计 decode 读历史成本的逻辑上下文长度，则是另一种量，不能与物理容量混用。

第一版保留查表，但只针对真实 trace 中有代表性的工作量和状态组合校准，不把所有监控字段做成笛卡尔积。下面第 2.4 至 2.7 节给出原始变量及归类方式；近期 TTFT/ITL 等结果指标先用于 profiling，不新增绕过查表的 AMPD 分支。

### 2.3 Reasoning 回放决定下一轮的有效前缀

当前模型为 Qwen3-30B-A3B-Thinking-2507，SWE-bench 一个任务内部通常是同一条真实 user message 后的多次 assistant/tool 交替。当前实验配置的 `replay_reasoning: true` 应保持不变；新路由与 TTL 不应靠切换 reasoning 策略制造缓存压力或收益。

此前本地源码核对与纯 CPU 模板渲染确认了以下区别：

| 层次 | 当前行为与证据 |
|---|---|
| Agent 日志 | 保存模型返回的 reasoning；保存 trace 不代表一定回放。[agent.py:390](/root/uni-agent/uni_agent/agents/react/agent.py:390) |
| 发往服务端的消息 | `_normalize_messages_for_api()` 按 `replay_reasoning` 控制历史 reasoning 字段。[model.py:104](/root/uni-agent/uni_agent/agents/react/model.py:104) |
| 配置 | 通用默认值为 false，本实验显式设为 true。[base.py:25](/root/uni-agent/uni_agent/agents/base.py:25)、[task_config_ppd.yaml:25](/root/uni-agent/examples/inference/task_config_ppd.yaml:25) |
| vLLM 输入规范化 | 将收到的 `reasoning` / `reasoning_content` 提供给模板；最终是否进入 token 序列仍由模板决定。[chat_utils.py:1592](/root/vllm-ppd/vllm-source/vllm/entrypoints/chat_utils.py:1592) |
| 模型原生模板 | 保留最后一条真实 user message 之后的 assistant reasoning；新增真实 user message 后，去掉更早的 reasoning。工具 observation 不等于新 user turn。[tokenizer_config.json:229](/root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507/tokenizer_config.json:229) |

纯 CPU 渲染中，`user → assistant(reasoning + tool_calls) → tool` 保留了 reasoning 测试标记；追加一条真实 user message 后，该标记不再出现。这是模板行为核对，不是新增 GPU 实验，也不能替代上线时对模板覆盖参数的检查。模型卡的 “No Thinking Content in History” 不能直接解释为每次 tool 返回都删除刚才的 reasoning。[模型说明](https://huggingface.co/Qwen/Qwen3-30B-A3B-Thinking-2507#best-practices)、[原生模板](https://huggingface.co/Qwen/Qwen3-30B-A3B-Thinking-2507/blob/main/tokenizer_config.json)

以下仅作序列示意：用 `H_old` 表示上一轮已有历史、`R` 表示本轮 reasoning、`M` 表示 assistant 的可见文本及 tool call、`O` 表示工具 observation。它们是序列片段，避免与上文长度 A 混淆。忽略模板分隔符时：

```text
已经生成：H_old + R + M
回放后下一轮：H_old + R + M + O
删除 R 后下一轮：H_old + M + O
```

回放时，原有序列更可能形成连续的可复用前缀，但会增加后续 attention 和 KV 容量成本。删除 R 后，原 M 的 KV 曾依赖 R，不能绕过 R 直接拼接复用；通常只能复用匹配的 H_old，再计算保留的后缀 M + O。新生成的干净历史仍可在以后复用，不能据此说每轮都要全量重算。

删除历史 reasoning 也不会撤销已经发生的 decode 成本，或把它从该轮 `completion_tokens` 中扣除。消息删减与 GPU 物理回收是两件事；无引用但仍缓存的 blocks 可能已经可回收。TTL 只应保护下一轮仍有效的前缀，并在回放策略改变时重新计算其价值。

后续 profiling 应分别记录 reasoning、assistant 可见文本/tool-call 序列化和 tool observation 的长度。“工具调用请求短、工具返回长”不能改写成“总 completion 一定短”，因为 reasoning 可能占据大部分输出。

### 2.4 Prefill 的计算过程及变量依据

以下以标准全因果注意力 decoder 为例，固定模型/精度，暂忽略 block 和末 token 的边界。没有缓存时，N 个输入位置都经过 embedding，并逐层执行 Q/K/V 投影、causal attention、输出投影和 MLP/MoE；各层保存 KV，最后位置的输出生成首个 token 的 logits。位置之间可以并行执行每层中的运算，不等于逐个 token 做 N 次 decode。

这包含两类工作：逐位置投影/前馈的规模主要随 N 增长；有效 query–key 配对数为：

```text
无缓存：1 + 2 + … + N = N(N+1)/2
```

有有效前缀 C 时，只为 U=N−C 个后缀位置计算各层的新 Q/K/V 和前馈结果。旧位置不再经过这些计算，但新 query 仍需读取旧 K/V，并关注此前的新位置：

```text
有缓存：Σ(j=1..U)(C+j) = C·U + U(U+1)/2
逐位置计算规模：U
```

缓存省去历史位置的重算，不消除新位置读取长历史的 attention。[KV caching 原理](https://huggingface.co/docs/transformers/cache_explanation)

示例中 H=8K、A=2K、N=10K，忽略 block 边界：

| 缓存状态 | C | U | 实际工作 |
|---|---:|---:|---|
| 未命中 | 0 | 10K | 历史与新增输入全部计算 |
| 部分命中 | 6K | 4K | 重算 2K 历史，再计算 2K 新输入 |
| 历史命中 | 8K | 2K | 计算新增 2K，但 attention 仍访问旧历史 |

因此，(C,U) 与 (N,U) 都能描述主要 prefill 算术规模；两路径选择 (N,U_P,U_D) 即可。H/A/C_P/C_D 全部保留在 trace 中用于解释 cache loss。当前 MoE 的专家分布、batch 利用率和 kernel 会造成额外差异，不能声称相同长度必有相同耗时。

上述 attention 配对数是工作量特征，不是 GPU 延迟公式，也不是 KV 存储量。FlashAttention 等实现会改变 IO 与墙钟表现；具体延迟仍由实测获得。[FlashAttention](https://arxiv.org/abs/2205.14135)

### 2.5 Decode 背景：B_D 与 K_D

普通、非 speculative 的自回归 decode 中，每条获调度序列每步处理约一个新位置：计算该位置各层的 Q/K/V，查询历史 KV，完成输出投影和 MLP/MoE，再生成下一个 token。

若本步有 B 条序列，当前历史长度分别为 L_j，则新位置计算规模约为 B，而 attention 配对数约为：

```text
Σ(j=1..B)(L_j+1) = K + B
K = Σ(j=1..B)L_j
```

这解释了第一版使用 (B_D,K_D) 的原因：B_D 描述批量规模，K_D 描述这些序列合计关注的历史。两者是路由前状态；TPOT/ITL 是执行后的表现，彼此不替代。

| 示意状态 | 序列数 | 每条历史长度 | 历史总和 |
|---|---:|---:|---:|
| A | 4 | 4K | 16K |
| B | 4 | 32K | 128K |
| C | 16 | 8K | 128K |

A/B 的 batch 相同但 attention 规模不同，B/C 的历史总和相同但新位置数和 batch 效率不同。表中数字仅用于解释变量，不是本机性能测量或容量承诺。K_D 不是去重后的物理 KV 占用，也不是精确 HBM 流量；共享前缀、长度分布和 MoE 路由都会影响实际执行。

用于在线描述需求的 B_D，是已经进入 decode、当前可参与调度的序列数，不包括执行工具中的 episode、尚在 prefill 或等待远端 KV 的请求。另记录本步实际调度的 decode 数以及被阻塞/抢占的请求，不能因“实际服务到的 batch 变小”就判断系统变闲。`running` 包含 chunked prefill，不能直接取其长度。[scheduler.py:228](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:228)

v1 的精确参考值限定为普通 n=1、非 speculative、完成结果已处理而下一次调度未开始的同步快照。已有 G≥1 个生成 token 时，下一步输入前的 KV 历史 L=原 prompt N+G−1，最后一个 sampled token 的 KV 尚未计算。无法证明 ready frontier 的 async/in-flight 状态标 unavailable，不为采集而强制 GPU 同步。

第一版查表先使用少量 (B_D,K_D) 类别；完整上下文分布留在日志中。如果同类场景仍有显著差异，再决定是否增加分布特征。

### 2.6 P/D prefill 积压：尚未完成的已知输入工作

P/D 的积压是两个节点各自的工作，不是共同队列，也不是显存中已有多少 KV。它回答的是：新请求到达前，这个节点已接下的 prefill 还有多少没有完成。

例如，P 上请求一还剩 2K，等待中的请求二需算 4K，则已知剩余 token 工作约为 6K；D 上有 8 条 decode，同时还有一个未完成的 2K append prefill，则“8 条”属于 decode 背景，“2K”属于 prefill 积压。新请求不必等这 6K/2K 严格串行完成，因为调度可以分块和合批。

令 J_i 为节点 i 当前可执行或正在执行的 prefill/恢复请求，以及已知要在本节点计算、尚未被快照确认且已去重的 pending 工作。待决策的本请求不计入背景积压，它由 N/U_P/U_D 单独描述；等待远端 KV、内存或其他准备条件的请求另列为 blocked，不进入当前可执行积压。对 r∈J_i 定义：

```text
N_r：本次已知的处理目标长度
E_r：当前已经完成并可用的前缀，含原缓存及本轮已完成 chunks
R_r = N_r − E_r：剩余待计算后缀

Q_tok,i = Σ_(r∈J_i) R_r
Q_att,i = Σ_(r∈J_i) [E_r·R_r + R_r(R_r+1)/2]
```

Q_tok 表示还有多少位置要做投影/前馈等计算；Q_att 表示这些位置的结构性 attention 工作。两个队列即使都只剩 1K tokens，接在 4K 与 100K 历史之后也不是相同工作。两项均不是毫秒，不假定可以用未经校准的线性权重换成等待时间。

积压覆盖 running 中未完成的 prefill 和已经排队可执行的 prefill；pending 按 request_id 与节点快照去重。Remote 请求在 D 上等待接收 KV 时，不把传输将提供的历史算成本地 prefill，待接收就绪后仅计实际残余工作。Preemption 恢复可能需要重算已经生成过的 tokens，因此 N_r 不一定只是最初 prompt 长度；这些已知 tokens 与未来输出预测无关。[scheduler.py:537](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:537)

统计 E_r 时需区分已调度、in-flight 和已完成进度。`num_computed_tokens` 在调度后就增加，不能直接把它当 GPU 已完成边界；不能漏掉正在执行的 chunk，也不能把同一工作在节点与 proxy 中重复相加。[scheduler.py:790](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:790)

规范契约把已知目标记为 T_r（即上文 N_r），另设已派发边界 S_r，冻结 0≤E_r≤S_r≤T_r。In-flight 为 S_r−E_r，未派发为 T_r−S_r，总剩余 T_r−E_r 只计一次。Pending 按包含 attempt、节点和 epoch 的物理 ID 对账；终止 ACK 后不能因请求从 active snapshot 消失而重新算作 pending。

第一版由 (Q_tok,Q_att) 形成少量“无/轻/重”等候选积压类别，边界待校准。最老等待年龄、请求数和最长剩余长度先作诊断字段。积压不是严格排在新请求前面的 FIFO 工作量，也不是精确等待时间。[vLLM chunked prefill](https://docs.vllm.ai/en/v0.13.0/configuration/optimization/#chunked-prefill)

### 2.7 原始状态、查表类别与诊断字段

| 描述对象 | 原始输入 | 第一版使用方式 |
|---|---|---|
| 当前请求 | N、U_P、U_D；由实际 C_P/C_D 派生 | 请求工作量类别 |
| D decode 背景 | B_D、K_D | Decode 背景类别 |
| P prefill 积压 | Q_tok,P、Q_att,P | P 积压类别 |
| D prefill 积压 | Q_tok,D、Q_att,D | D 积压类别 |

H/A、完整长度分布、实际 scheduled batch、等待年龄、TTFT/ITL、QPS 和 preemption 保留用于解释与验证。KV 容量、节点状态和快照新鲜度用于安全/有效性检查；不把所有字段都加成独立查表轴，也不把未知前缀伪装成零命中。当前仍需决定分桶边界及覆盖范围，没有预设高维全枚举矩阵。

## 3. 已确认：核心管理信息在 CPU 上

### 3.1 本地源码证据

以下区分已有 CPU 管理数据、需要增加的聚合/接口，以及必须估计的未来成本。CPU 可观测不等于当前 proxy 已经拿得到。

| 信息 | 可获得程度与源码入口 | 用途及限制 |
|---|---|---|
| prompt 长度、已生成长度、生成上限 | CPU 请求对象已有。[request.py:74](/root/vllm-ppd/vllm-source/vllm/v1/request.py:74)、[request.py:190](/root/vllm-ppd/vllm-source/vllm/v1/request.py:190) | 本轮 N 和已发生的输出可知，最终输出长度仍未知 |
| 计算进度、剩余 prompt 工作量 | CPU 上有 `num_computed_tokens` 等计数。[request.py:99](/root/vllm-ppd/vllm-source/vllm/v1/request.py:99) | 必须区分已调度与 GPU 已完成；不能直接当 ready 计数 |
| running/waiting、等待年龄与原因 | scheduler 已有集合和部分状态；完整原因分类需补充。[scheduler.py:445](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:445)、[scheduler.py:1492](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:1492) | 可派生队列长度与年龄；内存不足、token budget、远端 KV 等原因不能合并 |
| decode 序列数、上下文分布 | 从调度任务和 CPU 请求对象聚合 | 需要排除 chunked prefill；`running` 不是 decode batch 的同义词 |
| block ID、引用计数、hash、空闲链表 | CPU 管理对象已有。[kv_cache_utils.py:108](/root/vllm-ppd/vllm-source/vllm/v1/core/kv_cache_utils.py:108) | 可识别正在使用和可回收的 blocks；按唯一物理 blocks 计容量 |
| 空闲/可回收 blocks、KV usage | block pool 已有。[block_pool.py:454](/root/vllm-ppd/vllm-source/vllm/v1/core/block_pool.py:454) | free queue 可包含缓存内容，不能把 cached 与 free 当互斥集合相加 |
| hash 索引、最长可复用前缀 | CPU 索引已有；需要纯只读探测接口。[block_pool.py:182](/root/vllm-ppd/vllm-source/vllm/v1/core/block_pool.py:182)、[kv_cache_manager.py:164](/root/vllm-ppd/vllm-source/vllm/v1/core/kv_cache_manager.py:164) | 需遵守 cache group、block 对齐和就绪规则；索引键数不是物理 block 数 |
| 近期迭代耗时/ITL | 可在 CPU 接收执行结果或输出时记时。[core.py:347](/root/vllm-ppd/vllm-source/vllm/v1/engine/core.py:347) | CPU 墙钟可能包含调度、等待与结果处理，不等于纯 GPU kernel 时间 |
| 传输计划、完成/失败状态 | 已有 connector metadata 和输出；bytes、积压和分阶段计时需补齐。[sched/output.py:205](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/output.py:205)、[outputs.py:110](/root/vllm-ppd/vllm-source/vllm/v1/outputs.py:110) | 已计划或已发送不等于接收端可用；需处理 invalid blocks |
| 工具阶段、episode 结束 | 需要 Agent 生命周期信号 | 不能只凭 LLM 请求结束推断正在执行工具或 episode 已结束 |
| TTL 额外保护容量与租约 | 新策略实现后由 CPU 控制器维护 | 当前尚未接入；必要计算引用和可选 pin 必须分开 |
| 下一段等待、prefill/传输成本及 decode 干扰 | 由上述状态和校准数据估计 | 属于预测量，不是读取 CPU 计数器就能精确获得的事实 |

这些 CPU 管理结构描述 GPU blocks 的归属和状态。**它们在 CPU 上，不意味着 K/V 数值本身在 CPU 上，也不是 KV offload。** 查询前缀归属、引用计数等原则上不需要 GPU kernel 或 GPU→CPU KV 拷贝。

但这些状态属于 engine/scheduler；不能假定 HTTP API 进程或 proxy 可以直接访问其内存。当前 proxy 的在途请求统计也不是 engine 的实际执行负载，更不是请求级前缀索引。

源码链接对应讨论时的本地文件位置；后续代码变化可能移动行号。

### 3.2 建议的低干扰状态观察器

第一版可以采用候选 P/D 节点的轻量快照与批量只读 prefix probe，不必先完成全局缓存事件镜像。建议的数据路径为：

1. 节点在调度循环的一致边界生成轻量负载快照。
2. 候选节点在其状态所有者线程中执行纯只读前缀探测，返回带版本的结果。
3. proxy 合并前缀与负载快照；查询量增大后，再考虑用缓存事件维护独立 CPU 镜像。
4. 真正执行前由节点重新核验。第一版不增加 TTL/reservation；将来引入保留保证时，再由节点确认租约或引用接管。

不能在 HTTP 线程中无约束地遍历调度器正在修改的字典，也不应为远程查询长时间持有调度热路径上的锁。

状态建议携带 `node_id、epoch/version、timestamp、ready/inflight、protection`。独立索引的 tokenizer、模板、block size、hash 规则和 cache namespace 必须与节点一致，不能假设在 proxy 中随意重新 hash 就能得到相同 key。

第一版用于查表的原始输入按第 2.7 节导出。等待原因、容量与传输状态用于解释失配和检查有效性；近期迭代耗时、ITL 及窗口 QPS 先保留为观测字段，不新增前置反馈路由。工具中的 session 数可为后续保留机制提供信息，但不能替代当前 decode 需求。

### 3.3 观察不能改变缓存行为或污染统计

只读探测不能调用会增加引用计数、改变空闲队列或 LRU 行为的 `touch()`。

现成的 `get_computed_blocks()` 也不宜直接作为高频探测接口：开启统计时，它会调用 `prefix_cache_stats.record()`。影子查询若直接复用，会污染 hit-rate 指标。应抽出不改变引用、LRU 和业务命中统计的只读路径。

### 3.4 现有异步事件发布并不等于无阻塞

本地 KV event publisher 使用后台线程，但入口调用有界队列的 `.put()`；队列积压满时仍可能阻塞 serving。[kv_events.py:308](/root/vllm-ppd/vllm-source/vllm/distributed/kv_events.py:308)

因此，低干扰接入需要考虑：

- 批量事件、非阻塞遥测，以及观测开销测量。
- 序号、epoch、事件丢失检测和快照重建。
- 镜像失效或过时时显式降级，不能继续把旧状态当成准确命中。
- TTL 授予/释放等控制确认必须可靠且幂等，不能像普通遥测一样随意丢弃。

查询到缓存存在也不是保留承诺：排队期间仍可能被驱逐；传输未完成的 KV 不能视为已 ready。最终结论是“可以不读取 GPU KV 内容地获取管理信息”，不是“已经实现了零开销、实时一致的外部查询”。

### 3.5 进度、容量与传输的易错点

`num_computed_tokens` 会在调度后、GPU 结果返回前增加，不能把这个名字按字面当成已完成 tokens；推测执行等路径还可能修正进度。[scheduler.py:790](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:790)

高 prefix hit 也不意味着容量增量很小。例如，100K 历史虽然仍在 free queue 中，接入请求并取得引用后，这些 blocks 会从可回收转为不可回收。即使只需新算 1K tokens，也必须核算历史引用接管造成的容量压力。NVML 显示的显存占用不能替代这种 block 级统计，因为 vLLM 已预分配的 KV 池大小可能一直不变。

观察传输状态时不能为了取 metadata 再调用一次 `build_connector_meta()`。该接口允许消费或清空 connector 内部状态；额外调用可能改变实际传输。应从正式调度调用产生的结果复制只读遥测，并在完成回报中处理失败和 invalid blocks。[接口约定](/root/vllm-ppd/vllm-source/vllm/distributed/kv_transfer/kv_connector/v1/base.py:470)、[完成处理](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:1599)

## 4. 第一版选择规则：状态条件化的 PPD 查表

### 4.1 保留原评分，改进匹配场景的输入

第一版不再要求在线精确相加 F/W/X，也不启用“正常时 TTFT 优先、过载时保护模式”。新输入用于找到更接近当前执行条件的离线实测场景，再使用原 PPD 评分：

```text
S = w_ttft × (TTFT_remote − TTFT_local) / TTFT_remote
  − w_tpot × (TPOT_local − TPOT_remote) / TPOT_remote

有效配对数据且进入评分分支时：
S > 0  → Local
S ≤ 0  → Remote
```

当前先固定 w_ttft=w_tpot=1，沿用原始 TTFT/TPOT 统计语义，隔离输入变化的效果。若后续改用背景 decode 的短窗 ITL、tail 指标或新的目标替换其中一项，须独立命名和对照，不能称为只改输入。[评分实现](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:280)、[读取 Turn 2 平均指标](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:342)

TPOT 是实测效果，(B_D,K_D) 是路由前状态；用当前状态选择测量结果，不等于用 B_D/K_D 预测每条 decode 的精确延迟。原 score 也不是总节省秒数，不直接保证 episode 完成时间或 SLO。

### 4.2 离线表可以同时保存结果和动作

```text
请求工作量类别(N,U_P,U_D)
  + D decode 背景类别(B_D,K_D)
  + P prefill 积压类别(Q_tok,P,Q_att,P)
  + D prefill 积压类别(Q_tok,D,Q_att,D)
    → 配对的 Local/Remote 实测 TTFT/TPOT
    → score 与推荐动作
    → 在线输出 Local 或 Remote
```

每条记录保存两边指标、score 和推荐动作，并附样本量、波动、支持范围及 calibration fingerprint。保留原始指标是为了说明收益大小、审计边界和以后重算权重，不妨碍直接存储或查询动作。

不枚举所有监控变量的笛卡尔积；只对具有实际代表性的状态校准。表外状态标记未覆盖，不沿用“huge 一律外推 large”等未经新校准支持的规则。缺表、状态过时与 score 临近零的回退方式仍需在实现前冻结；不能把默认动作当成实测结果。

### 4.3 请求处理与观测边界

```text
实际完整 prompt + 固定 P/D
    → 查询有效前缀，读取带 epoch/时间戳的 CPU 状态
    → 合入尚未被节点快照确认的派发工作，避免重复统计
    → 校验路径支持与状态有效性，映射至新查表类别
    → 用已冻结的评分/默认规则选择 Local 或 Remote
    → 派发一次 logical request；retry 仍遵守已有路由契约
    → 执行后记录实测结果、查表来源与状态偏差
```

近期 TTFT/ITL、QPS 和 preemption 先进入 profiling，不作为自动绕过 lookup 的前置路由。实际容量准入仍由 engine 决定：快照不足以承诺分配安全，当前 free blocks 不足也不一定意味着永久不可执行。Remote 不能消除 D 最终持有 KV 并 decode 的需求。

只读 probe 不 pin 所有候选缓存，不触碰 LRU；查询到执行间仍可能发生 eviction。前缀未知不能直接记为 C=0，也不能由上次 response 的 cached_tokens 假定当前命中。实际复用量与计划不一致时，应保留差异以评估观测误差。

首轮路径和短 append bypass 需与旧 baseline 明确区分。复现版本保持原行为；新策略必须验收两种动作可达，避免逻辑 append<512 在查询实际 U_D 之前就屏蔽新表。具体移除或替换 bypass 的方式仍属待实施协议决策，本次未修改代码。

### 4.4 请求日志与结果解释

新策略至少记录实际请求变量、D decode 需求与 scheduled batch，以及两端积压摘要。每次决策附带状态时间戳、表版本、命中类别、两边参考指标、score、最终动作和 fallback 原因；实际执行后的 cached/uncached、TTFT/TPOT 与 preemption 继续记录。

实测 TTFT 已包含的等待或传输不能再额外加一次。第一步源码核对后明确区分：旧 score 的 `legacy_client_stream_text_v1` 计到首个非空文本事件；现有 node-engine TTFT 计本节点首次生成输出被 frontend 处理，不含 Remote 中先发生的 P 阶段。包含 reasoning 的精确 route-wide 首生成 token 尚缺采集支持，不能给旧文本/非流式 HTTP first-byte 指标换名冒充。背景 decode 的 token-age/ITL 尖峰作为诊断，不替换原 TPOT。

相同结构特征未必有相同耗时，尤其在 MoE、长度分布不均或状态变化较快时。第一版的可验证主张是“当前工作量描述更有解释力，可能改善场景匹配”，不是已经证明比 TPOT 更准确或可以保证吞吐收益。

### 4.5 后置备选：AMPD 前置判断及精细成本模型

用户提出的后续组合为：先看 P 的近期 TTFT 余量，再看 D 的近期 ITL 余量；两边都紧张时，由改进后的 PPD 查表选择。第一版不启用这三步中的前两步。今后接入时仍需检查 P 空闲但缺长历史的情况，不能用近期平均值掩盖当前请求的大量重算。

此前讨论的组件模型保留用于诊断或后续对照：

```text
T_local  ≈ W_D + F_D(N,C_D,S_D)
T_remote ≈ W_P + F_P(N,C_P,S_P) + X_actual(P→D) + W_resume,D
F⁰(C,U) 的候选拟合特征：a + b·U + c·C·U + d·U²
```

F 表示 prefill，W 表示获得执行机会前的等待，X 表示实际 connector 的传输。分块、合批、等待和通信可能重叠，不能机械相加；末端需统一到同一个首 token 锚点。传输按实际 blocks/bytes 而非逻辑 observation 长度估计。

这些是早期候选模型，不是第一版必须拟合的公式；二次项仅有算术工作量解释，没有已验证的二次延迟定律。若以后采用组件实测表或分段插值，还需独立验证背景 decode 下的误差。将队列中组件成本相加也只能形成工作量估计，不能承诺精确等待时间。

此前“normal/protect 模式、有限 TTFT 代价换 decode 保护”的控制器及逐请求混合 batch 干扰预测同样后置。它们不是当前已冻结的优化目标或默认实现。

## 5. 状态条件化校准与原始 benchmark 的保留

### 5.1 旧表与新表的区别

| | 原始 PPD Dynamic | 第一版新方案 |
|---|---|---|
| 请求输入 | context 与 input/output workload 分类 | N/U_P/U_D，依据实际有效前缀 |
| 负载输入 | QPS 分类，间接反映离线负载 | B_D/K_D 及两端 prefill 积压类别 |
| 实测标签 | matched PD/PPD 的 Turn 2 TTFT/TPOT | 保持相同评分口径，同时记录背景 decode 与状态证据 |
| 保存结果 | score/动作及校准指标 | score/动作、两边实测结果、覆盖和波动 |
| 在线行为 | 输入映射旧 cell，可能先经 bypass | 状态有效性检查后映射新 cell；无新增 AMPD 前置门控 |

新方案仍然是经验查表，不是用组件模型取代所有动作标签。原输出长度是请求特征，现在移除的是对未来 output 的依赖；实际 completion 仍是测量与评价的一部分。

### 5.2 旧数据与代码可以复用的部分

原始 benchmark 保留为复现 baseline，其 HTTP 发压、健康检查和断点恢复可以复用，但不能给旧结果换上新索引就当成完成新校准。

- [run_single_turn:402](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:402) 使用 `/v1/completions`、固定 `max_tokens=output_tokens`、`temperature=0.8` 和 `ignore_eos=True`，控制合成输出长度。
- TTFT 取流中首个非空 `choices[0].text`，TPOT 为 `(E2E−TTFT)/(completion_tokens−1)`；chat/reasoning/tool 流需单独核对解析，不直接照搬。[TTFT/TPOT:474](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:474)
- [run_conversation:539](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:539) 为无工具间隔的两轮请求，conversation arrival 不等于 live Agent 的 request arrival。
- [TurnResult:128](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:128) 的聚合数据没有完整 C_P/C_D、B_D/K_D 和 prefill 积压标签，也不能唯一反推 W/F/X。额外保留的原始服务器事件可单独审查，不能凭均值虚构状态。

保留失败点和 clean rerun 的原始记录。Timeout 不记为零延迟，也不能删掉失败后只报告成功样本。缺测 cell 是未覆盖，不是某条路径获胜的证据。

本次不要求立即重跑原始 360 点矩阵。旧数据用于原始 Dynamic 对照；新数据用于实际 KV/负载状态下的经验选择，分别版本化。

### 5.3 配对测量与覆盖范围

先从已有 trace 选择代表性的长上下文与 append，覆盖双边高复用、D 高/P 低、D 低/P 高和双边低复用；再覆盖少量 D decode 背景与 P/D 积压。分桶阈值、实验点数量和失败回退仍待校准，不预设全维度网格。

第一步已冻结参考群体 `candidate_turn2_first_attempt_v1`：指定候选的逻辑 Turn 2、首 physical attempt、非 warmup，背景和 P-only 请求不混入；平均 TPOT 为有效请求比值的算术平均，不是 token 加权平均。G≤1/缺 usage 时 TPOT=null，失败与有效样本数分别记录。生成预算的具体 calibration 取值仍待实验设计，不改变现有 Agent 上限。新的统计边界见规范契约第七、八节。

每个配对场景恢复并核验同样的实际 token 序列、两端缓存和背景请求，分别执行 Local/Remote，随机或交替安排次序。硬件/编译 warm-up 与 KV warm-up 分开。精确调度相位不能复原时，记录偏差并用独立重复估计波动，不能声称两个起点完全相同。

离线可以固定候选请求和背景请求的生成预算以控制工作量，但该值不是线上预测特征。若仅生成很短输出来测首 token，只能作为路径诊断，不能冒充完整 TTFT/TPOT matched calibration。

重复的单位是重建状态后的独立运行，不是同一次 GPU run 中的每条请求。超时与状态恢复失败分别报告；状态无效样本不能算作成功的路径比较。

### 5.4 干扰诊断与整套策略验证

原 TPOT 之外，额外记录同一背景 decode cohort 在共同固定窗口内的 ITL、token 产出与最长未产出间隔。没有 token 输出不是“没有干扰”；跨出窗口的长 prefill/传输也需标记，观察其后续恢复，不能把延迟推到窗外就算改善。这些是诊断数据，不自动进入第一版 score。

先验证新字段和类别是否能区分真实执行背景，再检查留出场景的动作差异、错选代价和回退率。Shadow 只能核验已执行路径，不能证明未选路径的反事实。

有限 live 仍固定模型、Agent 契约、样本、concurrency 与 map，保留 Static x=1 和原始 Dynamic。完整 response、tool-ready、episode、超时及质量用于判断 serving 是否真正改善；严格归因还需控制生成工作量和工具间隔。AMPD gate 与 TTL 分别后置，不能同时加入后把全部收益归给状态查表。

## 6. TTL 接口与后续联合设计备选

### 6.1 当前只预留接口

第一版仅约定后续接口所需的状态与事件：有效前缀及 ready 状态、必要引用与可选保护的区别、按唯一物理 blocks 计的保护容量，以及 request/tool/episode 生命周期和 node epoch。接口尚未实现；未启用 TTL 时不能产生 pin/unpin、改变引用计数或伪造已授予租约。缓存过期仍遵循现有引擎行为。

以下第 6.2 至 6.6 节保留长期联合设计讨论，不是本轮实施任务。TTL 改变未来缓存状态，缓存状态又影响路由；将来可以分两个时刻决策：

- 请求到达时：按当前缓存与负载选择执行路径。
- 请求结束、进入工具阶段时：按缓存保留的边际价值选择节点、前缀和期限。

将来启用后，保留状态必须反馈给路由。TTL 控制器不能把上一次路由永久锁定，路由器也不能忽略 TTL 占用对其他请求的影响。下一轮 append 和返回时间并非已知，因此以下缓存价值仍是带假设的候选估计。

### 6.2 保留价值取决于整个系统的替代路径

建议以如下概念定义节点 i 的缓存边际价值：

```text
Benefit_i = 没有节点 i 这份缓存时的最优路径成本
            - 有节点 i 这份缓存时的最优路径成本
```

这是候选模型，需要在可观测状态及明确假设下估计，不是下一轮真实成本的 oracle。

它不同于“丢掉这份 KV 就必须在本节点重算多少”：

- D 丢失缓存，但 P 保存得很好，系统损失可能小于 D 全量重算。
- D 本地路径已经很好，再保留 P 副本的额外收益可能很小。
- D 拥挤时，P 的副本又可能具有较高价值。

P、D 两份缓存的收益不能独立计算后简单相加。评估一份副本时，应条件化于另一份副本的状态；批量选择保护对象时也要更新这种边际价值。

### 6.3 TTL 的代价是额外阻止回收的容量与时间

可采用的净收益表达为：

```text
TTL 净收益
  = 预期因保护而避免的系统延迟
    - 额外保护 blocks 的容量机会成本
```

保护产生命中收益，至少要求：agent 在保护期内回来、前缀仍有效，而且不保护时缓存原本会被驱逐。它们不一定相互独立，不能随意相乘为精确概率。

**本来不会被驱逐的缓存，pin 并未增加命中，却可能减少调度余量。** 本地 vLLM 的 free queue 包含仍保存内容、但可回收的 cached blocks；缓存内容还在并不等于这些容量不能分配给新请求。[block_pool.py:164](/root/vllm-ppd/vllm-source/vllm/v1/core/block_pool.py:164)

本地 Continuum 在请求结束时可暂不释放 KV，直到 unpin 才调用 `free()`。因此 pin 确实会改变可分配容量。[pin/unpin](/root/vllm-continuum/vllm/v1/core/sched/scheduler.py:237)、[请求结束保留入口](/root/vllm-continuum/vllm/v1/core/sched/scheduler.py:1349)。

TTL 必须有节点级预算、容量压力回退和超时清理。保护成本应按额外受保护的唯一物理 blocks 统计，而不是逐 episode 把共享前缀重复计费。不能只追求 cache loss 降低而忽略 waiting、preemption 与完成时间。

### 6.4 P 与 D 的保留时间轴不同

若从 P 的 prefill 完成时开始保护，P 的等待区间可能包括：

```text
D 剩余生成时间 + 工具执行时间
```

D 在 response 完成后保护，通常只需覆盖工具间隔。因此不能把同一个工具耗时直接用作 P、D 的共同 TTL。

在不依赖 output-length 预测的前提下，可以考虑分阶段、可撤销的短租约：根据 prefill 完成、response 完成、工具开始/结束等实际事件更新保护，而不提前预测整个等待区间。如果等到工具开始才尝试保护 P，必须承认 P 的缓存可能已经不在，不能事后 pin 回已经被驱逐的内容。

这部分仍是备选方案；租约期限、续租条件与预算如何确定，需要后续验证。

### 6.5 协议与生命周期约束

1. **必要引用与可选保护分开。** 计算中、传输中的必要引用，不能与工具期间的 TTL pin 混为一谈。
2. **压力回退只撤销可选保护。** 不能释放仍被计算或传输使用的 blocks。
3. **下一轮先接管，再解除旧保护。** 避免 unpin 与请求取得有效引用之间出现驱逐窗口。
4. **到期是解除保护。** 不必立即删除仍可复用的缓存内容。
5. **按唯一物理 blocks 管理容量。** 处理共享前缀和重复 lease，不重复计费或错误减引用。
6. **使用可靠、幂等的控制确认。** 节点 epoch、lease 版本、请求取消与 episode 结束都应进入清理规则。
7. **只保护语义上仍可复用的前缀。** 若下一轮删除历史 reasoning，不能因为其后的 KV 仍在显存，就将其当成有效前缀。

上述约束是待实现设计，不代表当前 connector 或 TTL 代码已经满足所有跨节点生命周期要求。

### 6.6 避免自我强化与振荡

“D 上已经 pin，所以必须走 D；一直走 D，所以永久 pin D”不是合理的闭环。

路由每次仍应比较候选路径的当前成本；TTL 则评估下一段时间的边际保留收益。可结合保护预算、压力反馈、短租约与决策差值门槛，避免保护不断膨胀或频繁切换。

P/D 缓存覆盖长度不同本身不等于一致性错误，也不自动要求双向传输。固定 D 的单向方案可以按实际可复用前缀计算缺失部分；是否引入反向传输，是后续独立设计取舍。

## 7. AMPD 对比、PPD 继承关系与贡献边界

### 7.1 已有工作与本方案定位

| 工作 | 已覆盖的方向 | 本方案必须进一步明确的区别 |
|---|---|---|
| Zongze Li 等，2026，[*Not All Prefills Are Equal: PPD Disaggregation for Multi-turn LLM Serving*](https://arxiv.org/abs/2603.13358) | 多轮场景下选择 D 本地 append-prefill | 使用实际执行状态和有效前缀，而非仅修补旧 workload 分类 |
| Wenhao He 等，2026，[*Efficient Multi-round LLM Inference over Disaggregated Serving*](https://arxiv.org/abs/2602.14516) | AMPD 的实时负载驱动路由与调度 | 不能只重复状态感知路由，需要证明与保留决策耦合的价值 |
| Hanchen Li 等，2025 首发，[*Continuum: Efficient and Robust Multi-Turn LLM Agent Scheduling with KV Cache Time-to-Live*](https://arxiv.org/abs/2511.02230) | Agent 工具间隔中的 KV TTL | 在 P/D 多副本、路径可替代条件下重新定义保留收益和成本 |

另一个需要纳入比较的公开实现是 NVIDIA Dynamo。其讨论时可查的开发版文档包含实验性的 conditional disaggregation：用 D 侧 KV 事件估计有效输入长度，结合负载在远端 prefill 与 D 本地执行之间选择。[Conditional Disaggregation](https://docs.nvidia.com/dynamo/dev/advanced-customizations/conditional-disaggregation)、[Router Design](https://docs.nvidia.com/dynamo/dev/knowledge-base/modular-components/router/router-design)。

因此，“缓存感知 + 负载感知”本身不能直接宣称为新贡献。长期联合方向的候选贡献表述是：

> 在 agent 交替生成和执行工具的过程中，依据有效前缀、实时节点压力和缓存副本的替代价值，联合控制 prefill 路由与 KV 保留。

这是一项待验证的研究假设，不是已完成的新颖性证明；当前第一版先验证实际前缀与结构性负载能否改善经验查表，尚不验证 TTL 联合收益。Dynamo 文档描述的是讨论时的开发版实验功能，不能据此推断其在本项目负载上的实际效果；也不能将路由索引的过期 TTL 与 GPU KV pin 机制混为一谈。

### 7.2 AMPD 已经建模的成本

以下对比以 [AMPD 2602.14516v2](https://arxiv.org/html/2602.14516v2) 为依据。其第 3 节使用离线拟合的分段 α/β 成本函数，包括历史长度与增量长度决定的 `T_pre`、batch 条件下的 `T_dec`，以及上下文传输成本 `T_kv`。因此，将 prefill、队列和传输拆开估计不是本方案独有的想法。[AMPD §3](https://arxiv.org/html/2602.14516v2#S3)

此前复核的上传 PDF 是 v1（第 3、4、12 页分别对应 §3、§4.1、附录 A.1），在线 v2 也未充分披露三个底层函数的展开式、分段边界、系数或具体拟合流程。附录补充了算子 profiling 与通信 α/β 模型，但不能据此把我们提出的二次项或插值表说成 AMPD 原公式。[AMPD 附录 A.1](https://arxiv.org/html/2602.14516v2#A1.SS1)

用 H 表示 session 历史长度、A 表示本轮新增输入，省略设备参数后，其 §4.1 的成本比较可概括为：

```text
AMPD D-local：T_pre(H, A; D) + D 侧 prefill 排队成本

AMPD remote：T_pre(H, A; P) + P 侧 prefill 排队成本
             + T_kv(H; D→P) + T_kv(A; P→D)
```

AMPD 的路由不只是无条件比较这两个和式：它先使用窗口内的 P 侧 TTFT SLO 余量，再看 D 侧 ITL 余量，两边都紧张时才进入成本比较。实际系统还采用 lazy read，并尝试将历史读取与前一任务的计算重叠，因此公式各项也不能机械理解为完全串行。[AMPD §4.1](https://arxiv.org/html/2602.14516v2#S4.SS1)、[AMPD §6](https://arxiv.org/html/2602.14516v2#S6)

AMPD 已通过近期 ITL 和队列感知实际压力，不能说它完全不考虑 decode。其 `T_dec` 没有显式进入上述路由成本式；ITL 余量也不是预测眼前 prefill 对每个已有请求的增量延迟，更不是严格的接入后 SLO 保证。Algorithm 1 中 α/β 是 slack 超参数，不能与性能模型系数混为一谈。PPD 则通过离线 workload 下实测 TPOT 反映综合 decode 表现，workload 的 input/output 比例不等于当前 D 的真实负载。

“AMPD 不考虑 KV 复用”这个说法需要修正。它由绑定的 D 管理历史 KV，P 可以按需读取历史，D 还会把传回的增量合并进本地 prefix cache。更准确的区别是：上述路由公式没有显式区分各候选节点当前真实保留的 C_P/C_D，也没有把候选 P 已有部分历史时的传输折扣、缓存丢失后的重算，以及副本保留价值放在同一个决策中。论文的历史可读假设不等于证明历史永久 pin 或永不驱逐，也不能仅凭公式断言其所有实现路径都不处理缓存失效。

### 7.3 双向传输改变执行计划

| 比较项 | AMPD 论文机制 | 当前 PPD 数据面上的候选方案 |
|---|---|---|
| 历史复用 | D 管理历史；P 可通过 D→P 获得历史 KV | 分别查询 P/D 本地有效前缀；P 不能通过当前 connector 读取 D-only 历史 |
| Prefill 输入 | 逻辑 H 与新增 A，历史读取作为通信成本 | N 与实际 C_i 决定 U_i；逻辑 append 相同也可能需要不同重算量 |
| 节点负载 | 队列成本、窗口化 TTFT/ITL SLO 状态 | 第一版用 B_D/K_D 与 prefill 积压匹配实测场景；窗口反馈先作观测 |
| P→D 传输 | 模型中传回新增 KV | 当前 connector 不保证只传 append；按实际 block 列表校准 |
| 保留决策 | 路由公式未联合优化工具间隔内 P/D 副本的 TTL | 当前只留接口，路径替代价值与保护预算属于后续备选 |

一个假设例子可以说明区别。忽略 block 边界，设历史 100K、新输入 1K，D 有 100K 有效前缀，P 只有 20K：

```text
当前 PPD，D-local：F_D(C=100K, U=1K)
当前 PPD，走 P：   F_P(C=20K, U=81K) + 实际 P→D 传输

AMPD 的可用能力：P 从 D 读取历史后计算新增 1K，再传回新增 KV
```

AMPD 能否进一步利用 P 已有的 20K，需看其具体实现，不能从上述论文公式直接推出。但双向传输提供了当前 PPD 没有的执行选择；不能仅向 PPD 公式加一个通信项，就默认 P 也能只计算 1K。

当前源码中，proxy 给 P 发送完整 prompt；P scheduler 会先做本地前缀查询，再计算缺失部分。[proxy:1340](/root/vllm-ppd/ppd/comprehensive_proxy.py:1340)、[scheduler:488](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:488)、[scheduler:537](/root/vllm-ppd/vllm-source/vllm/v1/core/sched/scheduler.py:537)。所以“P 一定全量重算”也不准确；真正要比较的是 C_P 与 C_D。

producer 的外部匹配接口不提供 D→P 历史读取；P→D metadata 则收集请求 block 列表，发送端按列表提取 KV，没有按 D 当前驻留前缀裁剪为 append-only 的保证。[外部匹配:399](/root/vllm-ppd/vllm-source/vllm/distributed/kv_transfer/kv_connector/v1/p2p/p2p_nccl_connector.py:399)、[metadata:453](/root/vllm-ppd/vllm-source/vllm/distributed/kv_transfer/kv_connector/v1/p2p/p2p_nccl_connector.py:453)、[发送:366](/root/vllm-ppd/vllm-source/vllm/distributed/kv_transfer/kv_connector/v1/p2p/p2p_nccl_connector.py:366)

### 7.4 保留的 PPD 框架与替换的决策逻辑

| 层次 | 本轮设计的处理方式 |
|---|---|
| 执行架构 | 保留 P/D 分离、D-local append prefill、正常路径上的固定 D 亲和性及 P→D NCCL |
| 服务与观测 | 复用 proxy、请求协议、Agent/服务器 profiling 及原有实验管理 |
| Dynamic 决策 | 保留原 TTFT/TPOT score 与权重，替换 output/QPS 分类为实际前缀与负载类别；新策略需单独验收 bypass 和动作可达性，原实现不动 |
| 离线流程 | 保留原 benchmark 与结果；另建状态条件化的配对 calibration，不从旧均值中虚构状态标签 |
| AMPD 借鉴范围 | 参考其区分队列与 decode 压力的思路；前置 TTFT/ITL 路由后置，不引入双向 KV、NIXL、部署规划或 prefill 重排 |

代码核对入口为 [decide_with_metadata:428](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:428)、[select_servers:816](/root/vllm-ppd/ppd/comprehensive_proxy.py:816) 和 [handle_request:1312](/root/vllm-ppd/ppd/comprehensive_proxy.py:1312)。当前 proxy 中的部分 active-load 快照是 profiling 数据，并未作为上述新状态参与决策；不能将已有日志字段当作新算法已接入的证据。

实现时还要验证两种动作真正可达。注册为固定 pD 的配置可能把后续 turn 直接送 D，单独替换 decision engine 并不足够。应以原 Dynamic 的 `2P_2D + ENABLE_PPD_MODE=true` 路径为基础核对配置与路由契约，并保留相同请求在两条路径上的完整日志。固定亲和性只规定正常路由目标，不保证该节点缓存仍在。

当前第一版可描述为“实际 KV 复用与轻量 decode 负载条件化的 PPD 经验查表”。它保留 PPD 执行框架和评分方法，但改变输入与校准适用域，所以属于新策略而非原始 Dynamic 的纯复现；并未切换到 AMPD 的双向传输系统。是否形成研究贡献，取决于这些状态是否确实减少错选和性能损失，不能只凭更换字段名称下结论。

## 8. 方案初评、风险与下一步

### 8.1 当前状态与历史初评

第一版方向已收敛，语义已冻结，节点导出接口及独立账本已通过 CPU 验证；原生服务观测、proxy 自动对账、新分类和校准尚待验收或实施。保留原评分不代表其已经等价于 Agent 最终目标；主指标与尾部/失败约束还需在实验前明确。下面的历史评分不是本次方案验收结果。

2026-10-10 讨论中的机制层面初评为：效率 8/10、稳健性 7/10、PD 场景拓展性 6/10。依据分别是减少不必要的重算/传输、针对状态变化调整动作，以及把保留价值扩展到可替代的 P/D 路径。这些分数仅保留为当时的主观研究判断，**不是测量结果或收益承诺，也不是本次文档更新后的重新评分**。任务正确率与成本收益当时暂留 5/10，没有独立验证依据。

已有 4×A800、可运行的 PPD/Agent 框架和 profiling 链路，足够开展初步验证；不能由此保证大规模泛化。主要工程风险在低干扰观测、状态过时、引用生命周期和传输安全。每周投入与投稿安排尚未明确，不估算项目完成日期。

### 8.2 查表风险与后续 TTL 风险

1. 新类别仍过粗、校准稀疏或状态过时，最终没有改善选路。需验证类别内差异、表外回退与实际收益，不能把更多输入字段本身当成成功。
2. 方案只是已有状态感知路由的变体，后续 TTL 联合机制也没有额外价值。路由先与 Static/原始 Dynamic 比较，将来再比较相同路由不加 TTL、独立 TTL 和联合决策，分离贡献。
3. 后续 pin 虽减少 cache loss，却压缩可分配容量，导致排队或超时恶化。需同时观察 waiting、preemption、tool-ready 和 episode 完成时间；缓存命中不是唯一成功指标。

低容量压力下 TTL 可能没有收益；工具间隔很长、历史被改写或节点状态严重过时时，也可能不值得保护。这些应作为方法的适用边界，而非事后隐藏的失败情况。

### 8.3 建议实施顺序

1. **冻结输入与结果语义（2026-10-22 已完成规范与 CPU 参考校验）。** [契约](/root/vllm-ppd/docs/stage5_state_semantics_contract.md) 明确 N/H/A/C/U、B_D/K_D、E/S/T 进度及统计群体，Agent、P:D 与 map 不变；不宣布 TTFT 优先，不表示 collector 已接通。
2. **补只读状态观测（2026-10-23 完成实现与 CPU 验证，GPU 验收待开卡）。** 已加入默认关闭的 owner-thread API 与 block 就绪证明，独立 pending 账本处理去重/终止；CPU 开销已记录。真实 IPC、NCCL 和长上下文开销，以及 proxy ID 自动映射尚未验收，不宣布只读观测已全链路通过。TTL 仍仅有 disabled 接口。
3. **建立新校准表并做 shadow。** 按第五章采集代表性配对状态，保留原 score/权重，验证新类别与支持范围；不是给旧表改字段名。
4. **有限 live 与对照。** 单独验证状态查表，记录错选、fallback、完整响应和任务结果，不同时加 AMPD gate 或 TTL。
5. **后续另测 AMPD 组合与 TTL。** 先决定反馈门控是否有额外价值，再按明确生命周期和预算实现保留机制；各自单独验收。

保持原始 Dynamic 和历史实验结果不变，新策略、状态契约及配置独立命名。后续的实际实现与实验完成状态写入 handoff，运行数据及结论写入 experiment log；本文保留本次理论讨论及其证据边界。
