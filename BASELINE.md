# Agentic PPD profiling baseline v1

Frozen on 2026-09-02 before implementing TTL or changing the Dynamic policy.

## Exact code revisions

| Component | Branch | Tag | Commit |
| --- | --- | --- | --- |
| vllm-ppd | `baseline/agentic-ppd-v1` | `agentic-ppd-baseline-v1` | `fbfb9a19992438f6be0a439e8d125b5e81463891` |
| uni-agent | `baseline/agentic-ppd-v1` | `agentic-ppd-baseline-v1` | `7e5145b5d25e351b57b6a1b61fc6877873b77990` |

The local upstream remotes are named `upstream`. Personal GitHub forks should
be added as `origin`; do not rebase these baseline branches before publishing.

## Frozen experiment configuration

- Model: `/root/autodl-tmp/Qwen3-30B-A3B-Thinking-2507`
- dtype: BF16
- serving max model length: 155648
- agent max context length: 155648
- episode generation budget: 131072
- per-turn generation cap: 16384
- temperature: 0
- serving topology: A800 80 GB x4, 2P+2D or 2P+2pD
- agent workload: live SWE-bench episodes through Modal sandboxes

## Validation performed at freeze time

- Modified Python files passed `python -m py_compile`.
- Both server start scripts passed `bash -n`.
- Both repositories passed `git diff --check`.
- Full unit tests were not run because `pytest` is not installed in the
  current shell environment. This is an environment limitation, not a passing
  test result.
- Both frozen component worktrees were clean after commit.

## Known baseline limitations

- Dynamic request features use an approximate chars-per-four token estimator.
- Dynamic `output_tokens` uses the request generation cap rather than a live
  prediction of actual completion length.
- Proxy QPS is a lifetime average, not a short-window arrival/load estimate.
- Integer/float QPS filename formatting can prevent matched calibration files
  such as `1.0`, `2.0`, and `4.0` from loading as intended.
- Missing Dynamic calibration cells default to PPD.
- Hidden reasoning is traced but not replayed into the next API request, so the
  generated assistant sequence and reconstructed next-turn prefix can differ.
- Tool schemas contain command-dependent requirements that are not fully
  represented in JSON Schema.
- PPD conversation TTL is routing-metadata expiry, not KV-block TTL.
- Continuum/TTL scheduling has not been implemented in this baseline.

## Artifact policy

Do not commit checkpoints, model weights, secrets, raw Modal sandboxes, or large
trace directories. For each experiment, record its command, code commits,
configuration, result location, and SHA-256 checksum.
