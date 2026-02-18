# Tasks: LLM-Assisted MAP TXT Converter

**Input**: Design documents from `/specs/001-llm-map-converter/`
**Prerequisites**: plan.md, spec.md, data-model.md, research.md, contracts/

**Tests**: Not explicitly requested. Manual validation against 6 sample input/output pairs is specified in the plan.

**Organization**: Tasks are grouped by user story. US1 and US2 are both P1 and tightly coupled (analyze then convert), so they share a single MVP phase.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different cells/functions, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- All file paths reference `map_converter.ipynb` (single-notebook architecture)

---

## Phase 1: Setup

**Purpose**: Create notebook skeleton and install/configure dependencies

- [x] T001 Create notebook file `map_converter.ipynb` with 10 empty cell placeholders matching the cell architecture in plan.md
- [x] T002 Implement Cell 1 (Setup & Dependencies): pip install `huggingface_hub` and `pydantic`, imports for all standard libraries, HF API token input via `getpass` in `map_converter.ipynb`
- [x] T003 Implement Cell 2 (Configuration): define `LLM_MODEL`, `CONFIDENCE_THRESHOLD`, `SAMPLE_LINE_COUNT`, `OUTPUT_DIR` constants in `map_converter.ipynb`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure cells that ALL user stories depend on

**Warning**: No user story work can begin until this phase is complete

- [x] T004 [P] Implement Cell 3 (Encoding Detection & File Reader): `chardet`-based detection with UTF-8 → CP949 → EUC-KR → Latin-1 fallback chain and `read_file()` function in `map_converter.ipynb`
- [x] T005 [P] Implement Cell 5 (LLM Provider Abstraction): `LLMProvider` base class with `generate_rules(prompt, schema)` method and `HuggingFaceProvider` implementation using `huggingface_hub.InferenceClient` in `map_converter.ipynb`
- [x] T006 [P] Implement Pydantic models for `RuleProposal`, `FieldExtraction`, `BinMapping`, `EvidenceSnippet`, `HeuristicAnalysis`, `ASEKROutput`, and `ValidationReport` entities per data-model.md in `map_converter.ipynb` Cell 2 (Configuration)
- [x] T007 Implement Cell 4 (Heuristic Analyzer): `format_type` classification (`grid_map`, `coordinate_xy_b`, `structured_kv`, `unknown`), delimiter detection, header boundary estimation, and sampled lines assembly (top/middle/bottom N) in `map_converter.ipynb`

**Checkpoint**: Foundation ready — file reading, format detection, LLM provider, and data models are in place

---

## Phase 3: User Story 1 + User Story 2 — Upload, Analyze & Convert (Priority: P1) MVP

**Goal**: User uploads a vendor MAP file, system proposes rules via LLM, applies rules deterministically, generates ASEKR standard output, and provides validation report + download

**Independent Test**: Upload each of the 6 sample files from `input/`, verify rule proposal JSON is valid and output matches corresponding file in `output/`

### US1: Rule Generation (Analyze)

- [x] T008 [US1] Implement Cell 6 (Rule Generator) — prompt construction: build LLM prompt from `HeuristicAnalysis` result including format_type hint, sampled lines, and target JSON schema in `map_converter.ipynb`
- [x] T009 [US1] Implement Cell 6 (Rule Generator) — LLM call and response parsing: call `HuggingFaceProvider.generate_rules()`, parse JSON response, validate against Pydantic `RuleProposal` model with retry on invalid JSON in `map_converter.ipynb`

### US2: Rule Application (Convert)

- [x] T010 [US2] Implement Cell 7 (Rule Applier) — metadata extraction: apply regex patterns from `metadata_extraction` rules to extract LotID, WaferNo, Customer, Supplier, WaferID, Notch; fallback to filename pattern matching per plan.md in `map_converter.ipynb`
- [x] T011 [P] [US2] Implement Cell 7 (Rule Applier) — grid construction for `grid_map` format: parse character grid lines, apply `grid_start_line`/`grid_end_line` boundaries in `map_converter.ipynb`
- [x] T012 [P] [US2] Implement Cell 7 (Rule Applier) — grid construction for `coordinate_xy_b` format: parse X/Y/B records, determine grid dimensions from min/max, build 2D character grid, fill empty positions with `.` in `map_converter.ipynb`
- [x] T013 [P] [US2] Implement Cell 7 (Rule Applier) — grid construction for `structured_kv` format: extract grid from structured blocks (e.g., `MAP = {}`, `[SOFT BIN MAP]`), handle multi-char bin codes in `map_converter.ipynb`
- [x] T014 [US2] Implement Cell 7 (Rule Applier) — bin code translation: apply `bin_mapping` rules to convert source bin codes to single-char ASEKR output characters, count good/fail dies in `map_converter.ipynb`
- [x] T015 [US2] Implement Cell 8 (ASEKR Output Generator): format 12-line header per `asekr-output-format.md`, add 38 blank padding lines, append wafer map grid, write to file in `map_converter.ipynb`
- [x] T016 [US2] Implement Cell 9 (Validator): check field completeness, compare die counts between input and output, calculate missing field rate and parse error count, generate `ValidationReport` in `map_converter.ipynb`

### Main Workflow (US1 + US2 integration)

