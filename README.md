# Local AI Guide — Windows + AirLLM + Qwen3.8-27B

A reproducible, hardware-specific guide for running **Qwen3.8-27B through AirLLM** on Windows with an NVIDIA RTX 3050 Laptop GPU (4 GB VRAM).

> **Status (verified 2026-10-11):** Qwen3.8-27B completed a real `/v1/chat/completions` request on the target Windows machine using `AIRLLM_CACHE=stream` and `AIRLLM_MAX_CONTEXT=1024`. The default TurboCache configuration failed with CUDA out-of-memory. The successful smoke test returned `finish_reason: stop` and 6 completion tokens, but the visible output contained `</think>` and a repeated `OK`; treat this as proof of basic inference, not yet a polished chat experience.

## Target system

| Component | Observed value |
|---|---|
| OS | Windows 11 / Windows build 10.0.26300.9550 |
| Python | 3.11.9 |
| Git for Windows | 2.53.0.windows.2 |
| GPU | NVIDIA GeForce RTX 3050 Laptop GPU |
| VRAM | Approximately 4 GB (PyTorch reported 4.3 GB) |
| System RAM | 16 GB (reported by user) |
| Free disk space before setup | Approximately 112 GB |
| Workspace | `C:\AI\AirLLM-Server` |
| Hugging Face cache | `C:\hf` |
| Model ID | `Qwen/Qwen3.8-27B` |
| API port | 8000 |
| Context configuration tested for inference | 1024 tokens |
| Compression configuration | 4-bit |
| Layer cache mode tested for inference | `stream` (not `turbo`) |

## Important reality check

AirLLM streams model layers from storage; it does not turn this model into a small, fast local model. This 27B model is very demanding on a laptop with 16 GB RAM and 4 GB VRAM. Expect slow initial preparation, disk activity, memory pressure, and potentially low generation throughput. Keep sufficient disk space available.

**Important for this machine:** the server's default `turbo` cache configuration reported about 4.24 GB VRAM in use at startup and failed during generation with CUDA out-of-memory. The verified smoke test used `AIRLLM_CACHE=stream` and a 1024-token context.

The server starts model loading in a background thread. An HTTP 200 from the health endpoint alone does **not** prove the model is ready.

This procedure uses an **OpenAI-compatible API server**, not Ollama. It does not make `ollama launch claude --model qwen3.8:27b` run through AirLLM. Claude Code compatibility is not guaranteed by the setup described here.

## 1. Prerequisites

Install and verify:

- Python 3.11.x (the recorded version is **3.11.9**); add Python to PATH.
- Git for Windows (recorded version **2.53.0.windows.2**).
- A current NVIDIA Windows driver.
- Approximately 100 GB or more of free SSD space is a prudent target for this checkpoint plus cache/sharding files. This is a practical buffer, not a universal hard requirement.

Open **Command Prompt (CMD)** and run:

```bat
python --version
git --version
nvidia-smi
```

The GPU should appear in `nvidia-smi`.

## 2. Clone the server repository

```bat
cd /d C:\
mkdir AI 2>nul
cd /d C:\AI
git clone https://github.com/AIExperimentsKooth/AirLLM-Server.git
cd /d C:\AI\AirLLM-Server
dir install.bat
```

**Do not combine commands on one line without a separator.** For example, `cd /d C:\AI\AirLLM-Servermkdir C:\hf` is a malformed command. Enter each command separately.

Repository: https://github.com/AIExperimentsKooth/AirLLM-Server

## 3. Run the Windows installer

The repository's `install.bat` installed a CPU-only PyTorch package on this system, despite its message claiming a CUDA-enabled install. Its optional Windows `bitsandbytes-windows` installation also caused a CUDA DLL error. The dependency correction below was needed on the target system.

Run the installer from the repository directory:

