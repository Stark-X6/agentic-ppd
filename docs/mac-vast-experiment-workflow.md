# Mac 本地开发与 Vast 实验协作说明

更新：2026-10-27。供 Mac 上的 Codex、Claude Code 或其他项目 AI 接手使用。

## 1. 分工与基本流程

Mac 是日常开发和研究工作的位置：在 VS Code 中编辑代码，讨论算法、维护文档，运行可在 CPU/macOS 上执行的测试，分析下载的 trace。Vast 是 Linux/CUDA 验证与实验执行位置。代码通过 Git 同步，实验产物通过 SSH/rsync 取回。

推荐每轮流程：

1. Mac 修改、检查 diff、运行适用测试，提交和推送目标分支。
2. 确定实验目的、代码提交、模型、参数、样本、预计资源和结束条件，再启用所需 GPU 实例。
3. 本地 AI 通过 SSH 检查远端环境和工作区，拉取代码，检出指定提交。
4. 用 tmux 或有退出码记录的后台作业运行实验，低频检查状态。
5. 归档原始结果、生成分析、取回 Mac，更新实验日志和 handoff。
6. 按用户已明确的停机安排停止实例，或报告需要用户在控制台执行的具体动作。

本地 AI 可以直接用自己的终端工具执行 ssh、scp、rsync，不需要为了远端实验再启动一个服务器侧 AI。SSH 执行不依赖服务器登录 Codex。Vast CLI/API 仅在需要管理实例启停、租用或查询平台状态时才需要，连接已经运行的实例只需 SSH。

## 2. 当前实例和资源边界

以下是迁移验收时记录的连接，不是实时在线保证。若实例重建，应以 Vast Connect 面板的新参数为准。

| 项目 | 已知配置 |
|---|---|
| SSH host | `89.25.223.118` |
| SSH port / user | `40691` / `root` |
| GPU | 单张 RTX PRO 6000 Blackwell Max-Q，实测97887MiB |
| CPU / RAM 配额 | 约52.66核 / 457GiB，来自cgroup |
| 项目 | `/root/agentic-ppd`、`/root/vllm-ppd`、`/root/uni-agent`、`/root/vllm-continuum` |
| Python | `/root/miniconda3/bin/python` |
| 环境入口 | `/etc/profile.d/ppd-env.sh` |
| 模型 | `/root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507` |
| 数据集 | `/root/data/swe_agent/swe_bench_verified.parquet` |
| 迁移验收记录 | `/root/ppd-migration/` |
| 单卡模型API | `http://127.0.0.1:8000/v1` |
| Jupyter | 8080，与模型API不同 |

目前只完成单卡部署和文本/reasoning/tool-call smoke。原始四卡 `2P:2D` benchmark、NCCL 多节点路径及新 observer 的原生验收未在新卡上完成。单卡不能直接运行需要 GPU0—3 的 start_2P_2D.sh 或 start_2P_2pD.sh。单卡测量也不能作为原四卡性能基线。

新卡为sm120；原A800编译物不通用。当前已完成fork原生扩展构建，普通Python代码调整通常不需要全量重建；CUDA/C++扩展或相关依赖变化则需要重新判断。

## 3. Mac 首次配置 SSH

先检查 Mac 是否已有用于 Vast 的密钥；已有可用密钥就复用。若没有，且下面文件名尚不存在，可在 Mac 终端生成：

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_vast_ppd -C "mac-vast-ppd"
```

将对应 `.pub` 公钥加入 Vast 当前实例的 SSH 授权。官方说明：账号新增的 key 面向新实例；已存在实例需要在实例自己的 SSH 界面添加，不会自动继承。若该实例界面无法更新，可由已有授权的连接把用户提供的公钥加入 authorized_keys。

此前自动迁移使用的是 AutoDL 上的密钥。Mac 要使用自己的授权密钥，不需要复制 AutoDL 私钥。

在 Mac 的 `~/.ssh/config` 加一个 Host；若已有同名配置，应修改原条目，不重复追加：

```sshconfig
Host vast-ppd
    HostName 89.25.223.118
    User root
    Port 40691
    IdentityFile ~/.ssh/id_ed25519_vast_ppd
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ConnectTimeout 12
```

IdentityFile按实际密钥修改。首次在Mac普通终端连接成功，完成主机确认和必要的密钥解锁：

```bash
ssh vast-ppd
```

有口令的key可先用ssh-add加载到SSH agent，本地AI才可用BatchMode执行。随后本地AI验证：

```bash
ssh -o BatchMode=yes -o ConnectTimeout=12 vast-ppd \
  'hostname; nvidia-smi -L; /root/miniconda3/bin/python --version'
