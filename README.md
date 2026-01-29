# Spreadsheet Validator Agent

A multi-agent workflow system that validates Excel expense files, enables human-in-the-loop (HITL) corrections, and produces clean output files with computed columns.

## Overview

This project implements a **LangGraph-based** validation pipeline that:
- Ingests `.xlsx` or `.csv` files
- Validates rows against deterministic business rules
- Prompts users for corrections on invalid rows via API
- Outputs `success.xlsx` (valid rows + computed columns) and `errors.xlsx` (invalid rows + reasons)
- Supports re-uploading error files for incremental reprocessing

## Technology Stack

| Component | Technology |
|-----------|------------|
| Framework | LangGraph (deterministic state machine) |
| API Server | FastAPI with async endpoints |
| Database | SQLite for run state persistence |
| File Processing | openpyxl for Excel I/O |

## Quick Start

### Prerequisites

- Python 3.9+
- pip

### Installation

```bash
# Clone the repository
cd spreadsheet-validator

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Running the Server

```bash
uvicorn app.api.main:app --reload --port 8000
```

The API will be available at `http://localhost:8000`.

## API Usage

### 1. Create a Validation Run

```bash
# Basic upload with default settings
curl -X POST -F "file=@sample_data/expenses.xlsx" http://localhost:8000/runs

# With custom parameters
curl -X POST \
  -F "file=@sample_data/expenses.xlsx" \
  -F "as_of=2024-01-15" \
  -F "usd_rounding=cents" \
  -F 'cost_center_map={"ENG":"CC-001","FIN":"CC-002","HR":"CC-003","OPS":"CC-004"}' \
  http://localhost:8000/runs
```

**Response:**
```json
{
  "run_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "QUEUED"
}
```

### 2. Check Run Status

```bash
curl http://localhost:8000/runs/{run_id}
```

**Response:**
```json
{
  "run_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "WAITING_FOR_USER",
  "error": null,
  "parse_errors": [],
  "global_prompts": [
    {
      "kind": "global",
      "id": "cost_center:OPS",
      "field": "cost_center_map",
      "dept": "OPS",
      "type": "string",
      "message": "cost_center_map is missing a value for dept=OPS. Enter cost center code."
    }
  ],
  "row_prompts": [],
  "artifacts": [],
  "success_count": 0,
  "error_count": 0
}
```

### 3. Submit Answers

```bash
# Answer global prompts (cost center mapping)
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"cost_center:OPS": "CC-OPS-004"}' \
  http://localhost:8000/runs/{run_id}/answers

# Answer row prompts (fix invalid fields)
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"row:14:fx_rate": 1.07}' \
  http://localhost:8000/runs/{run_id}/answers

# Skip a prompt
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"skip:row:14:fx_rate": true}' \
  http://localhost:8000/runs/{run_id}/answers
```

### 4. Skip All Prompts

```bash
curl -X POST http://localhost:8000/runs/{run_id}/skip-all
```

### 5. Download Artifacts

```bash
# Download success file (only when status=COMPLETED)
curl -O http://localhost:8000/runs/{run_id}/artifacts/success.xlsx

# Download errors file
curl -O http://localhost:8000/runs/{run_id}/artifacts/errors.xlsx
```

### 6. List Recent Runs

```bash
curl http://localhost:8000/runs
```

## Run States

```
QUEUED --> RUNNING --> WAITING_FOR_USER --> RUNNING --> COMPLETED
                              |                            |
                              v                            v
                           (answers)                    FAILED
```

| State | Description |
|-------|-------------|
| `QUEUED` | Run is waiting to be processed |
| `RUNNING` | Pipeline is actively processing |
| `WAITING_FOR_USER` | Prompts pending user response |
| `COMPLETED` | Processing finished, artifacts available |
| `FAILED` | Error occurred during processing |

## Validation Rules

### Required Columns

| Column | Type | Rules |
|--------|------|-------|
| `employee_id` | string | Regex: `^[A-Z0-9]{4,12}$`, unique per file |
| `dept` | enum | One of: `FIN`, `HR`, `ENG`, `OPS` |
| `amount` | decimal | `0 < amount <= 100000` |
| `currency` | enum | One of: `USD`, `EUR`, `GBP`, `INR` |
| `spend_date` | date | Format: `YYYY-MM-DD`, must be `<= as_of` |
| `vendor` | string | Non-empty |
| `fx_rate` | decimal | Required if `currency != USD`, range: `0.1-500` |

