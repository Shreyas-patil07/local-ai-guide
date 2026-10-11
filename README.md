# Local AI Guide — Ollama + Qwen2.5-Coder on Windows

A hardware-specific guide for installing Ollama on Windows and running Qwen2.5-Coder locally on an NVIDIA RTX 3050 Laptop GPU with 4 GB VRAM.

> **Verified on 2026-10-11:** Qwen2.5-Coder 3B Instruct Q4_K_M and 7B Instruct Q3_K_S both ran successfully in Ollama. The 3B model reached **6.69 tokens/second** with 100% GPU placement; the 7B model reached **5.83 tokens/second** with a 40% CPU / 60% GPU split. These are single observed runs, not multi-run averages.

## 1. Target system

| Component | Observed value |
|---|---|
| OS | Windows 11, build 10.0.26300.9550 |
| CPU | Intel Core i5-12450H |
| RAM | 16 GB |
| GPU | NVIDIA GeForce RTX 3050 Laptop GPU |
| Dedicated VRAM | 4 GB |
| Model runtime | Ollama, llama.cpp runner |
| Model | `qwen2.5-coder:3b-instruct-q4_K_M` |
| Download size | 1.9 GB as shown by `ollama list` |
| Runtime size | 2.2 GB as shown by `ollama ps` |
| Observed context | 4096 tokens |
| Free disk after previous cleanup | 109.98 GB |

Hardware and performance details refer to this specific machine. Your results may differ depending on drivers, background applications, temperature, power mode, and prompt length.

## 2. Why this model

The selected tag is **Qwen2.5-Coder 3B Instruct, Q4_K_M**. The instruct variant is intended for conversational prompts, code explanation, and coding assistance. The Ollama registry lists this tag at about **1.9 GB** and shows a 32K context capability for the tag, but the tested active session reported an effective context of **4096** in `ollama ps`. This guide sticks to the tested runtime configuration instead of assuming a larger active context.

Model tag page: https://ollama.com/library/qwen2.5-coder/tags

Exact tag: https://ollama.com/library/qwen2.5-coder:3b-instruct-q4_K_M

## 3. Install Ollama on Windows

### Option A — Official installer (recommended)

1. Open the official Windows download page: https://ollama.com/download/windows
2. Download and run the Windows installer.
3. Complete the installation.
4. Open a **new PowerShell window** so the updated PATH is loaded.

### Option B — Install from PowerShell

Ollama also publishes this installation command on its Windows download page:

~~~powershell
irm https://ollama.com/install.ps1 | iex
~~~

Use one installation method; you do not need to run both.

### Verify the installation

~~~powershell
ollama --version
ollama list
~~~

If `ollama` is not recognized, close and reopen PowerShell. If it still fails, confirm Ollama is installed through Windows Settings → Apps → Installed apps and use the official installer page above.

Official source: https://ollama.com/download/windows

## 4. Download and run the exact model tag

Run this in PowerShell:

~~~powershell
ollama run qwen2.5-coder:3b-instruct-q4_K_M
~~~

Ollama downloads the model if it is not already cached, then opens an interactive chat. The tag used during this setup appeared in `ollama list` as:

~~~text
NAME                                ID              SIZE
qwen2.5-coder:3b-instruct-q4_K_M    f72c60cabf62    1.9 GB
~~~

Do not install every quantization variant. This guide uses the single tag above.

### First coding prompt

At the model prompt, enter:

~~~text
Write a Python function that checks whether a string is a palindrome.

Requirements:
- Use type hints.
- Ignore spaces, punctuation, and capitalization.
- Include three test cases.
- Explain time and space complexity.
~~~

To leave an interactive session, enter /bye if supported by the running CLI, or press Ctrl+D/Ctrl+C in the terminal.

## 5. Verify GPU usage

Start the model, send a prompt, and while the model is still loaded open a **second PowerShell window**:

~~~powershell
ollama ps
~~~

During the verified session, output included:

~~~text
NAME                                ID              SIZE    PROCESSOR    CONTEXT    RUNNER
qwen2.5-coder:3b-instruct-q4_K_M    f72c60cabf62    2.2 GB  100% GPU     4096       llamacpp
~~~

Interpretation:

- **100% GPU** means Ollama reported the model fully on the GPU for that session.
- **CPU/GPU split** would indicate that only part of the model is offloaded to the GPU.
- **CONTEXT 4096** is the effective context reported for this observed run.
- The UNTIL column indicates when the model will be unloaded after an idle period.

The model may unload after it has been idle. Run a prompt first, then check `ollama ps`.

