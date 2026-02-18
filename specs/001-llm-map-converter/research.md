# Research: LLM-Assisted MAP TXT Converter

**Date**: 2026-02-18
**Feature**: 001-llm-map-converter

## Decision 1: LLM Provider & Model

**Decision**: Hugging Face Inference API with `Qwen/Qwen2.5-7B-Instruct` as default model.

**Rationale**:
- Qwen2.5 family is explicitly optimized for structured JSON output
- 7B parameter size balances quality vs. free-tier cost ($0.10/month free credits)
- 128K context length supports large file snippets
- HF Inference API supports `response_format` with JSON schema enforcement
- `huggingface_hub` InferenceClient provides clean Python API

**Alternatives considered**:
- `Qwen/Qwen3-32B`: Higher quality but consumes more credits per call
- `NousResearch/Hermes-2-Pro-Mistral-7B`: 84% JSON accuracy, specialized but smaller community
- `meta-llama/Llama-3.1-8B-Instruct`: Good alternative if Qwen unavailable
- OpenAI/Anthropic APIs: Ruled out (paid, user requested free HF models)

**Rate limits**: ~few hundred requests/hour on free tier. $0.10/month credits sufficient for 50-200 calls to 7B model. Adequate for demo use.

## Decision 2: Encoding Detection

**Decision**: `chardet` (pre-installed in Colab) as primary, with manual fallback chain (UTF-8 → CP949 → EUC-KR → Latin-1).

**Rationale**:
- chardet v5.2.0 is pre-installed in Google Colab (no pip install needed)
- Supports CP949 detection
- For this demo, a simple fallback chain handles the known encoding variants
- charset-normalizer is faster/more accurate but requires pip install

**Alternatives considered**:
- `charset-normalizer`: 10x faster, 98% accuracy, but needs pip install
- `cchardet`: Unmaintained, avoid for new projects
- Pure fallback (try/except): Works but less elegant

## Decision 3: UI Approach

**Decision**: Native Colab cells with `google.colab.files` for upload/download and IPython display for output.

**Rationale**:
- User explicitly requested "just use Colab"
- `google.colab.files.upload()` / `files.download()` are built-in
- IPython.display handles JSON pretty-printing, HTML tables
- Zero additional dependencies

**Alternatives considered**:
- Gradio: User rejected as overkill
- ipywidgets: Adds unnecessary complexity for a demo
- Panel/Streamlit: Requires server, not Colab-native

## Decision 4: JSON Schema Enforcement

**Decision**: Use Pydantic models for rule proposal schema definition + HF `response_format` parameter for strict JSON output.

**Rationale**:
- Pydantic generates JSON Schema automatically from Python classes
- HF Inference API supports `response_format: { type: "json_schema", json_schema: {...} }`
- This constrains LLM output to valid JSON matching exact schema
- Eliminates need for manual JSON parsing/validation

**Alternatives considered**:
- Manual JSON parsing with try/except: Brittle, LLM may produce invalid JSON
- Outlines/guidance libraries: Overkill for API-based inference
- Regex extraction from prose: Unreliable

## Decision 5: Project Delivery Format

**Decision**: Single `.ipynb` Jupyter notebook with modular cells.

**Rationale**:
- Colab native format
- "Run All" workflow requirement (SC-005)
- Each cell is a logical module (ingest, analyze, convert, etc.)
- No file imports needed, everything self-contained

**Alternatives considered**:
- Python package + notebook wrapper: Over-engineered for demo
- Multiple .py files with Colab `!python` calls: Breaks "Run All" flow