### Cross-Field Rules

- If `currency != USD`, `fx_rate` is required
- Duplicate `(employee_id, spend_date)` pairs are invalid
- If `dept = FIN` and `amount > 50000`, row is flagged for CFO approval

### Computed Output Columns

| Column | Formula |
|--------|---------|
| `amount_usd` | `amount * fx_rate` (rounded per `usd_rounding`) |
| `cost_center` | Looked up from `cost_center_map[dept]` |
| `approval_required` | `YES` if FIN dept and amount > 50000, else `NO` |

## User-Provided Globals

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `as_of` | date | Today | Reference date for spend_date validation |
| `usd_rounding` | enum | `cents` | `"cents"` (2 decimals) or `"whole"` (integer) |
| `cost_center_map` | JSON | From sample_data | Mapping: `dept -> cost_center_code` |

## Reprocessing Error Files

When you re-upload an `errors.xlsx` file:
1. The system detects it contains error metadata (`row_hash`, `error_reason`)
2. Rows with unchanged `row_hash` are skipped (not re-validated)
3. Only modified rows are re-validated
4. This allows incremental fixing of errors

## Project Structure

```
spreadsheet-validator/
├── app/
│   ├── api/
│   │   ├── main.py          # FastAPI endpoints
│   │   ├── models.py        # Pydantic models
│   │   ├── store.py         # SQLite persistence
│   │   └── worker.py        # Background task executor
│   └── pipeline/
│       ├── graph.py         # LangGraph state machine
│       ├── ingest.py        # Excel/CSV parsing
│       ├── validate.py      # Validation rules
│       ├── transform.py     # Computed columns
│       ├── package.py       # Output file writing
│       ├── prompts.py       # Row-level HITL prompts
│       ├── hitl_globals.py  # Global HITL prompts
│       ├── hashing.py       # Row fingerprinting
│       └── trace.py         # JSONL trace logging
├── sample_data/
│   ├── expenses.xlsx        # Test data
│   └── cost_center_map.json # Department mappings
├── data/                    # Runtime data (gitignored)
│   ├── uploads/             # Uploaded files
│   ├── runs/                # Run artifacts
│   └── runs.db              # SQLite database
├── traces/                  # JSONL execution logs
├── requirements.txt
├── PRD.md                   # Product requirements
└── README.md
```

## LangGraph Pipeline

```
┌──────────┐    ┌─────────────┐    ┌──────────┐    ┌───────────┐    ┌─────────┐
│ Validate │───▶│ Global HITL │───▶│ Row HITL │───▶│ Transform │───▶│ Package │
└──────────┘    └─────────────┘    └──────────┘    └───────────┘    └─────────┘
                     │                   │
                     ▼                   ▼
                [END if prompts]    [END if prompts]
```

Each node saves state to SQLite, enabling safe resume after HITL pauses.

## Trace Logging

Each run generates a JSONL trace file at `traces/{run_id}.jsonl`:

```jsonl
{"timestamp":"2024-01-15T10:30:00Z","run_id":"abc123","event":"run_started","data":{"row_count":10}}
{"timestamp":"2024-01-15T10:30:01Z","run_id":"abc123","event":"prompt_generated","data":{"prompt_id":"cost_center:OPS"}}
{"timestamp":"2024-01-15T10:30:05Z","run_id":"abc123","event":"answer_received","data":{"prompt_id":"cost_center:OPS","value":"CC-004"}}
{"timestamp":"2024-01-15T10:30:06Z","run_id":"abc123","event":"artifact_created","data":{"name":"success.xlsx","row_count":8}}
```

## Sample Data

### cost_center_map.json

```json
{
  "ENG": "CC-ENG-001",
  "FIN": "CC-FIN-002",
  "HR": "CC-HR-003"
}
```

Note: `OPS` is intentionally missing to demonstrate the HITL global prompt flow.

### expenses.xlsx

A sample expense file is provided with various test cases:
- Valid rows with all fields
- Rows missing `fx_rate` for non-USD currencies
- Rows with `spend_date` after `as_of`
- Duplicate `(employee_id, spend_date)` pairs
- Large FIN expenses requiring CFO approval

## Development

### Running Tests

```bash
# TODO: Add test suite
pytest tests/
```

### API Documentation

When the server is running, visit:
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

## License

MIT
