# Local AI Guide — Windows + AirLLM + Qwen3.8-27B

A reproducible, hardware-specific guide for running **Qwen3.8-27B through AirLLM** on Windows with an NVIDIA RTX 3050 Laptop GPU (4 GB VRAM).

> **Status:** CUDA and bitsandbytes imports were verified, and the captured model-preparation log reached layer 30/68 (44%). A full end-to-end inference response has not yet been confirmed. Use the verification checklist before considering the setup complete.

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
| Context configuration | 4096 tokens |
| Compression configuration | 4-bit |

## Important reality check

AirLLM streams model layers from storage; it does not turn this model into a small, fast local model. This 27B model is very demanding on a laptop with 16 GB RAM and 4 GB VRAM. Expect slow initial preparation, disk activity, memory pressure, and potentially low generation throughput. Keep sufficient disk space available.

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

## 7. Start the server and prepare the model

Use one CMD window for the server:

```bat
cd /d C:\AI\AirLLM-Server
set "HF_HOME=C:\hf"
set "AIRLLM_MODEL=Qwen/Qwen3.8-27B"
set "AIRLLM_DEVICE=cuda"
set "AIRLLM_COMPRESSION=4bit"
set "AIRLLM_MAX_CONTEXT=4096"
run.bat
```

The server log should report CUDA-enabled PyTorch, the NVIDIA GPU, the selected model, compression, and context.

The first run downloads the source checkpoint and prepares 4-bit layer files beneath a path similar to:

```text
C:\hf\hub\models--Qwen--Qwen3.8-27B\snapshots\<snapshot>\splitted_model.4bit\
```

The captured log showed 68 layer/file preparation steps with some individual shard downloads reaching several gigabytes. At the latest captured progress, it had reached **30/68 (44%)** and was loading shard 9 of 18. **This is only a progress snapshot; completion has not yet been confirmed.**

- Keep the server terminal open.
- Keep the laptop plugged in and connected to the internet.
- Close RAM-intensive apps.
- Monitor disk space and Windows memory usage.
- If C: becomes critically low (for example, less than 15 GB free), stop intentionally with Ctrl+C rather than risk filling the system drive.
- The Hugging Face unauthenticated warning is not by itself fatal. A Hugging Face token can help with rate limits; never commit a token to this repository.

Check remaining C: space from a second CMD window:

```bat
powershell -NoProfile -Command "$d=Get-PSDrive C; 'C: free space: {0:N1} GB' -f ($d.Free/1GB)"
```

## 8. Test the API

Leave the server running and open a second CMD window.

Health endpoint:

```bat
curl.exe http://127.0.0.1:8000/health
```

List available models:

```bat
curl.exe http://127.0.0.1:8000/v1/models
```

A health response of HTTP 200 only means the web server is alive. The model loads in a background thread, so check the server terminal for model-ready logs and any traceback. Then make a real completion request:

```bat
curl.exe -X POST http://127.0.0.1:8000/v1/chat/completions -H "Content-Type: application/json" -d "{\"model\":\"Qwen/Qwen3.8-27B\",\"messages\":[{\"role\":\"user\",\"content\":\"Reply with exactly: AirLLM is working.\"}],\"max_tokens\":40}"
```

Success means the endpoint returns JSON containing a non-empty assistant message. Record response latency and check RAM/VRAM. If it fails, save the traceback and latest server output.

## 9. Known problems and fixes

| Symptom | Cause observed | Fix |
|---|---|---|
| `install.bat` not recognized | CMD was in `C:\AI`, not the cloned repository; a `cd` and `mkdir` command had been concatenated | Run `cd /d C:\AI\AirLLM-Server`, verify with `dir install.bat` |
| `PyTorch: 2.14.1+cpu`, CUDA false | Installer selected CPU-only torch on this machine | Install pinned PyTorch/vision/audio 2.6.0 CUDA 12.4 wheels |
| `libbitsandbytes_cuda116.dll` missing | Incompatible `bitsandbytes-windows 0.37.5` package | Remove both variants and install `bitsandbytes==0.43.3`; verify imports |
| `total_mem` attribute error | Server used incorrect PyTorch CUDA property | Change `total_mem` to `total_memory` in `server.py` |
| `/health` returns 200 but inference does not work | API server can start while model loading crashes in a background thread | Inspect model-load logs, check models endpoint, and run a real completion |
| Slow first launch | Checkpoint download and layer quantization/sharding are I/O intensive | Leave process running; monitor progress, internet, RAM, and disk |

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
Max context:            4096
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
- [ ] Full model preparation completed without traceback
- [ ] `/v1/models` confirms the configured model is ready
- [ ] Real `/v1/chat/completions` request returns a response

Update this checklist only after verifying each item on the target machine.
