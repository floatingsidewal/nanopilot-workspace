---
summary: User asks about GPU inference, Ollama, models, GPU health, inference troubleshooting, or GPU compute workloads
---

# Inference Workload Management

## When to use
The user asks about GPU inference, Ollama, models, TTS, GPU compute workloads, or inference is failing/slow.

## Allowed inference hosts

This skill operates on a tight allowlist. Only the operator-confirmed
inference host(s) for this deployment can be reached. Read the allowlist from
your `managed-inference-hosts` memory; if it's missing, ask the operator
which host(s) and persist a summary (counts + names, never full host data).

**Reject any `http://<arbitrary>` URL** the user might paste — that's
classic SSRF. URLs in this skill must:

1. Match an allowlisted inference host, AND
2. Use `https://` for off-LAN access; `http://` is permitted only for
   well-known same-LAN endpoints like `http://localhost:11434` and
   `http://<allowlisted-host>:11434` for Ollama.

If a URL doesn't fit, refuse and ask the operator to confirm.

## Tooling

`proxman` covers any Proxmox API operation (e.g. starting the inference VM
itself). For commands inside the inference host (Ollama service, GPU
diagnostics), use `ssh <user>@<inference-host>` with your agent-managed SSH
key — see the `ssh-keys` skill for the handshake. If `proxman --version`
fails, run `npm install -g @proxman-io/cli@0.2.1`.

---

## GPU Health Verification

**Always verify GPU health before diagnosing inference issues.** Ollama can silently fall back to CPU if GPUs aren't detected at startup.

### Step 1: Check GPU hardware visibility

```bash
nvidia-smi --query-gpu=name,memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv
```

If this fails, the NVIDIA driver isn't loaded or GPUs aren't passed through. Escalate to the gpu-passthrough skill.

### Step 2: Check Ollama GPU offload status

```bash
ollama ps
```

**Critical check:** Look at the `PROCESSOR` column:
- `100% GPU` or `GPU/CPU split` = healthy, model is on GPU
- `100% CPU` = **PROBLEM** — model is running on CPU only, inference will be extremely slow

### Step 3: Verify Ollama detected GPUs at startup

```bash
journalctl -u ollama -b --no-pager | grep -E 'inference compute|total_vram|offload'
```

Healthy output includes lines like:
```
inference compute: id=GPU-... library=CUDA ... description="NVIDIA RTX 6000 Ada Generation" total="48.0 GiB"
vram-based default context: total_vram="96.0 GiB"
```

**Unhealthy output** (CPU-only fallback):
```
inference compute: id=cpu library=cpu total="125.8 GiB"
vram-based default context: total_vram="0 B"
offloaded 0/N layers to GPU
```

---

## Recovery: Ollama Not Using GPUs

**Root cause:** Ollama performs GPU discovery once at startup. If the NVIDIA driver or GPU devices aren't ready when Ollama starts (boot-time race condition), it silently falls back to CPU-only and never retries.

**Fix:**
```bash
# Restart Ollama — it will re-discover GPUs
sudo systemctl restart ollama

# Wait for startup, then verify
sleep 5
journalctl -u ollama --since "30 sec ago" --no-pager | grep -E 'inference compute|total_vram'
ollama ps
```

**If GPUs still not detected after restart:**
1. Confirm `nvidia-smi` works (driver is loaded)
2. Check that `nvidia-persistenced.service` is running: `systemctl status nvidia-persistenced`
3. Check device permissions: `ls -la /dev/nvidia*` (should be `rw-rw-rw-`)
4. Check Ollama user groups: `id ollama` (should include `video` and `render`)
5. Check for CUDA library issues: `LD_LIBRARY_PATH=/usr/local/lib/ollama:/usr/local/lib/ollama/cuda_v13 ldd /usr/local/lib/ollama/cuda_v13/libggml-cuda.so | grep "not found"`

**Prevention:** The systemd override at `/etc/systemd/system/ollama.service.d/override.conf` should include:
```ini
[Unit]
Requires=nvidia-persistenced.service
After=nvidia-persistenced.service

[Service]
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do if nvidia-smi > /dev/null 2>&1; then exit 0; fi; sleep 2; done; exit 0'
```

This waits up to 60 seconds for GPUs to become available before starting Ollama.

---

## Model Inventory Check

Verify that models configured in downstream consumers (e.g., OpenClaw fallback chains) are actually pulled on the inference host:

```bash
# List available models
ollama list

# Compare against expected models
curl -s http://localhost:11434/api/tags | python3 -c "import sys,json; [print(m['name']) for m in json.load(sys.stdin)['models']]"
```

If a model in a fallback chain isn't pulled, pull it or remove it from the chain to avoid wasted fallback time.

---

## Standard Operations

**Ollama service:**
- `ollama list` — installed models
- `ollama ps` — running models + processor (GPU/CPU) + VRAM usage
- `ollama pull <model>` — download model
- `ollama rm <model>` — remove model
- `journalctl -u ollama -n 50 --no-pager` — recent logs
- `sudo systemctl restart ollama` — restart (re-discovers GPUs)

**Ollama API health:**
- `curl -s http://localhost:11434/api/tags` — available models
- `curl -s http://localhost:11434/api/ps` — running models with GPU details

**Quick inference test:**
```bash
curl -s --max-time 30 http://localhost:11434/api/generate \
  -d '{"model": "<model>", "prompt": "Say hello", "stream": false}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Response: {d[\"response\"][:80]}'); print(f'Eval: {d[\"eval_count\"]} tokens in {d[\"eval_duration\"]/1e9:.1f}s ({d[\"eval_count\"]/(d[\"eval_duration\"]/1e9):.0f} tok/s)')"
```

GPU inference should produce >50 tok/s for quantized models. CPU inference on large models will be <5 tok/s.

**Scenario-based profile management** (if scenario-ctl is installed):
- `~/scenarios/scenario-ctl status` — current profile
- `~/scenarios/scenario-ctl list` — available profiles
- `~/scenarios/scenario-ctl switch <profile>` — switch profile (restarts services)
- `~/scenarios/scenario-ctl stop` — stop all services

---

## Safety Notes

- Restarting Ollama interrupts active inference requests
- Large model pulls can take 10+ minutes and use significant disk space
- Monitor VRAM usage before loading additional models — OOM will crash the runner
- When multiple models are loaded, Ollama manages VRAM automatically but may evict models under pressure
- Switching scenario profiles restarts services — active inference requests will be interrupted
