# Data Model: LLM-Assisted MAP TXT Converter

**Date**: 2026-02-18
**Feature**: 001-llm-map-converter

## Entity: HeuristicAnalysis

Pre-LLM analysis result from deterministic heuristic code.

| Field | Type | Description |
|-------|------|-------------|
| encoding | string | Detected file encoding (e.g., "utf-8", "cp949") |
| total_lines | int | Total number of lines in the file |
| format_type | enum | One of: "grid_map", "coordinate_xy_b", "structured_kv", "unknown" |
| likely_delimiter | string or null | Detected delimiter character (tab, comma, space, or null for fixed-width/grid) |
| header_end_line | int | Estimated line number where header ends and data begins |
| sample_top | list[string] | First N lines of the file |
| sample_middle | list[string] | N lines from the middle of the file |
| sample_bottom | list[string] | Last N lines of the file |

## Entity: RuleProposal

LLM-generated transformation rules (strict JSON schema).

| Field | Type | Description |
|-------|------|-------------|
| detected_format.encoding | string | Confirmed encoding |
| detected_format.delimiter | string or null | Confirmed delimiter |
| detected_format.fixed_width | bool | Whether the format uses fixed-width columns |
| detected_format.format_type | string | Description of the format structure |
| header_rules.lines_to_skip | int | Number of header lines to skip before data |
| header_rules.detection_reason | string | Why these lines are header |
| metadata_extraction.lot_id | FieldExtraction | How to extract LotID |
| metadata_extraction.wafer_no | FieldExtraction | How to extract WaferNo |
| metadata_extraction.customer | FieldExtraction or null | How to extract Customer (null if not found) |
| metadata_extraction.supplier | FieldExtraction or null | How to extract Supplier |
| metadata_extraction.wafer_id | FieldExtraction or null | How to extract Wafer ID |
| metadata_extraction.notch | FieldExtraction or null | How to extract Notch direction |
| map_parsing.type | enum | "grid_character", "coordinate_records", "structured_grid" |
| map_parsing.grid_start_line | int or null | Line where grid map starts |
| map_parsing.coordinate_pattern | string or null | Regex for X/Y/B coordinate lines |
| bin_mapping | list[BinMapping] | Per-vendor bin code to output character mapping |
| confidence | dict[string, float] | Per-field confidence scores (0.0-1.0) |
| candidates | dict[string, list[FieldExtraction]] | Alternative mappings for uncertain fields |
| evidence | list[EvidenceSnippet] | Line snippets used as justification |

### Sub-entity: FieldExtraction

| Field | Type | Description |
|-------|------|-------------|
| source | string | Where the field comes from (e.g., "header_line_3", "filename", "coordinate_field") |
| pattern | string | Regex or key name to extract the value |
| example_value | string | Example extracted value from the file |

### Sub-entity: BinMapping

| Field | Type | Description |
|-------|------|-------------|
| source_code | string | Original bin code in input (e.g., "0001", "1", "Y") |
| output_char | string | Character to use in ASEKR output map (e.g., "1", "X", "Y") |
| meaning | string | Semantic meaning (e.g., "pass", "fail", "edge", "reference") |
| is_pass | bool | Whether this bin code represents a passing die |

### Sub-entity: EvidenceSnippet

| Field | Type | Description |
|-------|------|-------------|
| line_number | int | Line number in the source file |
| content | string | The actual line content |
| relevance | string | What this line demonstrates |

## Entity: ASEKROutput

The standard output format structure.

| Field | Type | Description |
|-------|------|-------------|
| version | string | Always "001" |
| customer | string | Customer name or "[UNKNOWN - please fill in]" |
| supplier | string | Supplier name or "N/A" |
| wafer_id | string | Wafer identifier |
| wafer_lot | string | Lot identifier |
| wafer_no | string | Wafer number |
| row_y | int | Number of rows in wafer map grid |
| column_x | int | Number of columns in wafer map grid |
| notch | string | Notch direction or blank |
| good_bin_code | string | Code(s) representing passing bins |
| good_bin_count | int | Count of passing dies |
| total_good_qty | int | Total good quantity |
| map_grid | list[string] | The wafer map grid lines (characters: 1, X, ., etc.) |

**Output file format**: Plain text with fixed structure:
- Lines 1-12: Header key-value pairs
- Lines 13-50: Blank padding (38 lines)
- Lines 51+: Wafer map grid

## Entity: ValidationReport

| Field | Type | Description |
|-------|------|-------------|
| input_file | string | Original input filename |
| fields_found | list[string] | Successfully extracted fields |
| fields_missing | list[string] | Required fields not found in input |
| missing_field_rate | float | Ratio of missing to total required fields |
| total_input_dies | int | Die count from input file |
| total_output_dies | int | Die count in generated output |
| parse_errors | list[string] | Any parsing errors encountered |
| parse_error_count | int | Total number of parse errors |
| good_die_count | int | Number of passing dies in output |
| confidence_summary | dict[string, float] | Per-field confidence from rule proposal |

## State Transitions

```
File Upload → HeuristicAnalysis → LLM Rule Proposal → Rule Application → ASEKR Output + Validation
     │              │                    │                    │                    │
     ▼              ▼                    ▼                    ▼                    ▼
  raw bytes    format_type,         RuleProposal         ASEKROutput        ValidationReport
               encoding,            (JSON)               (text file)        (displayed)
               samples
```