```bat
cd /d C:\AI\AirLLM-Server
mkdir C:\hf 2>nul
set "HF_HOME=C:\hf"
set "AIRLLM_DEVICE=cuda"
set "AIRLLM_COMPRESSION=4bit"
set "AIRLLM_MAX_CONTEXT=4096"
install.bat
```

Wait until it exits. Complete the following dependency steps before starting the model.

## 4. Install the tested CUDA-enabled PyTorch version

The final tested versions were:

- `torch==2.6.0+cu124`
- `torchvision==0.21.0+cu124`
- `torchaudio==2.6.0+cu124`
- CUDA wheel index: `https://download.pytorch.org/whl/cu124`

From `C:\AI\AirLLM-Server`, run:

```bat
venv\Scripts\python.exe -m pip uninstall -y torch torchvision torchaudio
venv\Scripts\python.exe -m pip install torch==2.6.0 torchvision==0.21.0 torchaudio==2.6.0 --index-url https://download.pytorch.org/whl/cu124
```

The PyTorch wheel is multi-gigabyte. Ensure the download completes.

> **Version note:** An earlier `torch==2.13.0+cu132` build also detected CUDA, but it was replaced during troubleshooting. The final verified environment was **PyTorch 2.6.0+cu124**. Stick to this tested combination unless you intentionally test another stack.

## 5. Remove the incompatible Windows bitsandbytes package

The installer added `bitsandbytes-windows==0.37.5`, which attempted to load `libbitsandbytes_cuda116.dll` and failed. The fix was to uninstall both package variants and install the regular pinned package:

```bat
venv\Scripts\python.exe -m pip uninstall -y bitsandbytes-windows bitsandbytes
venv\Scripts\python.exe -m pip install bitsandbytes==0.43.3
```

Do **not** reinstall `bitsandbytes-windows`.

The successful dependency check in this session printed:

```text
PyTorch: 2.6.0+cu124
CUDA available: True
GPU: NVIDIA GeForce RTX 3050 Laptop GPU
bitsandbytes: 0.43.3
NF4 available: True
```

Verify with this exact command:

```bat
venv\Scripts\python.exe -c "import torch; import bitsandbytes as bnb; import bitsandbytes.functional as F; print('PyTorch:',torch.__version__); print('CUDA available:',torch.cuda.is_available()); print('GPU:',torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'NOT DETECTED'); print('bitsandbytes:',bnb.__version__); print('NF4 available:',hasattr(F,'quantize_nf4'))"
```

Do not proceed if CUDA is false or the imports fail.

## 6. Patch the server's PyTorch VRAM property bug

The first server start failed in the model-load thread with:

```text
AttributeError: 'torch._C._CudaDeviceProperties' object has no attribute 'total_mem'. Did you mean: 'total_memory'?
```

Back up the file and open it:

```bat
cd /d C:\AI\AirLLM-Server
copy server.py server.py.bak
notepad server.py
```

In Notepad press **Ctrl+H**:
- Find: `total_mem`
- Replace with: `total_memory`
- Select **Replace All**, save, and close.

The affected expression should use the valid PyTorch property:

```python
f"{torch.cuda.get_device_properties(0).total_memory/1e9:.1f} GB VRAM")
```

Keep `server.py.bak` as a rollback copy. This patch is based on the observed traceback.

## 7. Start the server with the tested low-VRAM settings

For the configuration that passed the basic inference test, use **PowerShell** (not CMD) and start `server.py` directly. Keep these environment variables in the same PowerShell window that launches Python:

```powershell
Set-Location "C:\AI\AirLLM-Server"
$env:HF_HOME = "C:\hf"
$env:HF_HUB_CACHE = "C:\hf\hub"
$env:TRANSFORMERS_CACHE = "C:\hf\hub"
$env:AIRLLM_MODEL = "Qwen/Qwen3.8-27B"
$env:AIRLLM_DEVICE = "cuda"
$env:AIRLLM_CACHE = "stream"
$env:AIRLLM_MAX_CONTEXT = "1024"

Write-Host "HF_HOME=$env:HF_HOME"
Write-Host "Cache=$env:AIRLLM_CACHE"
Write-Host "Context=$env:AIRLLM_MAX_CONTEXT"
& ".\venv\Scripts\python.exe" ".\server.py"
```

