# PPD Dynamic 推理：校准、函数调用与通信流程

更新时间：2026-09-15

PPD Dynamic 分成两个时间阶段：

1. `comprehensive_benchmark.py` 提前运行真实 GPU benchmark，生成 offline calibration JSON；
2. `comprehensive_proxy.py` 在线加载这些 JSON，对 Agent、ShareGPT 或其他客户端的每一轮请求决定走传统 PD，还是绕过 P 在 D 上执行 append-prefill。

因此，`comprehensive_benchmark.py` 是 offline calibration 工具，但它仍会启动或连接真实的 vLLM 服务并发送 HTTP 请求。“Offline”表示校准发生在在线决策之前，并不表示不使用 GPU。运行 live Agent Dynamic 时通常不再运行这个脚本，proxy 只读取它之前产出的 JSON。

## 一、整体函数调用与通信关系

```text
阶段 A：offline calibration

comprehensive_benchmark.main
  └─ run_config_benchmark
      └─ run_benchmark_point
          └─ run_benchmark_point_inner
              ├─ 生成 Poisson conversation arrivals
              ├─ asyncio.gather(...)
              │   └─ run_conversation
              │       ├─ run_single_turn(Turn 1)
              │       └─ run_single_turn(Turn 2)
              │           └─ aiohttp session.post(:10001)
              │               └─ comprehensive_proxy
              │                   └─ vLLM P/D 节点
              └─ 统计 TTFT/TPOT/success rate
                  └─ save_result(JSON)


阶段 B：online Dynamic

start_2P_2D.sh
  └─ comprehensive_proxy.main
      ├─ PPDDecisionEngine(...)
      │   ├─ 读取阶段 A 的 JSON
      │   ├─ 比较 2P_2D 和 2P_2pD
      │   └─ 构造 lookup table
      ├─ 启动 ZMQ service discovery
      └─ app.run(:10001)
          └─ 等待 Agent/benchmark 请求
              └─ handle_request
                  ├─ 提取 append/output/context/QPS
                  ├─ decide_with_metadata
                  ├─ select_servers
                  ├─ x=0：P → D
                  └─ x=1：直接 D
```

## 二、阶段 A：offline benchmark 如何生成校准表

### 1. 遍历配置、workload 和 QPS

[`run_config_benchmark`](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:823) 遍历 workload 和 QPS：

```python
for workload in workloads:
    for qps in qps_points:
        result = await run_benchmark_point(...)
        save_result(result, ...)
```

对 2P+2D Dynamic 决策，最重要的是成对测量：

```text
2P_2D   ：x=0，Turn 2 经过 P
2P_2pD  ：x=1，Turn 2 在 D 本地执行
```

同一 workload/QPS 下必须有两份结果，decision engine 才能比较两条路径。

### 2. 一个 benchmark point 生成多个 conversation

`run_benchmark_point_inner(config, workload, qps, t1_config, t2_config)` 根据 QPS 生成 Poisson conversation arrival：

```python
interval = random.expovariate(qps)
```

随后使用：

```python
await asyncio.gather(*tasks)
```

并发执行多个 `run_conversation()`。每个 conversation 固定两轮：

```text
Turn 1 完成
    ↓
立即发送 Turn 2
```

两轮之间没有工具调用、用户等待或 Modal gap。

### 3. `run_single_turn()` 先访问 proxy

```python
async with session.post(
    "http://localhost:10001/v1/completions",
    json=request_data,
) as response:
```

这次 HTTP POST 访问的是 proxy，不是直接访问 vLLM。Proxy 再根据静态配置把请求转发给 P、D 或 pD。

### 4. 保存 calibration JSON

一个 workload/QPS 点完成后，[`save_result`](/root/vllm-ppd/scripts/benchmark/comprehensive_benchmark.py:773) 保存类似：

```json
{
  "config": "2P_2D",
  "workload": "large_mid_bal",
  "qps": 2.0,
  "turn1": {
    "avg_ttft_ms": 0,
    "avg_tpot_ms": 0
  },
  "turn2": {
    "avg_ttft_ms": 0,
    "avg_tpot_ms": 0
  },
  "success_rate": 100,
  "throughput": {}
}
```

示例中的延迟数字只是字段结构示意。Dynamic 实际主要读取：

- Turn 2 average TTFT；
- Turn 2 average TPOT；
- request transport success rate。

然后比较相同 workload/QPS 下的 `2P_2D` 与 `2P_2pD` 文件。

