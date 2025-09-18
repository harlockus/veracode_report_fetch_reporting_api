# Veracode Reporting API – Full Fetcher (HTTPie + HMAC)

![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![Output](https://img.shields.io/badge/output-JSONL%20%7C%20JSON%20%7C%20CSV%20%7C%20XLSX-green)
![Status](https://img.shields.io/badge/status-production--ready-brightgreen)

Production-ready CLI to export **all findings** from the Veracode Reporting REST API across any date range, with resilient retries, verification/auditing, and scalable outputs.

---

## ✨ Features

- **Full export** across any date range → auto-splits into ≤ **180-day windows** (API “6-month rule”)
- **Exhaustive pagination**
  - Follows HAL `next` and **enforces your `--size`**
  - Falls back to page metadata and length heuristics
- **Resilient retries** (5xx / 429 / network) with exponential backoff + jitter
- **Verification** (`--verify`)
  - Pages **seen vs reported**, totals **collected vs expected**
  - Writes per-window **audit JSON**
- **Stamping** (default)
  - Adds `source_report_id`, `window_start`, `window_end` to each row
- **Outputs**
  - **JSONL** (lossless, one row per line)
  - **JSON** (array)
  - **CSV** → **single file** (streamed; effectively unlimited rows)
  - **XLSX** → **single workbook** (adds sheets as needed, never multiple files)
- **Skip outputs** via flags: `--no-csv`, `--no-xlsx`
- **Professional console icons** (`--icons`)

---

## 🔧 Prerequisites

- **Python 3.9+**
- **HTTPie** + Veracode HMAC plugin  
  ```bash
  pip install httpie veracode-api-signing

	•	For Excel export (optional):

pip install pandas openpyxl xlsxwriter

If you don’t need Excel, use --no-xlsx (no pandas required).

⸻

🔐 Authentication

export VERACODE_API_KEY_ID=YOUR_KEY_ID
export VERACODE_API_KEY_SECRET=YOUR_KEY_SECRET

Optional (macOS trust store):

export REQUESTS_CA_BUNDLE=$(python -m certifi)

Avoid setting the legacy VERACODE_API_ID / VERACODE_API_KEY.

⸻

🚀 Quick Start

python3 VERACODE_REPORT_FETCH.py \
  --from 2023-01-01 --to 2025-09-15 \
  --report-type FINDINGS \
  --size 200 \
  --out ./out \
  --icons --verify

Outputs (in ./out):
	•	report_all_YYYYMMDD_HHMMSS.jsonl
	•	report_all_YYYYMMDD_HHMMSS.json
	•	report_all_YYYYMMDD_HHMMSS.csv   (single file)
	•	report_all_YYYYMMDD_HHMMSS.xlsx  (single workbook; multi-sheet if needed)
	•	audit/audit_<report_id>.json  (one per window when --verify is used)

⸻

⚙️ CLI Options

--from YYYY-MM-DD       Start date (inclusive; 00:00:00 per window)
--to YYYY-MM-DD         End date (inclusive; 23:59:59 per window)
--report-type FINDINGS  Report type (default FINDINGS)
--size INT              Page size for GET (default 1000)
--out PATH              Output directory (default ./out)
--filters FILE|<(JSON)  JSON merged into POST body (e.g., status, severity, application_name)
--sleep FLOAT           Delay after POST before polling (default 0.5s)
--poll-interval FLOAT   Seconds between status polls (default 2.0)
--poll-timeout INT      Max seconds to wait for COMPLETED (default 600)
--icons                 Show console icons
--no-stamp              Do not add source_report_id/window_start/window_end
--verify                Verify pages/totals; write audit JSON
--strict                With --verify, exit on mismatch/dupes
--id-field FIELD        Unique key for duplicate check (e.g., finding_id)
--no-xlsx               Skip Excel output
--no-csv                Skip CSV output

Filters (POST body)

Provide a JSON file with constraints (omit status to include all statuses: open + closed + mitigated):

{
  "status": "open",
  "policy_name": "Corporate Security Policy",
  "severity": ["5 - Very High", "4 - High"],
  "application_name": "Demo Web App"
}

Use with:

--filters filters.json

Or inline (bash/zsh):

--filters <(cat <<'JSON'
{ "status": "open", "severity": ["5 - Very High", "4 - High"] }
JSON
)


⸻

🔍 Examples

All outputs (single CSV + single XLSX workbook):

python3 VERACODE_REPORT_FETCH.py \
  --from 2022-01-01 --to 2025-09-17 \
  --report-type FINDINGS --size 200 \
  --out ./out --icons --verify

Skip Excel, keep CSV:

python3 VERACODE_REPORT_FETCH.py \
  --from 2022-01-01 --to 2025-09-17 \
  --report-type FINDINGS --size 200 \
  --out ./out --icons --verify --no-xlsx

JSON/JSONL only (no CSV, no XLSX):

python3 VERACODE_REPORT_FETCH.py \
  --from 2022-01-01 --to 2025-09-17 \
  --report-type FINDINGS --size 200 \
  --out ./out --icons --verify --no-xlsx --no-csv

Gentler polling & longer timeout (busy tenants):

python3 VERACODE_REPORT_FETCH.py \
  --from 2022-01-01 --to 2025-09-17 \
  --report-type FINDINGS --size 200 \
  --poll-interval 3.0 --poll-timeout 1800 \
  --out ./out --icons --verify

Open-only findings (filters):

python3 VERACODE_REPORT_FETCH.py \
  --from 2024-01-01 --to 2025-09-15 \
  --report-type FINDINGS --size 500 \
  --filters filters.json \
  --out ./out_open --icons --verify


⸻

🧾 Verification & Audit

With --verify, per window you’ll see:

🧾 running verification …
      ✅ pages: seen=7 reported=7 => OK
      ✅ totals: collected=3002 expected=3002 => OK

An audit file is written to ./out/audit/audit_<report_id>.json summarizing:
	•	Page indexes seen and the API’s total_pages
	•	API-reported total_elements (when present) vs collected
	•	Duplicate count when --id-field is set

Use --strict to fail the run on mismatches/duplicates.

⸻

🔁 Resilient Retries

All HTTP calls:
	•	Retry up to 7 attempts on 5xx, 429, and common network errors
	•	Honor Retry-After for 429
	•	Retry JSON decode hiccups
	•	Fail fast on 401 Unauthorized

Tuning tips:
	•	Large datasets: --size 200..500, --poll-interval 3..5, --poll-timeout 1800..3600

⸻

📄 Output Details
	•	JSONL – Source of truth; easiest to stream/pipe.
	•	JSON – Pretty-printed array.
	•	CSV – Single file; streamed from JSONL, flattened; lists are JSON-encoded strings in cells.
	•	XLSX – Single workbook; creates additional sheets (findings_01, findings_02, …) when a sheet approaches Excel’s row cap (~1,048,576). This avoids crashes while keeping one file.

⸻

🧰 Post-Run Checks

# Totals consistent
wc -l ./out/report_all_*.jsonl
jq 'length' ./out/report_all_*.json

# Provenance fields present
jq '.[0] | {source_report_id, window_start, window_end}' ./out/report_all_*.json

# Optional duplicate scan (adjust field)
jq -r '.[].finding_id' ./out/report_all_*.json | sort | uniq -d | head


⸻

🛡️ Best Practices
	•	Use --verify in production to prove full capture
	•	Prefer CSV for massive flat exports; use XLSX for analyst convenience
	•	Keep status unset in filters unless you need to narrow scope
	•	If the API is busy, reduce --size and increase poll interval/timeout

⸻

📸 Sample Console Output

🗂️ === Window 2023-12-22 → 2024-06-18 ===
  📄 report id: cae52e31-69e6-4994-be8f-20e146c96c71
  🔄 status: PROCESSING
  ✅ status: COMPLETED
    📦 page 0: 928 items  ➡️  window_total=0, grand_total=3754
    📦 page 1: 283 items  ➡️  window_total=928, grand_total=4682
    📦 page 2: 1036 items ➡️  window_total=1211, grand_total=4965
    ...
    🧾 running verification …
      ✅ pages: seen=7 reported=7 => OK
  📊 window complete: 3002 items  (grand_total=6756)
Outputs:
  JSONL : out/report_all_20250918_213455.jsonl
  JSON  : out/report_all_20250918_213455.json
  CSV   : out/report_all_20250918_213455.csv
  XLSX  : out/report_all_20250918_213455.xlsx
📊 Grand total items: 10126


⸻
Not a VERACODE official tool.
Utilizing https://docs.veracode.com/r/Reporting_REST_API
