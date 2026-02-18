# Feature Specification: LLM-Assisted Semiconductor MAP TXT Converter

**Feature Branch**: `001-llm-map-converter`
**Created**: 2026-02-18
**Status**: Draft
**Input**: User description: "LLM-Assisted Semiconductor MAP TXT Converter (Google Colab Demo)"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Upload and Auto-Analyze a Vendor MAP File (Priority: P1)

A team lead or engineer uploads an unknown vendor MAP TXT file in the Colab notebook. The system automatically detects the file structure (encoding, delimiter, header boundaries, data format) and proposes transformation rules as a structured JSON. The user can review the proposed rules, including which fields were detected (LotID, WaferNo, BinCode, etc.) and the confidence level for each mapping.

**Why this priority**: This is the core value proposition of the demo. Without file analysis and rule proposal, nothing else works. It demonstrates "AI understands unknown formats."

**Independent Test**: Can be fully tested by uploading any of the 6 sample input files and verifying that the system produces a valid rule proposal JSON with correct field mappings.

**Acceptance Scenarios**:

1. **Given** a vendor MAP file in any of the 6 known input formats, **When** the user uploads it and runs the analysis cell, **Then** the system displays detected format information (encoding, delimiter/structure type, header boundaries) and a field mapping proposal with confidence scores.
2. **Given** a file with an unrecognized format, **When** the user uploads it, **Then** the system still attempts analysis and flags uncertain fields with low confidence scores and alternative candidates rather than failing silently.
3. **Given** a file with non-UTF-8 encoding (e.g., cp949), **When** the user uploads it, **Then** the system correctly detects and handles the encoding.

---

### User Story 2 - Convert MAP File to ASEKR Standard Output (Priority: P1)

After reviewing the proposed rules, the user runs the conversion cell to generate the standardized ASEKR output file. The output follows the company-standard format with a fixed header block (Customer, Wafer ID, Wafer lot, Wafer No, Row/Column dimensions, Notch, Good bin code/count, Total good qty) followed by blank-line padding and a wafer grid map with standardized bin codes. The user can download the converted file.

**Why this priority**: Conversion output is the primary deliverable users care about. Without it, the demo has no practical value.

**Independent Test**: Can be fully tested by converting each of the 6 sample input files and comparing the result against the corresponding expected output in the `output/` directory.

**Acceptance Scenarios**:

1. **Given** a successfully analyzed MAP file, **When** the user runs the conversion, **Then** a downloadable standard output file is generated in the ASEKR format with all required header fields populated.
2. **Given** the 6 sample input files in the repository, **When** each is converted, **Then** the output matches or closely approximates the corresponding file in the `output/` directory.
3. **Given** a conversion is complete, **When** the user reviews the output, **Then** a validation report shows missing field rate, parse error count, and total die counts.

---

### User Story 3 - View and Download Rule Proposal JSON (Priority: P2)

The user can view the generated rule proposal JSON directly in the notebook output and download it for reference. The JSON captures the complete transformation logic: detected format details, header parsing rules, column/field mappings, bin code translation rules, and normalization steps. This makes the AI's reasoning transparent and auditable.

**Why this priority**: Transparency and explainability are key demo goals. Stakeholders need to see what the AI decided and why.

**Independent Test**: Can be fully tested by verifying the rule JSON is valid, contains all required schema sections, and includes evidence snippets from the source file.

**Acceptance Scenarios**:

1. **Given** a completed analysis, **When** the user views the rule JSON, **Then** it contains all required sections: detected_format, header_rules, column_mapping, value_normalization, confidence, candidates, and evidence.
2. **Given** a rule JSON, **When** an uncertain mapping exists, **Then** the JSON lists alternative candidates rather than making an unsupported guess.

---

### User Story 4 - Preview Converted Output Before Download (Priority: P3)

Before downloading the full converted file, the user can preview the first N rows of the converted output directly in the notebook, along with extracted metadata (LotID, WaferNo, etc.) and per-field confidence scores. This helps the user quickly verify correctness.

**Why this priority**: Improves usability for non-technical users who want to spot-check results before downloading.

**Independent Test**: Can be fully tested by uploading a sample file and verifying that the preview shows correctly parsed metadata and a partial wafer map grid.

**Acceptance Scenarios**:

1. **Given** a completed conversion, **When** the preview is shown, **Then** extracted metadata fields and the first portion of the wafer map grid are visible in the notebook output.

---

### Edge Cases