## 三、Dynamic proxy 启动时如何建立决策表

启动 Dynamic 时使用基础 `2P_2D` 拓扑，并设置：

```bash
ENABLE_PPD_MODE=true
PPD_BENCHMARK_PATH=<calibration-directory>
```

`start_2P_2D.sh` 将其转换为：

```bash
--enable-ppd-mode
--ppd-benchmark-path <path>
```

Proxy 的 `main()` 创建：

```python
PPD_DECISION_ENGINE = PPDDecisionEngine(
    benchmark_data_path=args.ppd_benchmark_path,
    base_config=CONFIG_NAME,
    w_ttft=args.w_ttft,
    w_tpot=args.w_tpot,
)
```

见 [`comprehensive_proxy.main`](/root/vllm-ppd/ppd/comprehensive_proxy.py:1559)。当 `CONFIG_NAME=2P_2D` 时，对应关系为：

```text
base_config = 2P_2D
ppd_config  = 2P_2pD
```

### 1. 加载 matched JSON

[`_build_lookup_table`](/root/vllm-ppd/ppd/optimizer/ppd_decision_engine.py:310) 对每个离散 cell 加载：

```python
pd_result = self._load_benchmark_result(
    "2P_2D", workload, qps
)

ppd_result = self._load_benchmark_result(
    "2P_2pD", workload, qps
)
```

### 2. 计算 score

```python
ttft_improvement = (
    pd_ttft - ppd_ttft
) / pd_ttft

tpot_degradation = (
    ppd_tpot - pd_tpot
) / pd_tpot

score = (
    w_ttft * ttft_improvement
    - w_tpot * tpot_degradation
)
```

判断：

```python
use_ppd = score > 0
```

布尔值的含义为：

```text
True  = x=1 = D-local
False = x=0 = 传统 PD
```

最终生成：

```python
lookup_table[
    context_class,
    workload_type,
    qps,
] = True_or_False
```

查找表在 proxy 启动时构造一次，不会为每个在线请求重新运行 benchmark。

### 3. 启动 HTTP 服务

```python
app.run(
    host=args.host,
    port=args.http_port,
)
```

Quart proxy 开始监听 10001 端口。`app.run()` 不主动发推理请求，只负责接收和并发处理客户端请求。

## 四、在线 Dynamic 请求的函数调用过程

以下以 live Agent 的第二轮请求为例。

### 1. Agent 发送完整请求

```text
POST http://localhost:10001/v1/chat/completions
```

请求可能包含：

```json
{
  "model": "...",
  "messages": [
    {"role": "system"},
    {"role": "user"},
    {
      "role": "assistant",
      "reasoning": "...",
      "tool_calls": []
    },
    {"role": "tool", "content": "..."}
  ],
  "tools": [],
  "max_tokens": 16384
}
```

客户端发送完整历史，而不是只发送 observation。

### 2. Quart 调用 `handle_request()`

```python
@app.route("/v1/completions", methods=["POST"])
@app.route("/v1/chat/completions", methods=["POST"])
async def handle_request():
```

两个 endpoint 都映射到同一个异步 handler。它首先解析 JSON：

```python
data = await request.get_json()
```

### 3. 识别 conversation 和 retry

```python
conv_hash = get_conversation_hash(...)
logical_record = get_logical_request(...)
logical_retry = logical_record is not None
```

`conv_hash` 用于保存：

- conversation turn count；
- 已分配的 P/D；
- D affinity；
- 当前 stack epoch 下的隔离状态。

如果相同 `client_request_id` 已存在，说明是同一逻辑请求的 retry。Proxy 会复用第一次冻结的 turn、P/D、Dynamic 决策、QPS snapshot 和 workload features，不让 retry 变成新 turn。

### 4. 记录在线 request rate

```python
observed_rate = record_request_arrival(
    t_enter,
    is_logical=not logical_retry,
)
```

原始 Dynamic 的决策 QPS 为：

```text
此前 logical LLM requests / proxy uptime
```

1/5/10/60 秒窗口 QPS 当前主要用于 profiling，没有进入原始 lookup decision。

### 5. 提取 Dynamic 特征

```python
input_tokens, output_tokens, context_tokens, metadata = (
    _extract_request_features_with_metadata(data)
)
```