```

连接失败时区分实例未启动、IP/端口变化、密钥未授权和网络问题；不要连续高频重试。SSH错误并不意味着需要重装环境。

## 4. 实验前只读检查

一次SSH可完成以下检查；这不会启动模型：

```bash
ssh -o BatchMode=yes vast-ppd 'bash -s' <<'REMOTE'
set -e
source /etc/profile.d/ppd-env.sh
hostname
nvidia-smi
df -h /root
python --version
python /root/ppd-migration/status.py
git -C /root/agentic-ppd status --short --branch
git -C /root/vllm-ppd status --short --branch
git -C /root/uni-agent status --short --branch
git -C /root/vllm-ppd rev-parse HEAD
git -C /root/uni-agent rev-parse HEAD
test -d /root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507
test -f /root/data/swe_agent/swe_bench_verified.parquet
REMOTE
```

status.py中的validation-status.json是上次验收记录；stage=ready不表示此刻server仍在运行。实际服务健康需另外查询：

```bash
ssh vast-ppd 'curl --noproxy "*" -fsS --max-time 10 http://127.0.0.1:8000/health'
```

如需确认正在导入fork，可在source环境后检查：

```bash
ssh vast-ppd 'bash -s' <<'REMOTE'
set -e
source /etc/profile.d/ppd-env.sh
python -c 'import importlib.util; print(importlib.util.find_spec("vllm").origin)'
REMOTE
```

应指向 `/root/vllm-ppd/vllm-source/vllm/`。最终依赖中有一项已记录的FlashInfer声明例外：实际0.6.14，vLLM metadata要求0.5.3。不要见到pip check非零就擅自降级；先核对 `/root/ppd-migration/pip-check-after-build.txt` 与实际错误。

## 5. 从 Mac 同步代码到服务器

以Mac提交为实验输入，避免一边在服务器改代码、一边在Mac改同一文件。每个组件是独立Git仓库，必须分别记录提交。

Mac上先检查和提交本次文件，再推送对应分支。例如：

```bash
cd ~/Projects/agentic-workspace/vllm-ppd
git status --short --branch
git diff
```

按本次实际改动选择文件执行git add、commit和push。不要把原始trace、密钥、环境目录或模型权重加入提交。原始实验结果不靠普通Git同步。

远端先确认目标仓库没有用户未提交修改；若有修改，先记录diff并处理归属，不能直接reset。随后fetch：

```bash
ssh vast-ppd 'git -C /root/vllm-ppd fetch origin'
ssh vast-ppd 'git -C /root/uni-agent fetch origin'
ssh vast-ppd 'git -C /root/agentic-ppd fetch origin'
```

本地AI应把目标完整SHA写入实验计划，再在远端检出该SHA。下面是需要填入实际值的模板，不可照搬占位符：

```text
git -C /root/vllm-ppd switch --detach <PPD_COMMIT_SHA>
git -C /root/uni-agent switch --detach <AGENT_COMMIT_SHA>
git -C /root/agentic-ppd switch --detach <CONTROL_COMMIT_SHA>
git -C /root/uni-agent submodule update --init --recursive
```

远端GitHub凭证与Mac分开；如果fetch遇到private仓库权限问题，先处理该凭证问题。已有clone不应因此直接删掉重建。

同一轮实验开始后代码固定。已启动Python进程不会因checkout而自动加载新逻辑；更新服务代码后要在确认无任务运行的情况下重启相关服务。新建worktree时还需检查editable vLLM安装、PYTHONPATH与代码路径是否仍指向旧目录，不能仅凭git HEAD宣称正在运行新代码。

## 6. 单卡验证与多卡实验的入口

### 6.1 单卡

需要启动单卡模型服务时，先确认8000未被占用且无正在进行的任务，再在远端tmux中source环境，用已验收参数启动：

```bash
ssh -t vast-ppd
tmux new -s ppd-single-gpu
```

在tmux会话里：

```bash
source /etc/profile.d/ppd-env.sh
python -m vllm.entrypoints.openai.api_server \
  --model /root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507 \
  --served-model-name /root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507 \
  --host 127.0.0.1 --port 8000 \
  --dtype bfloat16 --max-model-len 155648 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 4 --max-num-batched-tokens 2048 \
  --enforce-eager --enable-prefix-caching \
  --enable-auto-tool-choice --tool-call-parser hermes \
  --reasoning-parser deepseek_r1
