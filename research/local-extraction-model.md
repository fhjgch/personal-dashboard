# Which local model on an 8 GB M1 can extract Readings from free-text journal entries?

Research for [#3](https://github.com/fhjgch/personal-dashboard/issues/3) (part of map [#1](https://github.com/fhjgch/personal-dashboard/issues/1)). Desk research only: no journal files were read and no models were run. Sources were checked on 2026-10-10.

## Short answer

- **Try `qwen3:4b-instruct-2507` (Q4_K_M, 2.5 GB) through Ollama first.** Ask for output with a JSON schema built from a Pydantic model (`format=Model.model_json_schema()`), set `temperature=0`, and set `num_ctx` to about 8192.
- **Runner-up: `gemma3:4b` (3.4 GB).** It has the highest published IFEval score among the small candidates and explicitly supports more than 140 languages.
- **Fast fallback: `llama3.2:3b` (2.0 GB).** It is the smallest and fastest candidate, and German is one of its officially supported languages, but its published scores are weaker.
- **Avoid 8B and larger models on this machine.** For example, `qwen3:8b` is a 5.2 GB download, which leaves too little of the 8 GB for macOS, the KV cache and other apps.
- **No published benchmark measures this exact task.** The prototype ticket should hand-label about 20 to 30 pages, run them through the top two models, and keep whichever extracts better.
- **Rough speed estimate: 10 to 40 minutes for all ~400 pages with a 4B model.** This is an inference from published M1 benchmarks, not a measurement. A cheap keyword pre-filter (for example `read`, `book`, `finished`, `Buch`, `gelesen`, or `[[...]]` links) can skip most pages.

## Candidates

| Model (Ollama tag) | Download | Context (Ollama) | Languages | Relevant published scores | Fit on 8 GB M1 |
|---|---|---|---|---|---|
| `qwen3:4b-instruct-2507` (q4_K_M) | 2.5 GB | 256K | Qwen3 family: "100+ languages and dialects" | IFEval 83.4, MultiIF 69.0, BFCL-v3 61.9, MMLU-Pro 69.6 | Good |
| `gemma3:4b` | 3.4 GB (includes the vision part) | 128K | "over 140 languages" | IFEval 90.2, MMLU-Pro 43.6, Global-MMLU-Lite 54.5 | Good |
| `llama3.2:3b` | 2.0 GB | 128K | 8 official languages, German among them | IFEval 77.4, BFCL V2 67.0, MMLU 63.4, German MMLU 53.3 | Very good |
| `phi4-mini:3.8b` | 2.5 GB | 128K | "multilingual"; no list on the Ollama page | none on the Ollama page | Good |
| `qwen3:8b` | 5.2 GB | 40K | 100+ | (not checked) | Tight; not recommended |
| `gemma3:12b` | 8.2 GB | 128K | 140+ | (not checked) | Does not fit |

Sources: Ollama library pages for [qwen3 tags](https://ollama.com/library/qwen3/tags), [gemma3](https://ollama.com/library/gemma3), [llama3.2](https://ollama.com/library/llama3.2) and [phi4-mini](https://ollama.com/library/phi4-mini). Scores come from the model cards for [Qwen3-4B-Instruct-2507](https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507), [Gemma 3](https://ai.google.dev/gemma/docs/core/model_card_3) and [Llama-3.2-3B-Instruct](https://huggingface.co/meta-llama/Llama-3.2-3B-Instruct). The Qwen language claim is from the [Qwen3-4B model card](https://huggingface.co/Qwen/Qwen3-4B).

Scores from different vendors are **not directly comparable**, because each vendor uses its own harness and IFEval averaging. Treat them only as a signal that the model follows instructions.

### Why Qwen3-4B-Instruct-2507 first

- **It does not think.** It is the non-thinking 2507 update of Qwen3-4B, which "supports only non-thinking mode and does not generate `<think></think>` blocks" ([model card](https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507)). Extraction gains nothing from reasoning tokens, and on an M1 they would cost generation time.
- **It improves on the original Qwen3-4B in non-thinking mode** on the benchmarks most relevant to extraction: IFEval rose from 81.2 to 83.4, MultiIF (multilingual instruction following) from 61.3 to 69.0, and BFCL-v3 (structured function calling) from 57.6 to 61.9 ([model card](https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507)).
- **It is small.** The Q4_K_M weights are 2.5 GB, and a q8_0 build (4.3 GB) exists if the 4-bit build proves too lossy ([tags](https://ollama.com/library/qwen3/tags)).
- **Mind the tag.** The Ollama tag list mixes 2507 builds (256K context) with the original hybrid-thinking Qwen3-4B (`qwen3:4b-q4_K_M`, 40K context), and does not say which build the plain `qwen3:4b` tag is. Pin the explicit `qwen3:4b-instruct-2507-q4_K_M` tag. If a hybrid Qwen3 model is used instead, pass `think=False` ([Ollama thinking docs](https://docs.ollama.com/capabilities/thinking)).

### Why Gemma 3 4B as runner-up

- **It has the best instruction-following score among the small candidates:** IFEval 90.2 ([Gemma 3 model card](https://ai.google.dev/gemma/docs/core/model_card_3)).
- **It has the broadest language coverage**, with "multilingual support in over 140 languages" (same model card).
- **It is about 0.9 GB larger than Qwen** (3.4 GB), because the 4B model is multimodal ([Ollama gemma3](https://ollama.com/library/gemma3)). Its MMLU-Pro is much lower (43.6 against 69.6), but world knowledge matters little for this task.

## Getting reliable structured output

**Ollama structured outputs.** Pass a JSON schema as the `format` parameter, and the output is constrained to that schema. The docs recommend:

- defining the schema with Pydantic,
- setting temperature to 0,
- and also putting the schema in the prompt: it is "ideal to also pass the JSON schema as a string in the prompt" ([docs](https://docs.ollama.com/capabilities/structured-outputs), [announcement blog, 2024-12-06](https://ollama.com/blog/structured-outputs)).

The same feature is available through the OpenAI-compatible API as `response_format`.

**Why the schema also belongs in the prompt.** In llama.cpp, which Ollama builds on, a JSON schema is converted to a GBNF grammar that constrains token sampling. The schema "is not injected into the prompt", so the model cannot see it unless you describe it ([llama.cpp grammars README](https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md)).

**Keep the schema simple.** The llama.cpp converter silently skips some features. Examples are nested `$ref`, `properties` mixed with `anyOf`/`oneOf`, `prefixItems` and `patternProperties` (same README). A flat schema fits well:

```python
from typing import Literal
from pydantic import BaseModel

class ReadingEvent(BaseModel):
    title: str                           # as written, e.g. "The Pragmatic Programmer"
    event: Literal["started", "finished"]
    evidence: str                        # the exact sentence it came from

class PageExtraction(BaseModel):
    events: list[ReadingEvent]           # empty list when nothing book-related
```

Usage: `ollama.chat(model=..., messages=..., format=PageExtraction.model_json_schema(), options={"temperature": 0, "num_ctx": 8192})`, then `PageExtraction.model_validate_json(resp.message.content)`.

Design notes (my own inferences, not from a source):

- **Take the date from the journal page filename, not from the model.** This keeps week assignment deterministic.
- **The `evidence` field makes spot-checking cheap.** It also discourages invented events: drop any event whose evidence string is not a substring of the page.
- **Constrained decoding guarantees valid JSON, not correct content.** It is the hand-labeled sample in the prototype that measures quality.

## Speed for ~400 pages

**Published baseline.** The llama.cpp Apple Silicon benchmark gives these figures for a base M1 (68 GB/s, 7 or 8 GPU cores) running LLaMA 7B at Q4_0 ([llama.cpp discussion #4167](https://github.com/ggml-org/llama.cpp/discussions/4167)):

- about 108–118 tokens/s for prompt processing (pp512),
- about 14 tokens/s for generation (tg128).

**Estimate for a 4B model (my inference).** Generation speed is memory-bandwidth-bound, so a model about 4/7 the size should run roughly 1.5 to 1.8 times faster: about 180 tokens/s of prompt processing and 20 to 25 tokens/s of generation. A page of about 800 prompt tokens (instructions, schema and journal text) with about 60 output tokens would then take about 5 to 7 seconds. That is about 30 to 45 minutes for 386 pages if every page goes to the model.

**Ways to cut the time:**

- **Pre-filter pages by keyword.** Only pages that mention reading or books go to the model.
- **Keep the model loaded.** Set `keep_alive` so the model is not unloaded between calls; the default unloads it after 5 minutes idle ([Ollama FAQ](https://docs.ollama.com/faq)).
- **Cache results by page hash.** Re-runs then only process pages that changed.

**Use the GPU backend.** Ollama on Apple Silicon runs on CPU and GPU, needs macOS 14 Sonoma or newer, and is not GPU-accelerated inside Docker on macOS ([Ollama macOS docs](https://docs.ollama.com/macos), [FAQ](https://docs.ollama.com/faq)). Install the native app, not a container.

## Memory use

| Item | Approx. size | Source / basis |
|---|---|---|
| Qwen3-4B-Instruct-2507 weights (Q4_K_M) | 2.5 GB | [Ollama tags](https://ollama.com/library/qwen3/tags) |
| KV cache at 8192 tokens, f16 | ~1.2 GB | My calculation from the model card architecture: 36 layers, 8 KV heads, head dim 128. That gives 2 × 36 × 8 × 128 × 2 bytes ≈ 147 KB per token. |
| Same cache with `OLLAMA_KV_CACHE_TYPE=q8_0` | ~0.6 GB | Per the [FAQ](https://docs.ollama.com/faq), q8_0 is "about half" of f16 with "very small" quality loss and needs Flash Attention, which is used automatically when supported. |

That is about 3.1 to 3.7 GB in total, which leaves room for macOS on 8 GB. Further memory notes:

- **Set `num_ctx` explicitly.** Ollama's default context is 4096 tokens ([FAQ](https://docs.ollama.com/faq)). A long journal page plus instructions could be truncated at that size, so set about 8192 rather than the model's 256K maximum.
- **Process pages one at a time.** Required RAM scales with `OLLAMA_NUM_PARALLEL × OLLAMA_CONTEXT_LENGTH` ([FAQ](https://docs.ollama.com/faq)), so leave parallelism at 1 on this machine.

## Alternatives to Ollama (considered, not recommended first)

- **llama.cpp `llama-server`.** It runs the same GGUF models and accepts a schema through `json_schema` or `response_format` ([grammars README](https://github.com/ggml-org/llama.cpp/blob/master/grammars/README.md)). It gives more control but means more setup.
- **MLX / `mlx-lm`.** It is Apple-native and often fast. Its README does not document JSON-schema-constrained output, only generic `logits_processors`. It also warns that models large relative to RAM "can be slow", and wiring memory requires macOS 15+ ([mlx-lm README](https://github.com/ml-explore/mlx-lm)). Structured output would need an extra library, so MLX is not the first choice for this prototype.

## Open questions for the prototype ticket (#4)

- **Extraction accuracy on real pages**, especially on implicit mentions such as "[[Book on Kundalini]] — Just finished the book", where the title is only a page link.
- **German handling**, if German entries exist. Compare Qwen and Gemma on any German pages in the labeled sample.
- **Actual tokens per second on this M1.** Running `ollama run <model> --verbose` prints eval rates.
