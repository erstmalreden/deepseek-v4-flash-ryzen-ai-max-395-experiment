# DeepSeek-V4-Flash on Ryzen AI Max+ 395: a technically successful local run that I still rolled back

**English** | [Deutsch](README.de.md)

Experiment date: August 14, 2026  
Report reconstructed from the retained server logs, API responses, request captures and benchmark outputs on August 15, 2026.

> Historical report: this setup has been completely removed. No model files, server process or OpenCode integration from this experiment remain on the machine. This report contains no usernames, private paths or credentials.

## TL;DR

I ran the full **284.33B-parameter DeepSeek-V4-Flash** model locally on a GMKtec EVO-X2 with a Ryzen AI Max+ 395, Radeon 8060S and 128 GB unified memory. The selected Unsloth `UD-IQ2_M` GGUF occupied 84.68 GiB across three shards and ran through a custom llama.cpp ROCm/ROCDXG build with full GPU offload.

The result was technically real:

- **11.36 generation tok/s** in a short API test
- **8.66 generation tok/s** after a 10,415-token prompt
- correct `/v1/models` metadata
- multi-turn chat, tool selection, sequential tools and tool-error recovery passed
- OpenCode was captured sending `reasoning_effort=max`, `temperature=1.0`, `top_p=0.95`
- the embedded DeepSeek Jinja/DSML tool template was actually used

But it was not a convincing daily coding agent on this machine. The model consumed about 88.33 GiB of the unified-memory GPU pool, leaving roughly 24.89 GiB for Windows, the browser and OpenCode. A large agent request with about 8,000 prompt tokens needed roughly 162.5 seconds just for prompt evaluation. The complete OpenCode-to-model run exposing 48 tools was aborted while it was still preprocessing. The browser/MCP stack was subsequently validated directly, not by pretending that the full model-driven agent loop had completed.

So the honest conclusion is: **the full model ran and its API/tool primitives worked, but the complete local-agent experience was too slow, memory-heavy and unreliable for my use. I removed it and freed about 90 GB.**

## Test system

- Mini PC: GMKtec EVO-X2
- APU: AMD Ryzen AI Max+ 395
- GPU: Radeon 8060S, `gfx1151`
- Memory: 128 GB unified memory
- Host: Windows 11
- Runtime environment: WSL Ubuntu
- GPU backend: ROCm/HIP 7.2.4 through AMD ROCDXG
- Offload: all model layers on the GPU backend

A Vulkan llama.cpp build was also tested, but WSL exposed only `llvmpipe`, not the Radeon GPU. ROCm/ROCDXG was therefore the viable backend in this specific WSL setup.

## What model actually ran

The requested API name was `deepseek-v4-flash-0731-local`, but that was only a local alias. The available model repository used in the experiment was:

- GGUF repository: `unsloth/DeepSeek-V4-Flash-GGUF`
- Base model: `deepseek-ai/DeepSeek-V4-Flash`
- `general.name`: `Deepseek-V4-Flash`
- `general.basename`: `Deepseek-V4-Flash`
- `general.base_model.0.name`: `DeepSeek V4 Flash`
- `general.architecture`: `deepseek4`
- Size label: `256x8.4B`
- Parameters reported by llama.cpp: **284,334,567,511**
- Quantization: Unsloth **UD-IQ2_M**, reported at runtime as `IQ2_M - 2.7 bpw`
- GGUF `general.file_type`: `29`

The three loaded shards were:

| File | Bytes |
|---|---:|
| `DeepSeek-V4-Flash-UD-IQ2_M-00001-of-00003.gguf` | 5,256,864 |
| `DeepSeek-V4-Flash-UD-IQ2_M-00002-of-00003.gguf` | 49,956,780,160 |
| `DeepSeek-V4-Flash-UD-IQ2_M-00003-of-00003.gguf` | 40,964,890,464 |
| **Total** | **90,926,927,488 bytes / 84.68 GiB** |

The tiny first shard mostly carried metadata; the full split set was loaded (`split.count=3`).

## llama.cpp runtime

