# Data Agents Demo

A small AI-powered tool that takes a messy Excel or CSV file (the kind of spreadsheet you actually get in real life — with notes above the table, weird column names, and inconsistent values) and turns it into a clean, predictable file that other software can use without complaint.

GitHub home: https://github.com/Alleyfoo/Data-agents-demo

## Why this project matters

Most "real" spreadsheets aren't clean. The headers might be on row 4 instead of row 1. Numbers are stored as text. Some cells have stray spaces. A normal data pipeline (an automated process that moves and transforms data) chokes on all of this.

This project shows a more thoughtful approach:

- **It asks before it guesses.** When the tool isn't sure which row holds the column titles, it stops and asks the user instead of silently picking wrong.
- **Every decision is recorded.** You can replay any run later and see exactly why it did what it did — useful for audits, debugging, or convincing a stakeholder.
- **The brains and the buttons are separate.** The cleanup logic lives in one place; the user interfaces (command line, terminal app, web app) are simple wrappers around it. You can swap or add interfaces without rewriting the core.

In short: it treats data cleanup like a careful conversation, not a black box.

## Quickstart (command line)

You'll need Python 3.11 or newer (a programming language), `git`, and `pip` (Python's package installer). The commands below assume you're using PowerShell on Windows, in the project folder.

```powershell
python -m venv .venv
.\\.venv\\Scripts\\Activate.ps1
python -m pip install -r demos/requirements-demo.txt

# Try the included sample
python data_agents_cli.py run --input data/samples/sample_mini.csv --run-id demo --interactive

# Or point to your own file (it will ask if it needs help confirming the header)
python data_agents_cli.py run --input path\\to\\your.xlsx --run-id demo --interactive

# Or run it in two steps: detect, confirm, then continue
python data_agents_cli.py run --input path\\to\\your.xlsx --run-id demo
python data_agents_cli.py confirm --run-id demo --choice row_1
python data_agents_cli.py resume --run-id demo
```

When it finishes, the results are saved in `artifacts/<run-id>/`. You'll find:

- `clean.csv` — the cleaned-up spreadsheet
- `schema_spec.json` — a description of the columns and their data types
- a "shadow log" — a step-by-step record of what happened

Any CSV or Excel file works — just point `--input` at it.

## Other ways to use it

- **Terminal app** (a keyboard-driven app inside your terminal window):
  `python demos/tui_app.py --input path\\to\\your.xlsx --interactive`
- **Web app** (opens in your browser, built with Streamlit):
  `streamlit run demos/streamlit_app.py`
- **Mapping studio** (web app for matching messy column names to clean ones):
  `streamlit run demos/streamlit_mapping_studio.py`
- **One-click launchers (Windows):** `demos/run_tui_demo.bat`, `demos/run_streamlit_demo.bat`

## What it does, step by step

1. You give it a CSV or Excel file.
2. It looks at the file and figures out which row is most likely the header (the row with column titles).
3. If the answer isn't obvious, it pauses and asks you — through whichever interface you're using — to confirm.
4. Once it knows the header, it cleans up the data: trims stray spaces, normalizes numbers stored as text, handles empty cells consistently.
5. It writes out the clean file plus a record of every decision it made.

Visually:

```
Your CSV/XLSX
   │
   ▼
runtime.excel_flow.puhemies_run_from_file
   ├─ Detect header candidates → evidence_packet.json
   ├─ If ambiguous → header_spec.json → CLI/TUI/Streamlit asks you
   ├─ You confirm → human_confirmation.json
   ├─ Orchestrator resumes → cleans/normalizes data
   └─ Writes outputs:
        • clean.csv
        • schema_spec.json (normalized headers/field types)
        • shadow.jsonl (trace log)
```

## Why this is more than "just read the file"

- **Header detection is treated as a real decision**, not something quietly guessed inside a parser. Mistakes here cause silent damage to data pipelines downstream.
- **Human confirmations are saved**, so a run can be replayed exactly the same way later — important for compliance and debugging.
- **The interfaces are intentionally thin.** The real work happens in the `runtime/` folder, so a new UI (or a fully automated agent) can plug in without changing the cleanup logic.
- **Outputs are structured** (JSON for metadata, CSV for data), so the next system in the chain can read them programmatically — no screenshots or copy-paste.

## Where to look in the code

A guided tour for engineers reviewing the project:

- **Orchestration** (the conductor that runs the steps in order): `runtime/excel_flow.py` — see `puhemies_run_from_file` (first pass), `puhemies_continue` (after the user confirms), and `_write_json` / `_append_shadow` (which save artifacts and audit logs).
- **Header detection** (figuring out which row is the column titles): `_normalize_header`, `_header_looks_like_data`, and the candidate-generation logic inside `excel_flow.py`. It scores possible header rows, flags ambiguous cases, and writes the result to `header_spec.json`.
- **Human-in-the-loop** (the part where the tool asks for help): `data_agents_cli.py` (command line), `demos/tui_app.py` (terminal app), and `demos/streamlit_app.py` & `demos/streamlit_mapping_studio.py` (web apps). Each one shows the same question from `header_spec.json` and calls `write_human_confirmation` once the user answers.
- **Data cleaning**: `runtime/data_janitor.py` — `clean_value` and `clean_series` strip whitespace, normalize number-like text, and handle empty cells before writing `clean.csv`.
- **Schema and evidence**: `schema_spec.json` and `evidence_packet.json` capture the cleaned-up column names, confidence scores, and decisions, so the run can be replayed or debugged.

### How a request flows through the system

```
CLI/TUI/Streamlit
   │ calls
   ▼
runtime.excel_flow.puhemies_run_from_file
   ├─ _build_header_candidates → header_spec.json
   ├─ _append_shadow           → shadow.jsonl (event log)
   ├─ write_human_confirmation → human_confirmation.json (after you pick a header)
   ├─ puhemies_continue        → re-loads confirmation + cleans data
   ├─ _infer_dtype/clean_series → schema_spec.json, clean.csv
   └─ _write_json              → evidence_packet.json, save_manifest.json
```

## Design principles

- **Be explicit about decisions.** When confidence is low, ask the human and write the answer down. No silent guesses.
- **Keep concerns separate.** Logic lives in `runtime/`. Interfaces are simple shells that prompt and display.
- **Traceability by default.** Every step is logged to `shadow.jsonl` and to structured spec files, so nothing is hidden.
- **Extensible.** Reusable agent and skill definitions live under `agent-base/` (also mirrored in `.github/`), so this can be dropped into a larger AI-agent workflow.

## What's in this repo

- `data_agents_cli.py` / `data-agents.ps1` — the command-line entry points (Python script and a PowerShell wrapper).
- `runtime/` — the core logic: header detection, cleaning, and orchestration.
- `demos/` — the terminal app, the web apps, and the dependencies they need.
- `tests/` — basic automated tests covering the main flow.
- `agent-base/` and `.github/` — shared agent definitions and templates.

## Example output

- A cleaned CSV produced from the bundled sample: `docs/example-output/sample_mini_clean.csv`

## Links

- Profile: https://github.com/Alleyfoo
- Related project: https://github.com/Alleyfoo/Data-tool-demo