- What happens when the uploaded file is empty or contains no recognizable data structure?
- How does the system handle files with mixed encodings or binary content?
- What happens when the LLM returns invalid JSON or times out?
- How does the system handle very large MAP files (>10MB) within Colab's memory constraints?
- What happens when the file matches no known pattern and the LLM confidence is below threshold for all required fields?
- How are files handled that have a wafer map grid format (e.g., 1ACB86) versus a coordinate-based format (e.g., 60XHHA) versus a structured key-value format (e.g., 68ZBC3P)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST run entirely within a Google Colab notebook with no external server dependencies.
- **FR-002**: System MUST use native Colab UI (cell outputs, file upload widgets, display functions) for all user interactions. No external UI framework required.
- **FR-003**: System MUST accept vendor MAP files via Colab file upload (supporting .txt, .map, .TXT, and extensionless files).
- **FR-004**: System MUST detect file encoding automatically (at minimum UTF-8 and CP949).
- **FR-005**: System MUST perform heuristic pre-analysis before LLM invocation, including delimiter detection, header/data boundary estimation, and format type classification (grid map vs. coordinate-based vs. structured key-value).
- **FR-006**: System MUST send only sampled lines to the LLM (top N, middle N, bottom N lines) rather than the entire file, to control token usage and improve response stability.
- **FR-007**: System MUST require the LLM to output strict JSON conforming to the rule proposal schema (no prose or markdown wrapping).
- **FR-008**: System MUST apply proposed rules using deterministic code (not the LLM), ensuring reproducibility and stability.
- **FR-009**: System MUST generate output in the ASEKR standard map format with these header fields: version, Customer, Supplier, Wafer ID, Wafer lot, Wafer No, Row(Y), Column(X), Notch, Good bin code, Good bin count, Total good qty, followed by blank-line padding and the wafer grid map.
- **FR-010**: System MUST extract at minimum: LotID, WaferNo, and BinCode mappings from any input format. When required ASEKR header fields (e.g., Customer, Supplier) are not present in the input, the system MUST infer what it can from available context (e.g., LotID and WaferNo from filename patterns) and output remaining fields as user-fillable placeholders (e.g., `[UNKNOWN - please fill in]`).
- **FR-011**: System MUST generate a validation report showing: missing field rate, parse error count, and row/die counts.
- **FR-012**: System MUST make the converted output file and rule JSON downloadable from the notebook.
- **FR-013**: System MUST flag fields with confidence below a configurable threshold as uncertain and provide alternative candidates in the rule JSON.
- **FR-014**: System MUST use a free LLM accessible via Hugging Face (e.g., Hugging Face Inference API with free-tier models) as the default provider, and support swapping to other providers without modifying the core conversion logic.
- **FR-015**: System MUST handle the 6 distinct input formats found in the sample data: (a) plain grid map with single-char bins, (b) bracketed header with X/Y/B coordinate data, (c) BOF/EOF structured format with soft bin table and grid map, (d) grid map with trailing metadata, (e-f) WAFER_MAP structured format with embedded grid.

### Key Entities

- **Vendor MAP File**: The raw input file from a semiconductor vendor. Varies in structure, encoding, delimiter, and metadata layout across vendors. Contains wafer test results including die coordinates and bin codes.
- **ASEKR Standard Output**: The company-standard output format containing a fixed header block with wafer metadata, followed by a grid-based wafer map. All output files conform to this single format.
- **Rule Proposal JSON**: A structured JSON document capturing the LLM's analysis of an input file: detected format details, header parsing rules, field-to-field mappings, bin code translations, normalization steps, confidence scores, alternative candidates, and evidence snippets.
- **Validation Report**: A summary showing conversion quality metrics: missing fields, parse errors, die count comparison, and any discrepancies between input and output.

## Clarifications

### Session 2026-02-18

- Q: When input file lacks required ASEKR header fields (Customer, Supplier, etc.), what should the system do? → A: Infer what's possible (e.g., LotID from filename), leave rest as user-fillable placeholders.
- Q: How should the system determine bin code translation rules (varies across vendors)? → A: LLM proposes per-vendor bin-to-character mapping in the rule JSON.

## Assumptions

- The demo will run in Google Colab's free or standard tier.
- The LLM will be accessed via Hugging Face's free Inference API or a similar free-tier service. No paid API keys required by default.
- The 6 sample files in `input/` and `output/` represent the primary vendor formats that must work out of the box.
- The ASEKR standard map format (observed in all output files) is the single target output format. No other output format variants are needed.
- Blank-line padding between the header and the wafer map in the output is part of the standard format (38 blank lines to make the map start around line 51).
- Bin code translation rules are vendor-specific and must be proposed by the LLM as part of the rule JSON. The LLM should analyze the input bin codes, identify pass/fail semantics, and propose per-vendor character mappings (e.g., preserve original characters, map all fail to "X", or map specific codes to specific letters).
- The demo targets a single-file-at-a-time workflow, not batch processing.
- Perfection is not required; the demo must show the AI's ability to understand, propose, and convert, with explainable results.
- All interactions happen through native Colab cells (no external UI frameworks).

## Non-Goals

- Production deployment or enterprise-scale infrastructure
- Large-scale batch optimization or parallel file processing
- Full MES (Manufacturing Execution System) integration
- 100% accuracy across all possible vendor formats
- Support for image-based or binary wafer map formats

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can upload any of the 6 sample input files and receive a correctly structured ASEKR output that matches or closely approximates the expected output within 60 seconds.
- **SC-002**: The system correctly identifies LotID, WaferNo, and at least one bin code mapping for all 6 sample files with confidence scores above 0.7.
- **SC-003**: A non-technical user can complete the full workflow (upload → analyze → convert → download) by running notebook cells sequentially, without any coding or manual configuration steps.
- **SC-004**: The rule proposal JSON is valid, parseable, and contains evidence snippets from the source file for every proposed mapping.
- **SC-005**: The entire demo runs from a single "Run All" execution in Colab — no additional setup steps beyond running cells.
- **SC-006**: When presented with a previously unseen MAP file format, the system produces a reasonable (if imperfect) rule proposal and flags uncertain mappings rather than failing.
