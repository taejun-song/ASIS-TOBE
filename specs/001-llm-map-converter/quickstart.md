# Quickstart: LLM-Assisted MAP TXT Converter

## Prerequisites

- Google account with access to Google Colab
- Hugging Face account with API token (free tier: https://huggingface.co/settings/tokens)

## Setup

1. Open the notebook `map_converter.ipynb` in Google Colab
2. Click **Runtime → Run all**
3. When prompted, enter your Hugging Face API token

## Usage

After "Run All" completes, the last cell will:

1. **Upload**: Show a file upload widget — upload any vendor MAP TXT file
2. **Analyze**: Detect encoding, classify format, send sampled lines to LLM
3. **Convert**: Apply proposed rules, generate ASEKR standard output
4. **Display**: Show extracted metadata with confidence scores, wafer map preview, rule JSON, and validation report
5. **Download**: Automatically trigger downloads for the converted output (`.txt`) and rule JSON (`.json`)

## Interpreting the Rule JSON

The rule JSON contains these key sections:

| Section | What it tells you |
|---------|-------------------|
| `detected_format` | Encoding, delimiter, format type detected |
| `header_rules` | How many header lines to skip and why |
| `metadata_extraction` | How LotID, WaferNo, Customer etc. are extracted |
| `map_parsing` | How the wafer map data is structured |
| `bin_mapping` | Which bin codes map to which output characters |
| `confidence` | Per-field confidence scores (0.0-1.0) |
| `candidates` | Alternative mappings for uncertain fields |
| `evidence` | Source file line snippets justifying each decision |

Fields with confidence below 0.7 are flagged as uncertain. Check `candidates` for alternatives.

## Swapping LLM Provider

Edit the configuration cell to change the model:

```python
LLM_MODEL = "Qwen/Qwen2.5-7B-Instruct"  # Change this
```

Or switch to a different provider by modifying the `LLMProvider` class.

## Sample Files

The repository includes 6 sample input/output pairs for testing:

| Input | Format Type |
|-------|------------|
| `input/1ACB86_W25.TXT` | Plain grid map with single-char bins |
| `input/60XHHA.22` | Bracketed header + X/Y/B coordinates |
| `input/68ZBC3P-01.map` | BOF/EOF structured with soft bin table |
| `input/B05388.01-12.txt` | Grid map with trailing metadata |
| `input/BN1737.023` | WAFER_MAP structured format |
| `input/GP300P043.005` | WAFER_MAP structured format |
