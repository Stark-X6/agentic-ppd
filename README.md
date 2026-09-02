# Agentic PPD

This repository is the experiment-control repository for the live-agent PPD
profiling baseline. It intentionally does not copy the source of vLLM-PPD or
uni-agent. Each component keeps its own upstream history and is pinned by exact
commit in `manifests/baseline-v1.yaml`.

## Component repositories

- `Stark-X6/vllm-ppd`: fork of `freelulul/vllm-ppd`
- `Stark-X6/uni-agent`: fork of `verl-project/uni-agent`
- `Stark-X6/agentic-ppd`: this experiment manifest and documentation repository

The frozen code baseline is `agentic-ppd-baseline-v1`. See
`BASELINE.md` for configuration, validation status, and known limitations.

Large model weights, Modal artifacts, metrics, and raw traces are not committed
to normal Git history. Store them separately and record their URI and SHA-256 in
an experiment manifest.