## 6. Reproducible generation benchmark

This benchmark uses Ollama's local generate API. It measures the model's generation duration and total request duration separately.

Run in a second PowerShell window while the Ollama service is running:

~~~powershell
$body = @{
    model = "qwen2.5-coder:3b-instruct-q4_K_M"
    prompt = "Write a Python palindrome checker with type hints and three test cases. Keep the explanation brief."
    stream = $false
    options = @{
        temperature = 0
        num_predict = 128
    }
} | ConvertTo-Json -Depth 5

$r = Invoke-RestMethod -Uri "http://127.0.0.1:11434/api/generate" -Method Post -ContentType "application/json" -Body $body

$evalSeconds = $r.eval_duration / 1e9
$tokensPerSecond = if ($evalSeconds -gt 0) {
    [math]::Round($r.eval_count / $evalSeconds, 2)
} else { 0 }

Write-Host "`n--- Benchmark Results ---"
Write-Host "Model: $($r.model)"
Write-Host "Prompt tokens: $($r.prompt_eval_count)"
Write-Host "Generated tokens: $($r.eval_count)"
Write-Host "Generation time: $([math]::Round($evalSeconds, 2)) sec"
Write-Host "Generation speed: $tokensPerSecond tokens/sec"
Write-Host "Total request time: $([math]::Round($r.total_duration / 1e9, 2)) sec"
Write-Host "`n--- Generated Response ---"
Write-Host $r.response
~~~

The numbers depend on the prompt, token cap, and system state. This is a simple smoke benchmark, not a standardized performance test. The script deliberately caps output at 128 tokens, so an answer can stop partway through a code block. Increase `num_predict` for a longer answer, understanding that it will take longer.

### Measured result on this laptop

The following values were observed on 2026-10-11 for the prompt above:

| Metric | Observed result |
|---|---:|
| Model | `qwen2.5-coder:3b-instruct-q4_K_M` |
| Generated tokens | 128 |
| Generation duration (`eval_duration`) | 19.14 seconds |
| Generation rate | 6.69 tokens/second |
| Total request duration | 39.88 seconds |
| GPU placement from `ollama ps` | 100% GPU |
| Runtime context from `ollama ps` | 4096 |

Calculation: 128 tokens ÷ 19.14 seconds = approximately **6.69 tokens/second**. Total request time includes prompt processing and other request overhead, so it is not the same as generation duration.

The response shown in the terminal stopped partway through a Python code example at the 128-token cap. This is expected for a benchmark that intentionally limits output; it does not by itself indicate an Ollama failure.

### Second model tested: Qwen2.5-Coder 7B Instruct Q3_K_S

A second run was tested on the same laptop using this model tag:

~~~powershell
ollama pull qwen2.5-coder:7b-instruct-q3_K_S
ollama run qwen2.5-coder:7b-instruct-q3_K_S
~~~

During the observed session, `ollama ps` reported:

~~~text
NAME                                ID              SIZE    PROCESSOR          CONTEXT    RUNNER
qwen2.5-coder:7b-instruct-q3_K_S    748645786a08    3.9 GB  40%/60% CPU/GPU    4096       llamacpp
~~~

The same 128-token generation benchmark prompt was then run with `num_predict = 128`. Observed results:

| Metric | Observed result |
|---|---:|
| Model | `qwen2.5-coder:7b-instruct-q3_K_S` |
| Generated tokens | 128 |
| Generation duration | 21.94 seconds |
| Generation rate | 5.83 tokens/second |
| Total request duration | 28.65 seconds |
| GPU placement from `ollama ps` | 60% GPU / 40% CPU |
| Runtime context from `ollama ps` | 4096 |

### Side-by-side results

| Metric | 3B Q4_K_M | 7B Q3_K_S |
|---|---:|---:|
| Download size in Ollama | 1.9 GB | 3.5 GB listed tag size; 3.9 GB shown as loaded runtime size |
| Generated tokens | 128 | 128 |
| Generation duration | 19.14 s | 21.94 s |
| Generation speed | 6.69 tok/s | 5.83 tok/s |
| Total request duration | 39.88 s | 28.65 s |
| Processor placement | 100% GPU | 40% CPU / 60% GPU |

In these single runs, the 7B model generated tokens about **13% slower** than 3B. Its reported total request duration was shorter, despite the slower generation rate; total duration includes prompt processing and other overhead, so do not treat total latency from one run as a stable winner. Neither result is a multi-run benchmark, and the 128-token cap truncated the generated code examples. Run each model several times with the same prompt and settings before making a final choice.