- `input_tokens`：最近一个 assistant message 之后新增的 messages，Agent 中通常是一个或多个 tool observations；
- `context_tokens`：最近一个 assistant message 及其之前的历史；
- `output_tokens`：请求的 `max_completion_tokens` 或 `max_tokens`，是 requested maximum，不是实际输出预测；
- `current_qps`：`record_request_arrival()` 得到的 `decision_qps_est`。

当前 append 使用 suffix tokenizer 计数；context 仍主要是估计值。

### 6. 查询 Dynamic 表

```python
ppd_decision, decision_metadata = (
    PPD_DECISION_ENGINE.decide_with_metadata(
        turn=candidate_turn,
        input_tokens=input_tokens,
        output_tokens=output_tokens,
        current_qps=decision_qps,
        context_tokens=context_tokens,
    )
)
```

内部控制流可以简化为：

```text
Turn 1？
  是 → False，强制传统 PD

Turn 2+ 且 append < bypass threshold？
  是 → True，直接 D-local

否则：
  context → small / large / huge
  input/output → workload type
  QPS → 最近离散点
  查询 lookup_table
```

返回：

```text
True  → Dynamic 选择 D-local
False → Dynamic 选择 P→D
```

## 五、`select_servers()` 如何落实布尔决策

见 [`select_servers`](/root/vllm-ppd/ppd/comprehensive_proxy.py:816)。函数先为 conversation 选择并保存：

```text
prefill_addr
decode_addr
```

例如：

```text
P = localhost:8101
D = localhost:8200
```

后续 turn 优先复用相同 D，这是 cache affinity。

在 `2P_2D` Dynamic 中，基础 mode 为：

```python
routing_mode = "pd"
```

当 `use_ppd_dynamic=True` 且 `current_turn>1` 时：

```python
if routing_mode == "pd" \
        and use_ppd_dynamic \
        and current_turn > 1:
    routing_mode = "ppd_dynamic"
```

对应关系：

| 决策 | `routing_mode` | 实际路径 |
|---|---|---|
| Turn 1 | `pd` | P → D |
| `ppd_decision=False` | `pd` | P → D |
| `ppd_decision=True` | `ppd_dynamic` | 直接 D |

`ppd_direct` 则是静态 pD topology 中 Turn 2+ 的标签，不是 Dynamic 查表产生的标签。

## 六、选择 x=1 时的通信：D-local

当 Turn 2+ 返回：

```python
ppd_decision = True
```

Proxy 设置：

```python
request_id = f"ppd_dynamic_{uuid...}"
target_url = f"http://{decode_addr}{request.path}"
```

并把原始 request 直接发给 D：

```text
Agent
  |
  | HTTP，完整 messages
  v
Proxy :10001
  |
  | HTTP，完整 messages
  v
固定 D :8200/8201
  |
  | 本地查 prefix cache
  | 对未命中部分做 append-prefill
  | 执行 decode
  v
Proxy
  |
  | HTTP response
  v
Agent
```

这条路径不访问 P，也没有本轮 P→D NCCL transfer。D 虽然收到完整 prompt，但本地缓存存在时只需计算未命中部分。

## 七、选择 x=0 时的通信：传统 PD

当：

```python
ppd_decision = False
```

路由保持 `routing_mode="pd"`，proxy 进行两次 HTTP 请求。

### 1. 向 P 发送 prefill-only 请求

Proxy 复制原请求：

```python
prefill_request = data.copy()
```

然后修改：

```python
prefill_request["max_tokens"] = 1
prefill_request["tool_choice"] = "none"
prefill_request["stream"] = False
```

再发送到：

```text
http://localhost:8100或8101/v1/...
```

`max_tokens=1` 不是让 P 回答用户，而是让 P 完成 prompt processing，同时避免大量生成。请求仍保留相同 messages 与 tool schemas，保证 chat template/KV 前缀一致。

### 2. P 通过 NCCL 向 D 传递 KV

P/D 之间的 KV tensor 不经过 proxy 的 HTTP response body。特殊 request ID 编码 P/D 的 ZMQ 地址，用于关联双方：

```text
___prefill_addr_<...>___decode_addr_<...>_<uuid>
```

通信为：

```text
P GPU
  |
  | P2pNcclConnector / NCCL
  | KV cache tensors
  v
D GPU
```

ZMQ 用于节点发现与控制信息，大块 KV tensor 走 NCCL。

### 3. 向 D 发送原始生成请求

P 请求完成后，proxy 再把原始 `data` 发送到：

```text
http://localhost:8200或8201/v1/...
```

D 根据关联的 request ID 使用 P 传来的 KV，随后执行 decode。

