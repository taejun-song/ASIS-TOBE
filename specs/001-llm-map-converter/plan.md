# Implementation Plan: LLM-Assisted MAP TXT Converter

**Branch**: `001-llm-map-converter` | **Date**: 2026-02-18 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-llm-map-converter/spec.md`

## Summary

Build a Google Colab notebook that accepts vendor MAP TXT files (6 known formats), uses a free Hugging Face LLM to propose transformation rules as structured JSON, applies those rules deterministically via Python, and generates standardized ASEKR output files. The notebook uses native Colab UI (no Gradio) and `Qwen/Qwen2.5-7B-Instruct` via HF Inference API for rule generation.

## Technical Context

**Language/Version**: Python 3.10+ (Google Colab default)
**Primary Dependencies**: `huggingface_hub` (LLM API), `chardet` (encoding detection, pre-installed), `pydantic` (JSON schema), `pandas` (data manipulation)
**Storage**: Colab filesystem (ephemeral, file-based)
**Testing**: Manual validation against 6 sample input/output pairs
**Target Platform**: Google Colab (browser-based Jupyter environment)
**Project Type**: Single Jupyter notebook (.ipynb)
**Performance Goals**: End-to-end conversion within 60 seconds per file (including LLM API call)
**Constraints**: Free-tier HF API (~$0.10/month credits, ~few hundred requests/hour), Colab memory limits
**Scale/Scope**: Single-file-at-a-time demo, 6 sample formats

## Constitution Check

*No constitution file found. No gates to evaluate.*

## Project Structure

### Documentation (this feature)

```text
specs/001-llm-map-converter/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0: Technology research
├── data-model.md        # Phase 1: Data model
├── quickstart.md        # Phase 1: Usage guide
├── contracts/
│   ├── rule-proposal-schema.json  # Rule JSON schema
│   └── asekr-output-format.md     # Output format spec
└── checklists/
    └── requirements.md  # Spec quality checklist
```

### Source Code (repository root)

```text
map_converter.ipynb      # Main Colab notebook (single deliverable)
input/                   # Sample vendor MAP files (6 files)
output/                  # Expected ASEKR standard outputs (6 files)
```

**Structure Decision**: Single notebook architecture. All code lives in `map_converter.ipynb` organized into logical cell groups. No separate Python modules — this keeps the "Run All" workflow simple and avoids import complexity in Colab.

### Notebook Cell Architecture

```text
map_converter.ipynb
├── Cell 1: Setup & Dependencies
│   └── pip install, imports, HF token input
├── Cell 2: Configuration
│   └── Model ID, confidence threshold, sample line count (N)
├── Cell 3: Encoding Detection & File Reader
│   └── chardet-based detection, fallback chain, read_file()
├── Cell 4: Heuristic Analyzer
│   └── format_type classification, delimiter detection, header boundary
├── Cell 5: LLM Provider Abstraction
│   └── LLMProvider base class, HuggingFaceProvider implementation
├── Cell 6: Rule Generator
│   └── Prompt construction, sampled lines assembly, LLM call, JSON parse
├── Cell 7: Rule Applier
│   └── Metadata extraction, grid construction, bin code translation
├── Cell 8: ASEKR Output Generator
│   └── Header formatting, padding, grid assembly, file writing
├── Cell 9: Validator
│   └── Field completeness check, die count comparison, report generation
├── Cell 10: Main Workflow
│   └── Upload → Analyze → Convert → Validate → Download
```

## Key Design Decisions

### 1. Heuristic Pre-Analysis Before LLM

The system classifies input files into format types BEFORE calling the LLM:
- **grid_map**: Lines of single characters (`.`, `1`, `X`, etc.) forming a wafer shape → 1ACB86, B05388
- **coordinate_xy_b**: Lines matching `X= NNNN Y= NNNN B= NNNN` → 60XHHA
- **structured_kv**: Key-value pairs with `=` or `:` delimiters → 68ZBC3P, BN1737, GP300P043

This pre-classification helps construct a more targeted LLM prompt and reduces hallucination.

### 2. Sampled Lines Strategy

Only send to LLM (configurable N, default 15 per section):
- Top N lines (captures headers/metadata)
- Middle N lines (captures data patterns)
- Bottom N lines (captures footers/trailers)

Total ~45 lines sent to LLM instead of full file (which can be 1000+ lines).

### 3. LLM Provider Swappability

Abstract `LLMProvider` class with single method `generate_rules(prompt, schema) -> dict`. Default implementation uses HF Inference API. Swap by implementing the same interface for OpenAI, Anthropic, etc.

### 4. Deterministic Rule Application

LLM ONLY proposes rules (JSON). All actual data transformation is done by Python code:
- Regex-based metadata extraction
- Line-by-line grid construction
- Lookup-table bin code translation
- Template-based ASEKR output generation

### 5. Coordinate-to-Grid Conversion

For coordinate-based formats (60XHHA), the rule applier must:
1. Parse all X/Y/B records
2. Determine grid dimensions from min/max X and Y
3. Build a 2D character grid
4. Map B codes to output characters per bin_mapping rules
5. Fill empty positions with `.`

### 6. Missing Metadata Handling

When required ASEKR fields cannot be extracted:
1. Try filename pattern matching (e.g., `1ACB86_W25` → lot=`1ACB86`, wafer=`W25`)
2. Try LLM inference from context
3. Output `[UNKNOWN - please fill in]` as placeholder

## Input Format Analysis (from samples)

| File | Format | Header | Data | Metadata Location |
|------|--------|--------|------|-------------------|
| 1ACB86_W25.TXT | grid_map | 1 line ("Passed qty:") | Character grid | Filename only |
| 60XHHA.22 | coordinate_xy_b | 26 lines (bracketed `[...]`) | X/Y/B records | Header lines 1-26 |
| 68ZBC3P-01.map | structured_kv | `[BOF]` to `[SOFT BIN MAP]` | Multi-char grid | Header key-value pairs |
| B05388.01-12.txt | grid_map | None (grid first) | Character grid | Trailing lines 128-137 |
| BN1737.023 | structured_kv | `WAFER_MAP = {` block | Character grid in `MAP = {}` | Header key-value pairs |
| GP300P043.005 | structured_kv | `WAFER_MAP = {` block | Character grid in `MAP = {}` | Header key-value pairs |