```

这些参数用于单卡smoke，不是正式性能配置。启动观察器需要在服务进程启动前设置 `VLLM_PPD_OBSERVER_ENABLED=1`，并先核对其支持的调度范围；不能把接口404当作损坏，因为默认关闭时应404。

不要用 `/root/ppd-migration/finish-setup.py` 作为通用重启命令；它是一次性迁移验收器，会检查端口并改写验收记录。

### 6.2 多卡 PPD

只有在连接目标已具备所需4GPU、环境和完整数据时，才使用：

- Static x=1：`/root/vllm-ppd/scripts/server/start_2P_2pD.sh`。
- 原始Dynamic：`/root/vllm-ppd/scripts/server/start_2P_2D.sh`，启动环境 `ENABLE_PPD_MODE=true`、`W_TTFT=1`、`W_TPOT=1`、明确的 `PPD_BENCHMARK_PATH`。
- 两者保持相同模型、Agent contract、样本和 `PPD_ROUTING_MAP_PATH`，设置 `PPD_ROUTING_MAP_STRICT=true`。

这些脚本会清理已有服务和滚动日志，运行前必须确认没有其他实验并归档需要的日志。新硬件不能直接沿用旧calibration做性能有效性声明。现有单卡实例不能作为上述脚本的运行目标。

## 7. Live Agent profiling 的真实入口

以下是多卡stack/proxy已健康、Modal认证已就绪后的“小规模命令模板”。它会真的运行任务；只有用户要求开展该轮实验时才执行。换run-id，避免覆盖已有run。若实验计划规定其他样本、并发或预算，应按计划填入。

在远端tmux中运行：

```bash
source /etc/profile.d/ppd-env.sh
cd /root/uni-agent

bash /root/vllm-ppd/scripts/run_agent_profile.sh \
  --run-id mac_smoke_YYYYMMDDTHHMMSSZ -- \
  env NUM_WORKERS=2 \
  python examples/inference/parallel_infer_api.py \
    --data-path /root/data/swe_agent/swe_bench_verified.parquet \
    --task-config examples/inference/task_config_ppd.yaml \
    --base-url http://127.0.0.1:10001/v1 \
    --model /root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507 \
    --limit 2 --n 1 --concurrency 2
```

`--limit`是task数，`--n`是每task rollout数，`--concurrency`是全局最多并发rollouts，NUM_WORKERS控制worker actor数量，不能只改它就假定并发改变。

wrapper会生成 `/root/vllm-ppd/logs/runs/<RUN_ID>/`，设置PPD_RUN_ID、trace/log输出，运行metrics poller、Agent并最后调用v2 analyzer。metrics采样默认0.5秒，用于profiling；这与AI每几分钟查看一次进度是两件事。

wrapper的manifest只覆盖部分信息；AI还应记录所有相关仓库SHA/dirty diff、硬件、模型/依赖、完整参数、样本与map/calibration指纹，并核验proxy状态中的实际契约。调用wrapper不代表全部server日志已经归档，完成后要补必要原始服务日志。

不要套用uni-agent里 `examples/inference/run_infer.sh` 的大规模RL/集群示例：它使用另一套模型、Ray job和并行参数，不是当前PPD实验的启动入口。

单卡8000可做直连Agent功能测试，但它绕过PPD proxy；必须单独标注为功能smoke，不能冒称原PPD请求级profiling完整复现。

## 8. 后台运行与低频监控

tmux会话用Ctrl-b再按d分离，任务继续运行；重新连接用 `tmux attach -t <SESSION>`。Mac可以在服务端任务正常启动并持久运行后断开SSH，不能假定在普通交互shell启动的前台进程一定会继续。

启动时向用户或负责监控的Claude Code交代：SSH别名、tmux会话、run-id、run目录、预期结束标志和停止边界。默认约5—10分钟查一次；短smoke可适当缩短。只在完成、失败、明显停滞或需要人工处理时通知。

本地命令示例，替换RUN_ID：

```bash
ssh vast-ppd 'tmux list-sessions'
ssh vast-ppd 'tail -n 30 /root/vllm-ppd/logs/runs/RUN_ID/agent.log'
ssh vast-ppd 'tail -n 8 /root/vllm-ppd/logs/runs/RUN_ID/manifest.txt'
ssh vast-ppd 'nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv,noheader'
```

结束判据包括manifest中出现end_ts/workload_status、退出状态和相应产物，而不只看GPU利用率变低。进入长工具等待时GPU可能空闲。失败时先取少量末尾日志定位，不要无条件重跑整个实验。

`run_agent_profile.sh`的最终退出码还包含analyzer失败的情况；workload_status=0但analysis失败仍需修复或重新分析，不能静默报告全部成功。不要同时启动多个分析器反复扫描大trace。

## 9. 取回与分析结果

在Mac创建本次结果目录，将整个run目录取回。下面的RUN_ID是占位符：

```bash
mkdir -p ~/Projects/agentic-workspace/agentic-ppd/artifacts/runs/RUN_ID

