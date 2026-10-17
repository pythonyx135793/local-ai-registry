# GTX 1080 Ti CUDA12 container evidence

The unchanged registry `lab/lab.py` from commit `a7769baeccb11a97ba3abc57aeedca645672eeea` ran on the owner's NVIDIA GeForce GTX 1080 Ti on 2026-10-17 UTC. All six gates passed: load, chat, reasoning, tools, context and speed. The generated recipe and run JSON were copied byte for byte.

| Measurement | Actual result |
| --- | --- |
| Context configuration | 32,768 tokens, one slot, f16 K/V cache |
| Context recall | 25,740 input tokens; correct code; 98.7 seconds |
| First 30-second streamed decode | 23.5 tokens/second |
| Complete uncapped streamed answer | 2,165 tokens in 108.9 seconds |
| Lab-reported prefill | 261 tokens/second |

The environment was Windows 10 with an Ubuntu 24.04 WSL2 guest, kernel `6.18.54.1-microsoft-standard-WSL2`, Docker 29.9.1, NVIDIA Container Toolkit 1.20.1 and Windows driver 582.28. Windows, WSL and the actual container reported the same physical GPU UUID and 11,264 MiB total memory. Private setup records retain those identity checks. No peak model-only VRAM measurement is claimed. Output was uncapped, with no clock, power or fan changes.

## Exact inputs and evidence

- Image: `ghcr.io/ggml-org/llama.cpp:server-cuda12-b11146@sha256:52192a23c7b258044463aebba78826cdff39a7393b713edc313324b343a7b836`.
- Weights: `unsloth/Qwen3.5-9B-GGUF@3885219b6810b007914f3a7950a8d1b469d598a5`, file `Qwen3.5-9B-Q4_K_M.gguf`, 5,680,522,464 bytes; SHA256 `03b74727a860a56338e042c4420bb3f04b2fec5734175f4cb9fa853daf52b7e8`.
- [Actual stock run JSON](../../lab-runs/gtx-1080-ti-11gb.qwen3.5-9b.llama.cpp-qwen3.5-9b-q4-k-m-32k-gtx1080ti-cuda12.32k.20261017T001049.json), raw-file SHA256 `31b60d591bdddc818bf9fe9cc77a05dac521e5fc65d74fc5d6f866b08cfc0cb9`.
- The recipe's `log` is `sha256:c17fd35d8eb2ba3c`. The stock lab hashes its JSON text before appending the file's final newline, so this differs from the raw-file checksum above.
- Rendered launch SHA256: `d52a35b685396671ebed983b6cf09bd1c9ac809013fa2a61df1172246ef71622`. Only the profile's descriptive `about` text was updated after measurement; the rendered launch is unchanged.
- The original `lab.py` SHA256 is `57ac443a28c4bf31dc75b7a62a0344d0f90a255641495433919b73265bee32e7`.

The production recipe now uses the Linux container profile. The earlier [native Windows measurements](../gtx1080ti-windows-cuda12/README.md) remain historical supporting evidence. The maintainer [requested this container replacement and waived the kit reference panel/sweep](https://github.com/sybil-solutions/local-ai-registry/pull/276#issuecomment-6190931161); independent maintainer reproduction remains required.

## Reproduction

Use a Linux Docker Engine with NVIDIA Container Toolkit and the exact weights above. Set `GPU218_MODEL_DIR` to the directory containing the checked GGUF. The commands below use the tracked profile's image and arguments, a read-only model mount and a loopback-only endpoint.

```bash
set -e
GPU218_MODEL_DIR=/path/to/Qwen3.5-9B-GGUF-3885219b
GPU218_IMAGE=ghcr.io/ggml-org/llama.cpp:server-cuda12-b11146@sha256:52192a23c7b258044463aebba78826cdff39a7393b713edc313324b343a7b836
printf '%s  %s\n' \
  03b74727a860a56338e042c4420bb3f04b2fec5734175f4cb9fa853daf52b7e8 \
  "$GPU218_MODEL_DIR/Qwen3.5-9B-Q4_K_M.gguf" | sha256sum -c -
docker pull "$GPU218_IMAGE"
GPU218_CID=$(docker create --gpus all --shm-size 16g \
  --publish 127.0.0.1:18082:8080 \
  --mount "type=bind,source=$GPU218_MODEL_DIR,target=/models,readonly" \
  --entrypoint /app/llama-server "$GPU218_IMAGE" \
  -m /models/Qwen3.5-9B-Q4_K_M.gguf --alias qwen3.5-9b \
  --host 0.0.0.0 --port 8080 --ctx-size 32768 --n-gpu-layers 999 \
  --flash-attn auto --cache-type-k f16 --cache-type-v f16 \
  --jinja --reasoning-format deepseek --parallel 1)
trap 'docker logs "$GPU218_CID"; docker stop "$GPU218_CID"; docker rm "$GPU218_CID"' EXIT
docker start "$GPU218_CID"
curl --noproxy '*' --fail --retry 300 --retry-delay 3 --retry-connrefused \
  http://127.0.0.1:18082/health
env -u LAB_API_KEY -u http_proxy -u https_proxy -u all_proxy \
  -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY python3 lab/lab.py try \
  unsloth/Qwen3.5-9B-GGUF@3885219b6810b007914f3a7950a8d1b469d598a5 \
  --model qwen3.5-9b \
  --engine llama.cpp-qwen3.5-9b-q4-k-m-32k-gtx1080ti-cuda12 \
  --card gtx-1080-ti-11gb --on endpoint \
  --endpoint http://127.0.0.1:18082 --gpu 'NVIDIA GeForce GTX 1080 Ti'
```