- [x] T017 [US1] [US2] Implement Cell 10 (Main Workflow): orchestrate Upload → `read_file()` → `HeuristicAnalyzer` → `RuleGenerator` → `RuleApplier` → `ASEKROutputGenerator` → `Validator` → file download via `google.colab.files.download()` in `map_converter.ipynb`

**Checkpoint**: Core MVP is complete — upload any MAP file, get ASEKR output and validation report

---

## Phase 4: User Story 3 — View and Download Rule JSON (Priority: P2)

**Goal**: User can view the LLM's rule proposal JSON in the notebook and download it as a file

**Independent Test**: Upload a sample file, verify rule JSON is displayed with all required sections (detected_format, header_rules, metadata_extraction, map_parsing, bin_mapping, confidence, evidence) and is downloadable

### Implementation for User Story 3

- [x] T018 [US3] Add formatted rule JSON display in Cell 10 output: pretty-print the `RuleProposal` JSON with syntax highlighting using `IPython.display` and `json.dumps(indent=2)` in `map_converter.ipynb`
- [x] T019 [US3] Add rule JSON file download: save rule proposal as `{filename}_rules.json` and trigger `google.colab.files.download()` in `map_converter.ipynb`

**Checkpoint**: Users can now inspect AI reasoning and download rule JSON alongside the converted output

---

## Phase 5: User Story 4 — Preview Converted Output (Priority: P3)

**Goal**: User can preview extracted metadata and partial wafer map grid in the notebook before downloading

**Independent Test**: Upload a sample file, verify metadata table and first N grid rows are displayed in notebook output with confidence scores

### Implementation for User Story 4

- [x] T020 [US4] Add metadata preview table in Cell 10: display extracted fields (LotID, WaferNo, Customer, etc.) with per-field confidence scores using `IPython.display.HTML` table in `map_converter.ipynb`
- [x] T021 [US4] Add partial wafer map grid preview in Cell 10: display first 10 rows of the converted grid with row/column dimensions annotation in `map_converter.ipynb`

**Checkpoint**: All user stories complete — full demo workflow operational

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: End-to-end validation and edge case hardening

- [x] T022 End-to-end validation: run all 6 sample input files through the complete pipeline and compare outputs against expected files in `output/` (deterministic parts validated locally; full LLM-based E2E requires Colab execution)
- [x] T023 Edge case handling: add graceful error handling for empty files, unrecognized formats, LLM timeout, invalid JSON response, and large files (>10MB) in `map_converter.ipynb`
- [x] T024 Run quickstart.md validation: verify the documented workflow in `specs/001-llm-map-converter/quickstart.md` matches the actual notebook behavior

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — BLOCKS all user stories
- **US1 + US2 (Phase 3)**: Depends on Phase 2 completion — this is the MVP
- **US3 (Phase 4)**: Depends on T009 (rule generation produces JSON to display)
- **US4 (Phase 5)**: Depends on T015 (ASEKR output available to preview)
- **Polish (Phase 6)**: Depends on all user stories being complete

### Within Phase 3 (MVP)

- T008 → T009 (prompt construction before LLM call)
- T009 → T010, T011, T012, T013, T014 (rules must exist before application)
- T011, T012, T013 can run in parallel (different format handlers)
- T010, T014 → T015 (metadata + translated grid needed for output)
- T015 → T016 (output needed for validation)
- T016 → T017 (all components needed for orchestration)

### Parallel Opportunities

- Phase 2: T004, T005, T006 can all run in parallel (different cells, no dependencies)
- Phase 3: T011, T012, T013 can run in parallel (different format type handlers)
- Phase 4: T018, T019 can run in parallel (display vs download)
- Phase 5: T020, T021 can run in parallel (metadata table vs grid preview)

---

## Parallel Example: Phase 2 (Foundational)

```
# Launch all independent foundational tasks together:
Task: "Implement Cell 3 (Encoding Detection & File Reader) in map_converter.ipynb"
Task: "Implement Cell 5 (LLM Provider Abstraction) in map_converter.ipynb"
Task: "Implement Pydantic models for all entities in map_converter.ipynb"
```

## Parallel Example: Phase 3 Grid Construction

```
# Launch all format-specific grid builders together:
Task: "Grid construction for grid_map format in map_converter.ipynb"
Task: "Grid construction for coordinate_xy_b format in map_converter.ipynb"
Task: "Grid construction for structured_kv format in map_converter.ipynb"
```

---

## Implementation Strategy

### MVP First (Phase 1 + 2 + 3)

1. Complete Phase 1: Setup (T001-T003)
2. Complete Phase 2: Foundational (T004-T007) — CRITICAL, blocks everything
3. Complete Phase 3: US1 + US2 (T008-T017)
4. **STOP and VALIDATE**: Test with all 6 sample files
5. Demo-ready with core upload → analyze → convert → download flow

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. US1 + US2 → Core demo works (MVP!)
3. US3 → Rule transparency added
4. US4 → Preview UX improvement added
5. Polish → Edge cases hardened, validation confirmed

---

## Notes

- All tasks target a single file: `map_converter.ipynb`
- Cell numbers in task descriptions match the 10-cell architecture from plan.md
- [P] tasks can run in parallel only if they modify different cells or non-overlapping functions
- No tests phase included (not requested); validation is manual against 6 sample pairs
- Total: 24 tasks across 6 phases