Expected startup settings:

```text
Context:   1024
Cache:     stream
Model ready: Qwen/Qwen3.8-27B
```

Do not expect to see `Initializing TurboCache` in this configuration. The ordinary `run.bat`/server defaults previously selected `turbo` and a larger context; on this 4 GB GPU, that setup failed during inference with CUDA out-of-memory.

The Hugging Face cache variables matter. The existing model files are in `C:\hf\hub`; if the server instead points at `C:\Users\<user>\.cache\huggingface`, AirLLM may try to prepare another copy and fail its disk-space check. Verify that the following directory exists before starting:

```text
C:\hf\hub\models--Qwen--Qwen3.8-27B\snapshots\1d4bf0f2ff6012fd82039f2fa52739d0dd7c60c0\splitted_model.4bit\
```

The model's compressed layer files were fully prepared in an earlier run (68/68). With the cache set correctly, the later server restart found the layer files already present.

- Keep the server terminal open.
- Keep the laptop plugged in.
- Close RAM-intensive apps and monitor disk/RAM usage.
- If C: becomes critically low (for example, less than 15 GB free), stop intentionally with Ctrl+C rather than risk filling the system drive.
- The Hugging Face unauthenticated warning is not by itself fatal. Never commit a Hugging Face token to this repository.
- `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` produced a Windows/PyTorch warning that expandable segments are unsupported on this platform. It is omitted from the recommended launch commands because it was not needed for the successful smoke test.

Check remaining C: space from a second PowerShell window:

```powershell
$d = Get-PSDrive C
'C: free space: {0:N1} GB' -f ($d.Free / 1GB)
```


## 8. Test the API

Leave the server running and open a second PowerShell window.

Health endpoint:

```powershell
Invoke-RestMethod "http://127.0.0.1:8000/health"
```

List available models:

```powershell
Invoke-RestMethod "http://127.0.0.1:8000/v1/models"
```

A health response only proves the web server is alive. Check the server terminal for `Model ready` and any traceback, then run an actual generation request:

```powershell
$body = @{
    model = "Qwen/Qwen3.8-27B"
    messages = @(
        @{ role = "user"; content = "Reply with exactly: OK" }
    )
    max_tokens = 8
    temperature = 0
} | ConvertTo-Json -Depth 5

$response = Invoke-RestMethod `
    -Uri "http://127.0.0.1:8000/v1/chat/completions" `
    -Method Post `
    -ContentType "application/json" `
    -Body $body
$response | ConvertTo-Json -Depth 8
```

**Observed successful smoke test (2026-10-11):** the endpoint returned `finish_reason: stop`, `prompt_tokens: 57`, `completion_tokens: 6`, and `total_tokens: 63`. The assistant content was `OK\n</think>\n\nOK`. This proves the current stream-mode configuration can generate text, but the visible `</think>` marker and repeated answer should be investigated separately.

## 9. Known problems and fixes

