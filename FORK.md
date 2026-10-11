# nvidia-customized-llama-cpp

Container build of [sergiuszm/ninfer-4090](https://github.com/sergiuszm/ninfer-4090) for a homelab
k3s cluster. The image is published to GitHub Container Registry on a daily schedule and is consumed
by an OpenAI-compatible inference endpoint that runs **Qwen3.8-27B at the model's full native
262,144-token context on a single 24 GB RTX 4090**.

This file describes only what this fork adds. For engine behaviour, flags and benchmarks read
[README.md](README.md).

## Provenance

| | |
|---|---|
| Upstream | `https://github.com/sergiuszm/ninfer-4090` |
| Vendored commit | `fdb7a6cf3284770b9609ef7b3eac873f84e3849f` (2026-10-07) |
| Engine version | `0.6.1-rtx3090` (see [`VERSION`](VERSION)) |
| Upstream licence | Apache-2.0, see [`LICENSE`](LICENSE) |
| CUDA | build and runtime stage both `nvidia/cuda:13.1.2` |
| Target arch | `sm_89` (upstream default in this fork), `linux/amd64` |

The source tree is vendored in-tree rather than cloned at build time so that every published image
digest maps to a reviewable commit. Upstream is Apache-2.0; the licence, copyright notices and
attribution links are retained verbatim.

### Deviations from upstream

Only three kinds of change were made to the vendored tree:

1. **Personal-path scrub.** Occurrences of the upstream author's home directory in documentation,
   eval configs and bench tooling were rewritten to `/home/example`. No code, no behaviour.
2. **`.codex/` removed.** Upstream's editor-side agent hook configuration; irrelevant to a build.
3. **Added files.** This file, [`.github/workflows/daily-release.yml`](.github/workflows/daily-release.yml),
   [`deploy/`](deploy/), and a short banner at the top of `README.md`.

`Dockerfile` is byte-for-byte upstream.

## Image

```
ghcr.io/birdrock00/nvidia-customized-llama-cpp:latest
ghcr.io/birdrock00/nvidia-customized-llama-cpp:<YYYYMMDD>
ghcr.io/birdrock00/nvidia-customized-llama-cpp:nightly          # scheduled builds
ghcr.io/birdrock00/nvidia-customized-llama-cpp:git-<sha>
ghcr.io/birdrock00/nvidia-customized-llama-cpp:buildcache       # buildx registry cache, not for use
```

The image is anonymous-pull (public). It contains `ninfer` and `ninfer-serve` only; **the model
artifact is not baked in** - mount it read-only (see below).

## Model artifact

Download once, on the host that will run the container:

```bash
NINFER_MODEL_DIR=/path/to/models bash scripts/download-qwen38.sh
```

The script pins Hugging Face revision `3526913004b1` of `neroued/Qwen3.8-27B-NInfer` (16.96 GiB) and
verifies SHA-256 `eec39564...14bf3e`. Do **not** substitute the `main` revision: it moved to a
container-v3 artifact that this engine rejects with `artifact magic is not NInfer v2`.

## Running the 262K profile

```bash
docker run --rm --gpus all --publish 8080:8080 \
  --volume /path/to/models:/models:ro \
  ghcr.io/birdrock00/nvidia-customized-llama-cpp:latest \
  ninfer-serve /models/qwen3_8_27b.ninfer \
  --host 0.0.0.0 --port 8080 \
  --max-context 262144 --kv-capacity 262144 \
  --max-concurrency 1 --max-pending-requests 16 \
  --pending-timeout-ms 600000 \
  --prefill-chunk 1024 --kv-dtype rk4v4-e8 \
  --spec mtp --draft-tokens 3 --lm-head-draft \
  --preserve-thinking
```

Why these particular flags, on a 24 GB card:

- `--kv-dtype rk4v4-e8` is the only mode that fits the full native 262,144 tokens with 4-bit keys.
  INT8 tops out near 176K; `rk2v4-e8` (2-bit keys) buys slack you do not need at 262K.
- The whole card belongs to this process. Measured startup slack is **1.4 GiB**, so nothing else may
  hold VRAM on the device - not a second LLM, not a speech model. Kubernetes will not enforce this:
  `nvidia.com/gpu: 1` is a whole-device token, not a memory partition, so co-located pods race for
  the same VRAM.
- The server validates its memory budget before it starts listening, so an oversized context exits
  at startup instead of failing a request later.
- `--prefill-chunk` must stay a multiple of 128 and at or below 2688.
- `--model-id` can override the name the OpenAI `/v1/models` payload advertises, which is how you
  keep an existing client's model string working across an engine swap.

A minimal Kubernetes pod spec is in [`deploy/`](deploy/ninfer-serve-262k.example.yaml).

## Daily release workflow

[`daily-release`](.github/workflows/daily-release.yml) runs at `30 8 * * *` UTC, and also on every
push to the default branch and on `workflow_dispatch`. It builds the vendored tree and publishes
`latest`, the date tag and a `git-<sha>` tag.

Notes for whoever maintains this:

- Layer cache lives in GHCR as the `buildcache` tag, not in the Actions cache. The CUDA devel stage
  alone is far larger than the 10 GiB Actions cache quota.
- The runner gets 16 GiB of swap because `nvcc` peak RSS on this tree can exceed the runner's RAM.
  If a scheduled build dies with a compiler signal but no error text, suspect memory.
- Builds are single-arch `linux/amd64` and push with provenance disabled so the registry serves one
  plain manifest to containerd.
- A daily build does **not** pull upstream. Syncing `sergiuszm/ninfer-4090` is deliberately a
  reviewed change: re-vendor the tree, build once by hand, re-run the retrieval gates, then merge.
  An unattended upstream bump against a pinned, hash-verified model artifact is how you ship a
  silently broken tokenizer.
- Because the deployment uses `imagePullPolicy: Always` on `:latest`, a pod restart can pick up the
  newest daily image. Pin a date tag when you want that behaviour frozen.