- Version string: `0.1.0-dev`
- Build: `1`
- API build ID: `b1-4c1a0af`
- Commit: [`4c1a0af40d88c7fbb3b15c85bf2e8016d1d5b64c`](https://github.com/ggml-org/llama.cpp/commit/4c1a0af40d88c7fbb3b15c85bf2e8016d1d5b64c)
- Commit subject: `llama : allow virtual igpu devices (#26953)`

The relevant server configuration was:

```text
--model /models/DeepSeek-V4-Flash-UD-IQ2_M-00001-of-00003.gguf
--alias deepseek-v4-flash-0731-local
--host 127.0.0.1 --port 8080
--ctx-size 65536 --parallel 1
--gpu-layers all --flash-attn on
--cache-type-k q4_0 --cache-type-v q4_0
--batch-size 512 --ubatch-size 128
--threads 16 --threads-batch 32
--load-mode dio
--jinja --reasoning-format deepseek
--cache-prompt --metrics --slots
--timeout 3600 --threads-http 4
```

The active context was 65,536 tokens. The model metadata advertised a native/training context of 1,048,576 tokens.

## Measured performance

These are retained llama.cpp/API measurements, not estimates.

| Test | Prompt | Output | Prompt processing | Generation | Timing |
|---|---:|---:|---:|---:|---:|
| Short reasoning/API test | 391 tokens | 124 tokens | 74.37 tok/s | **11.36 tok/s** | 5.37 s TTFT; 16.08 s total |
| Long-context retrieval | 10,415 tokens | 155 tokens | 52.85 tok/s | **8.66 tok/s** | 197.05 s prompt; 17.79 s generation; 214.84 s total |

The long-context test successfully recovered the two hidden markers `NORTHSTAR-173` and `SOUTHPORT-741`. An earlier attempt truncated the answer before the complete marker; the repeated test passed.

One realistic tool-heavy request contained about 8,023 prompt tokens. Prompt evaluation alone took approximately **162.51 seconds**, or **49.37 tok/s**, before the model could start answering. That cold-start latency mattered more in practice than the headline generation rate.

## Memory use

- llama.cpp GPU/unified-memory pool: approximately **90,453 MiB / 88.33 GiB**
- Total system memory in use during the loaded run: approximately **98.76 GiB**
- Remaining memory for Windows, OpenCode, browser and other tools: approximately **24.89 GiB**

The server process RSS was only about 1.45 GiB because most model allocation lived in the GPU/shared-memory pool. RSS alone was therefore a misleading measure of the real footprint.

## `/v1/models` response

The retained response identified the local alias and reported:

```json
{
  "id": "deepseek-v4-flash-0731-local",
  "owned_by": "llamacpp",
  "meta": {
    "vocab_type": 2,
    "n_vocab": 129280,
    "n_ctx": 65536,
    "n_ctx_train": 1048576,
    "n_embd": 4096,
    "n_params": 284334567511,
    "size": 90921582940,
    "ftype": "IQ2_M - 2.7 bpw"
  }
}
```

## Chat template, reasoning and tool encoding

The server used the Jinja template embedded by Unsloth in the GGUF, not a generic ChatML fallback:

- tokenizer model: `gpt2`
- pre-tokenizer: `joyai-llm`
- llama.cpp chat format: `peg-native`
- reasoning format: `deepseek`
- generation prefix: `<｜Assistant｜><think>`
- role markers: `<｜User｜>` and `<｜Assistant｜>`
- tool-call encoding: `｜DSML｜tool_calls`, `｜DSML｜invoke`, `｜DSML｜parameter`
- BOS: `<｜begin▁of▁sentence｜>`
- EOS: `<｜end▁of▁sentence｜>`

The template handled parallel tool calls and object arguments. The API tests passed:

- system prompt plus multi-turn chat
- tool selection with valid JSON arguments
- sequential `find_project_files` then `read_file` continuation
- recovery from a failed `lookup_ticket` tool by calling `search_ticket_archive`

## What OpenCode actually sent

A captured OpenCode request, rather than the model's own description of itself, showed:

```text
reasoning_effort = max
temperature      = 1.0
top_p            = 0.95
max_tokens       = 8192
```

The llama.cpp slot confirmed `reasoning_format=deepseek`, and generation began with `<think>`.

This is worth stating because the model later claimed in chat that it used no explicit chain-of-thought/reasoning mode. That self-report was false; the request and server capture proved that MAX reasoning was active.

There was also an important configuration limitation. The official DeepSeek model card recommends `temperature=1.0`, `top_p=1.0` for local deployment and at least 384K context for Think Max. This experiment used `top_p=0.95` and only 64K context because of the OpenCode configuration and the machine's practical memory headroom. Therefore this was a real MAX request, but not the officially recommended full-context Think Max environment.

## Browser and vision experiment: what passed, and what did not

The follow-up setup connected Playwright MCP 0.0.79 and a small, separate `qwen3-vl:4b` Ollama model for screenshot understanding. A previously considered 31B vision helper was rejected as wasteful for this machine and was not used.

Direct end-to-end checks of the connected tools passed:

- page navigation and screenshots
- local screenshot analysis
- button and slider interaction
- DOM inspection
- network `/health` request
- console inspection
- desktop and mobile viewport checks
- before/after visual comparison
- no horizontal mobile overflow

But the crucial distinction is this: **the complete DeepSeek → OpenCode → 48 exposed tools → browser loop did not finish.** It was aborted after several minutes of prompt preprocessing. The MCP and vision chain was then tested directly. That proves the tool stack itself worked; it does not prove that DeepSeek successfully drove the entire browser-agent loop through OpenCode.

## Why I rolled it back

The setup passed low-level tests but disappointed in the workflow that mattered:

1. Loading the full model consumed most of the machine's unified memory.
2. Cold evaluation of large OpenCode tool schemas took minutes.
3. First/MAX responses often appeared to hang while the model was still preprocessing.
4. The model inaccurately described its own active reasoning mode.
5. During a Three.js landing-page task, it proposed splitting the file into PowerShell write operations because the file was supposedly “too large for a single write call.” This was not a real platform limit and was a poor agent decision. A file was created, but the workflow was confusing and did not produce a convincing result.
6. The complete 48-tool OpenCode browser run was never demonstrated end to end.

I therefore stopped the server and removed the three GGUF shards, the custom llama.cpp build, the WSL project directory, the DeepSeek OpenCode provider/agent configuration, the temporary Playwright/vision integration and the small vision helper. Port 8080 was freed and approximately 90 GB of storage was recovered. Unrelated projects and pre-existing models were preserved.

## Comparison with the later Qwen3.8-27B run

The later [Qwen3.8-27B Ryzen AI Max+ 395 report](https://github.com/erstmalreden/qwen3.8-27b-ryzen-ai-max-395-benchmarks) measured 16.68 tok/s at 64K and 16.45 tok/s at 128K with a Q5 model occupying a much smaller process working set.

Compared with DeepSeek's 11.36 tok/s short test and 8.66 tok/s after a 10.4K prompt, Qwen was roughly 1.47× faster in the short comparison and 1.90× faster in the long-prompt comparison. These are useful practical reference points, not an apples-to-apples model benchmark: prompts, quantizations, runtimes and memory counters differed.

## Lessons for similar hardware

- A 128 GB Strix Halo system can genuinely load and run this 284B model at IQ2-class quantization.
- “It loads” and “the API passes tool tests” are much lower bars than “it is a pleasant daily coding agent.”
- Measure cold tool-schema prefill, not just generation tok/s.
- Verify reasoning parameters from the outgoing request/server logs; do not ask the model to report its own runtime configuration.
- Treat Windows process RSS and the GPU/shared pool separately on unified-memory systems.
- Keep the vision helper small when the main model already consumes most memory.
- Do not label a direct MCP test as a successful model-driven agent test.
- On this machine, a smaller, better-quantized model delivered the better overall agent experience.

## Primary references

- [DeepSeek-V4-Flash official model card](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash)
- [Unsloth DeepSeek-V4-Flash GGUF repository](https://huggingface.co/unsloth/DeepSeek-V4-Flash-GGUF)
- [Unsloth discussion about llama.cpp prompt-cache compatibility](https://huggingface.co/unsloth/DeepSeek-V4-Flash-GGUF/discussions/6)
- [AMD ROCm/ROCDXG WSL installation guide](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/docs/install/installryz/wsl/howto_wsl.html)
- [Exact llama.cpp commit used in the experiment](https://github.com/ggml-org/llama.cpp/commit/4c1a0af40d88c7fbb3b15c85bf2e8016d1d5b64c)

All results came from one machine and one software snapshot. They are documented observations, not universal performance claims.