| Symptom | Cause observed | Fix |
|---|---|---|
| `install.bat` not recognized | CMD was in `C:\AI`, not the cloned repository; a `cd` and `mkdir` command had been concatenated | Run `cd /d C:\AI\AirLLM-Server`, verify with `dir install.bat` |
| `PyTorch: 2.14.1+cpu`, CUDA false | Installer selected CPU-only torch on this machine | Install pinned PyTorch/vision/audio 2.6.0 CUDA 12.4 wheels |
| `libbitsandbytes_cuda116.dll` missing | Incompatible `bitsandbytes-windows 0.37.5` package | Remove both variants and install `bitsandbytes==0.43.3`; verify imports |
| `total_mem` attribute error | Server used incorrect PyTorch CUDA property | Change `total_mem` to `total_memory` in `server.py` |
| `/health` returns 200 but inference does not work | API server can start while model loading crashes in a background thread | Inspect model-load logs, check models endpoint, and run a real completion |
| Slow first launch | Checkpoint download and layer quantization/sharding are I/O intensive | Leave process running; monitor progress, internet, RAM, and disk |
| CUDA out-of-memory during inference in `turbo` mode | TurboCache pinned too much memory for this 4 GB VRAM GPU; logs reported around 4.24 GB VRAM used before generation | Stop the server, set `AIRLLM_CACHE=stream` and `AIRLLM_MAX_CONTEXT=1024` in PowerShell, and launch `server.py` directly |
| `NotEnoughSpaceException` references `C:\Users\<user>\.cache\huggingface` | Hugging Face cache variables were not applied to the server process, so it inspected a different cache path | Set `HF_HOME=C:\hf` and `HF_HUB_CACHE=C:\hf\hub` in the same PowerShell session before launching |
| `expandable_segments not supported on this platform` | This PyTorch CUDA allocator option is unsupported in the observed Windows environment | It was non-fatal; omit `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` from the recommended launch steps |

## 10. Compatible clients and network safety

The server exposes an OpenAI-compatible endpoint at:

- Base URL: `http://127.0.0.1:8000/v1`
- Model ID: `Qwen/Qwen3.8-27B`
- API key: the basic local test does not require one.

Clients that let you configure an OpenAI-compatible base URL may connect if their API feature set matches the server implementation. **Do not assume Claude Code works with this endpoint.** Tool calls, streaming, structured outputs, and long-running coding-agent workflows need to be tested individually.

The server's logged default listener is `0.0.0.0:8000`, which makes it reachable on network interfaces. For machine-local-only use, bind to `127.0.0.1` if supported by the server's configuration. Do not expose an unauthenticated model API to untrusted networks.

## 11. Reproducibility snapshot

Versions and settings explicitly observed during the session:

```text
Windows build:          10.0.26300.9550
Python:                 3.11.9
Git for Windows:        2.53.0.windows.2
GPU:                    NVIDIA GeForce RTX 3050 Laptop GPU
PyTorch:                2.6.0+cu124
torchvision:             0.21.0+cu124
torchaudio:              2.6.0+cu124
bitsandbytes:           0.43.3
airllm:                 3.3.0
transformers:            5.12.1
Model:                  Qwen/Qwen3.8-27B
Compression:            4bit
Max context:            1024 (tested)
Cache mode:             stream (tested)
API port:               8000
HF_HOME:                C:\hf
Workspace:              C:\AI\AirLLM-Server
```

For a fuller environment snapshot from the installed environment:

```bat
venv\Scripts\python.exe -m pip freeze > requirements-windows-lock.txt
```

Review the resulting file before committing it. It pins transitive dependencies exactly as installed on that machine.

## Upstream references

- AirLLM OpenAI-compatible server: https://github.com/AIExperimentsKooth/AirLLM-Server
- Qwen3.8-27B checkpoint: https://huggingface.co/Qwen/Qwen3.8-27B
- PyTorch previous-version installation selector: https://pytorch.org/get-started/previous-versions/
- bitsandbytes project: https://github.com/bitsandbytes-foundation/bitsandbytes

## Completion checklist

- [x] Python and Git available
- [x] CUDA-enabled PyTorch verified on RTX 3050 Laptop GPU
- [x] bitsandbytes imports and NF4 function detected
- [x] Server property bug identified and documented
- [x] Initial download/sharding confirmed to progress
- [x] Full compressed layer preparation completed (68/68)
- [x] `/v1/models` lists the configured model
- [x] Real `/v1/chat/completions` request returned a response in stream mode
- [ ] Response formatting checked for unwanted visible `</think>` marker and duplicate answer

Update this checklist only after verifying each item on the target machine.