完整路径：

```text
Agent
  |
  | HTTP full request
  v
Proxy
  |
  +-- HTTP prefill-only --> P
  |                         |
  |                         | NCCL KV
  |                         v
  +-- HTTP original ------> D
                            |
                            | decode
                            v
Proxy <---------------- response
  |
  v
Agent
```

当前 connector 是单向 P→D。P 不会自动获得此前由 D 生成的所有历史 KV，因此在长期 D-local 后重新选择 PD 时，P 可能需要重算较长历史。这也是不能只用当前 observation 长度估计 PD 成本的原因。

## 八、Turn 1 为什么固定走 PD

Turn 1 没有本 conversation 的已有 KV：

```python
if turn == 1:
    return False
```

因此：

```text
Turn 1：Client → Proxy → P → NCCL KV → D → response
```

完成后 D 持有 Turn 1 prompt 与 output 的 KV。Turn 2 起才存在 D-local prefix reuse 的价值。

## 九、响应如何返回客户端

### Streaming

Offline benchmark 使用：

```python
stream=True
```

Proxy 返回 Quart streaming response：

```python
return Response(
    timed_stream(),
    mimetype="text/event-stream",
)
```

每收到一个 vLLM SSE chunk，proxy 就 `yield` 给 benchmark：

```text
vLLM chunk → proxy → benchmark iter_any()
```

### Non-streaming

Live Agent 通常使用 non-streaming。Proxy 收集完整 response，提取：

- prompt tokens；
- cached tokens；
- completion tokens；
- request-level PPD metrics。

然后写 profiling trace，并将完整 JSON 返回 Agent。

## 十、Dynamic 是否会使用 `comprehensive_benchmark.py`

### 1. 生成 calibration：直接使用

```text
comprehensive_benchmark.py
    ↓
matched PD/PPD JSON
    ↓
PPDDecisionEngine lookup table
```

这是原始 Dynamic 的必要前置阶段。缺失 calibration cell 时，当前代码使用默认决策，默认通常为 PPD；这不等于已经完成该 cell 的动态校准。

### 2. 运行 live Agent：不直接运行，但依赖其产物

```text
uni-agent
    → comprehensive_proxy
    → P/D vLLM
```

此时 `comprehensive_benchmark.py` 不在运行，但 proxy 通过 `PPD_BENCHMARK_PATH` 读取它此前生成的 calibration 目录。

### 3. Dynamic 可以使用其他 workload generator

在线 Dynamic 的客户端可以是：

- `sharegpt_benchmark.py`；
- uni-agent SWE-bench；
- 普通 OpenAI-compatible client；
- 有意配置时，也可以是 `comprehensive_benchmark.py --skip-startup`。

最后一种方式需要谨慎。`comprehensive_benchmark.py` 的主要职责是生成静态 calibration；若让它请求 Dynamic proxy 并把结果写回同一 calibration 目录，JSON 中的 config 标签可能不能准确表示请求实际混合走过的路径，也可能污染用于构造 lookup table 的基准数据。

推荐分工：

```text
comprehensive_benchmark.py
    = 生成静态 PD/PPD calibration

sharegpt_benchmark.py 或 uni-agent
    = 运行在线 Dynamic evaluation
```

## 十一、当前实验对应的实际路径

当前已经完成：

```text
阶段 A：
2P_2D 与 2P_2pD matched offline profiling
    ↓
获得 clean calibration JSON

阶段 B：
启动 2P_2D
ENABLE_PPD_MODE=true
    ↓
proxy 加载 calibration
    ↓
uni-agent 发送 SWE-bench live requests
```

已完成的原始 Dynamic run 中：

```text
Turn 1：全部 pd
Turn 2+：全部 ppd_dynamic
```

即控制链确实执行了，但查表结果始终为 `ppd_decision=True`，所以实际通信退化成：

```text
Turn 1：Agent → Proxy → P → D
Turn 2+：Agent → Proxy → 同一个 D
```

这与 Static x=1 的实际数据路径相同，区别主要是 route 标签和 Dynamic 决策过程。

## 十二、一句话总结

`comprehensive_benchmark.py` 负责预先测出不同 workload 下 PD 与 D-local 哪条路径更好；`PPDDecisionEngine` 将这些测量结果变成查找表；`comprehensive_proxy.py` 在线提取请求特征并查表。查到 `True` 就直接访问固定 D，查到 `False` 就先访问 P、通过 NCCL 传递 KV，再访问 D。