**Practical takeaway:** the 3B model currently offers the higher measured generation rate and fits entirely on the GPU. The 7B model is viable to test for coding quality, but it requires CPU offloading on this 4 GB VRAM laptop. Whether the 7B model is the better daily driver depends on code correctness and output quality, not model size alone.

## 7. Useful everyday commands

~~~powershell
# Show installed models
ollama list

# Check models currently loaded in memory
ollama ps

# Run the chosen model
ollama run qwen2.5-coder:3b-instruct-q4_K_M

# Show model details
ollama show qwen2.5-coder:3b-instruct-q4_K_M

# Remove this model's downloaded files
ollama rm qwen2.5-coder:3b-instruct-q4_K_M
~~~

Do not run the remove command unless you intend to delete the downloaded model.

## 8. Local API

Ollama's local API is served at **http://127.0.0.1:11434** on the tested setup.

Simple health check (the root response only confirms the HTTP service is reachable):

~~~powershell
Invoke-WebRequest "http://127.0.0.1:11434"
~~~

Generate a single response from PowerShell:

~~~powershell
$body = @{
    model = "qwen2.5-coder:3b-instruct-q4_K_M"
    prompt = "Write a Python function to add two integers."
    stream = $false
} | ConvertTo-Json -Depth 5

$response = Invoke-RestMethod -Uri "http://127.0.0.1:11434/api/generate" -Method Post -ContentType "application/json" -Body $body
$response.response
~~~

The API example uses the local service and does not require an API key for a standard local request. Keep the service local unless you intentionally configure and secure remote access. Do not expose an unauthenticated local model API to untrusted networks.

## 9. Troubleshooting

| Symptom | What to check |
|---|---|
| `ollama` is not recognized | Reopen PowerShell after installation; verify Ollama is installed in Windows Settings; rerun the official installer if needed. |
| Model download fails | Check internet connectivity and free disk space, then rerun `ollama run qwen2.5-coder:3b-instruct-q4_K_M`. |
| Model does not appear in `ollama ps` | Send a prompt first; the model may have unloaded after being idle. |
| `ollama ps` reports CPU/GPU split | Close other GPU-intensive applications and check the NVIDIA driver and available VRAM. |
| Response ends mid-code | Check the requested output cap. The benchmark uses `num_predict = 128`; increase it for longer answers. |
| Slow or variable output speed | Repeat the same prompt more than once, keep the laptop plugged in, close memory-heavy applications, and record each run. Avoid comparing one short run with a different prompt length. |
| GPU is unusually hot | Ensure laptop vents are unobstructed, use a hard surface, and reduce sustained load if temperatures remain high. |

## 10. Model storage and cleanup

Ollama stores downloaded model data in its configured models directory. On a standard Windows user installation, check the user profile's Ollama directory before manually removing data. The actual location can be changed with the `OLLAMA_MODELS` environment variable.

Prefer Ollama commands for model cleanup:

~~~powershell
ollama list
ollama rm qwen2.5-coder:3b-instruct-q4_K_M
~~~

To uninstall Ollama itself, use Windows Settings → Apps → Installed apps → Ollama → Uninstall. After stopping/uninstalling it, inspect any remaining Ollama data directory before deleting it. Removing the model or uninstalling the application is destructive and should be done only if you no longer need the downloads.

No Hugging Face cache or separate model sharding step is required for the Ollama workflow described in this guide.

## 11. Security and privacy notes

- The commands in this guide use Ollama locally on the same Windows machine.
- Model downloads require internet access; ordinary local inference uses the local Ollama process.
- The API port used in the examples is 11434. Do not forward it to the public internet or expose it to untrusted networks without an appropriate security layer.
- Do not paste secrets, API keys, private keys, or credentials into prompts.
- Only install Ollama from the official download page: https://ollama.com/download/windows

## Completion checklist

- [x] Ollama installed and CLI responds
- [x] Exact model tag downloaded: `qwen2.5-coder:3b-instruct-q4_K_M`
- [x] `ollama ps` verified 100% GPU placement on the observed run
- [x] Local API generation request succeeded
- [x] 128-token benchmark recorded at 6.69 tokens/second
- [ ] Run repeated benchmarks and record median speed
- [ ] Validate coding quality against a small set of real Python/backend tasks

## References

- Ollama Windows download: https://ollama.com/download/windows
- Qwen2.5-Coder tags: https://ollama.com/library/qwen2.5-coder/tags
- Exact selected model tag: https://ollama.com/library/qwen2.5-coder:3b-instruct-q4_K_M
- Ollama documentation: https://docs.ollama.com/