rsync -avP -e ssh \
  vast-ppd:/root/vllm-ppd/logs/runs/RUN_ID/ \
  ~/Projects/agentic-workspace/agentic-ppd/artifacts/runs/RUN_ID/
```

`agentic-ppd/artifacts/`已被gitignore。这里没有--delete，重复下载用于补传，不会删除Mac上的额外文件。rsync需两端有程序；否则用scp取回明确目录。

关键产物包括manifest、Agent log/rollouts/traces、metrics.csv、proxy timing、request_profile.jsonl/csv、analysis.txt、proxy start/end status，以及有价值的server logs。验证结束标志、样本数、文件大小和校验信息后再考虑清理远端。

已核对v2分析器只使用Python标准库，需Python3.10+；重新分析不需要vLLM、PyTorch、CUDA或GPU。指定--run-dir后从目录内读取proxy_timing.jsonl、agent_trace*.json和metrics.csv，不依赖服务器的/root路径。重新运行会更新该本地目录内的request_profile.jsonl/csv。

如要在Mac重新分析，执行：

```bash
cd ~/Projects/agentic-workspace
python3 vllm-ppd/scripts/profile_analyze_v2.py \
  --run-dir agentic-ppd/artifacts/runs/RUN_ID
```

本地AI按以下顺序读取：

1. manifest.txt、冻结配置和proxy状态：确认代码/参数、完成标志、样本与路由，避免比较错run。
2. aggregate_summary.json（若该run生成过）和analysis.txt：先了解总体耗时、失败、cache loss和工具长尾。aggregate_summary不是当前wrapper保证自动生成的文件，缺少它不应伪造统计。
3. request_profile.csv：按episode、step、logical request与attempt筛选慢请求；检查queue/prefill/decode、cache、节点和重试。CSV不包含全部嵌套细节。
4. request_profile.jsonl、proxy_timing.jsonl、metrics.csv：针对所选请求检查完整字段与同节点时间窗，不把整个文件直接粘入模型上下文。
5. 对应agent_trace*.json、rollouts/和必要的server_logs/：定位具体request/response、reasoning、工具参数、observation、Modal阶段与错误。

原始JSONL逐行处理；大型trace按episode和step取片段，保留原始文件。AI先生成小表和统计，再带有针对性的证据解释原因。文件中历史/root路径保留为事实，不要求Mac创建相同路径。

下载后先核验proxy_timing和metrics非空、trace数量符合episode数量及join覆盖。分析器某些缺失输入会按空集合处理，退出成功不保证数据完整；只看到analysis.txt不能据此认定采集完整。

两个现有代表性run在源端的整个目录约40MiB（Dynamic c14）和49MiB（Static c14），包括原始trace与日志，适合完整取回。未来更大的run可先取摘要与请求表，但完整原始证据仍需独立备份。

可以给本地AI：

> 先将指定RUN_ID的完整目录从vast-ppd同步到agentic-ppd/artifacts/runs/，检查采集完整性；随后在Mac用profile_analyze_v2.py离线分析。先读manifest、摘要和request_profile.csv，筛选慢请求/cache loss，再按request ID与episode/step读取原始JSONL、trace和metrics窗口。不要一次性加载所有重复历史。将结论写入experiment log，并注明样本、版本、缺失数据及live轨迹分叉的限制。

在 `agentic-ppd/docs/agent-ppd-experiment-log.md` 记录实验和数字，在handoff更新当前结论与下一步。live Agent轨迹分叉、失败generation成本及cache loss不等于容量驱逐等边界沿用研究交接文档。

## 10. 费用与停机

停止vLLM进程不会停止Vast实例费用；要在Vast控制台或获授权的CLI中停止实例。停止后通常保留磁盘并继续收存储费，重启需等待GPU可用；Destroy会删除实例数据。因此按用户计划执行Stop，不能用Destroy代替。

Mac本地分析、写文档或讨论方案时不需要运行GPU实例。SSH只读检查同样要求实例可连接；实例停止期间不应为一次普通代码修改而重新启动它。

若用户已授权“本轮结束后停实例”，AI应在结果保存后落实该动作；没有平台API凭证或明确实例ID时，指出控制台中待停止的目标，不能猜测ID或在服务器中执行shutdown冒充平台Stop。

## 11. 可直接给本地项目AI的指令

> 项目采用Mac本地编辑、Git同步、Vast运行实验的方式。请先阅读agentic-ppd/docs/mac-local-project-handoff-2026-10-27.md和mac-vast-experiment-workflow.md。Mac项目根目录是当前agentic-workspace，几个组件是同级独立Git仓库。你可以用本地终端工具通过SSH直接管理远端实验，不需要在服务器再开一个Codex。
>
> 当前已记录Vast主机root@89.25.223.118，端口40691，建议使用Mac的SSH别名vast-ppd。先核对~/.ssh/config与Mac密钥授权，再用BatchMode检查连通。现有实例新增公钥需要添加到该实例，而不仅是账号。若实例停止或密钥未授权，报告具体情况，不要高频重试或重装环境。
>
> 连接成功后先source /etc/profile.d/ppd-env.sh，检查nvidia-smi、磁盘、三个仓库状态与SHA、模型/数据集、上次验收记录和实时health。Python在/root/miniconda3/bin/python，项目在/root/agentic-ppd、/root/vllm-ppd、/root/uni-agent，迁移记录在/root/ppd-migration。
>
> 当前只有一张RTX PRO 6000，单卡8000端口服务做过验收；四卡2P:2D实验和新observer原生GPU/IPC验收尚未完成。先按我本轮明确的实验范围推进，不能直接运行四卡脚本或完整offline benchmark。
>
> 日常在Mac修改并测试，提交推送后让远端fetch并检出确定SHA，记录PPD和Agent两仓库版本。远端未提交修改先检查归属。实验用tmux运行，生成独立run-id并保存参数/样本/map/calibration与硬件信息。默认5—10分钟检查一次，只在完成、失败或需要处理时通知；如果我让Claude Code监控，交代会话、日志和结束条件即可。
>
> 结束后核验退出状态和trace完整性，取回整个run目录到本地agentic-ppd/artifacts/runs，再分析并更新experiment log与handoff。只停止我授权的这轮任务/实例。现在请先做连接与环境的只读检查，汇报是否具备开展我指定实验的条件。

## 12. 依据与关联文档

- [详细研究与迁移交接](./mac-local-project-handoff-2026-10-27.md)。
- [现有请求/路由契约](../../vllm-ppd/docs/agent_serving_contract.md)。
- [观察器支持与验收边界](../../vllm-ppd/docs/stage5_readonly_observer.md)。
- [Vast官方SSH说明](https://docs.vast.ai/guides/instances/connect/ssh)。
- [Vast官方数据传输说明](https://docs.vast.ai/guides/instances/storage/data-movement)。
- [Vast官方实例管理说明](https://docs.vast.ai/guides/instances/manage-instances)。

本文是协作操作说明；编写时没有远端启动、重启或停止任何实验，也未修改SSH密钥或平台设置。
