# Veracode Reporting API – Full Fetcher (HTTPie + HMAC)

![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![Output](https://img.shields.io/badge/output-JSONL%20%7C%20JSON%20%7C%20XLSX-green)
![Status](https://img.shields.io/badge/status-production--ready-brightgreen)

Production-ready CLI to export **all findings** from the Veracode Reporting REST API across any date range.

---

## ✨ Features

- **Full export** across any date range → auto-splits into ≤ **180-day windows** (API “6-month rule”)
- **Exhaustive pagination** → HAL `next`, metadata, length fallback; enforces your `--size`
- **Resilient retries** → handles 5xx / 429 / network hiccups with exponential backoff + jitter
- **Verification** (`--verify`)  
  - Pages seen vs reported  
  - Totals collected vs expected  
  - Auto-fetch missing pages  
  - Writes audit JSON
- **Stamping** (default) → adds `source_report_id`, `window_start`, `window_end`
- Outputs: JSONL + JSON + optional XLSX (skip with `--no-xlsx`)
- Professional console icons (`--icons`)

---

## 🔧 Prerequisites

- **Python 3.9+**
- **HTTPie** + Veracode HMAC plugin  

```bash
pip install httpie veracode-api-signing

	•	For Excel export (optional):

pip install pandas openpyxl xlsxwriter

Or skip Excel with --no-xlsx.

Or simply pip install -r requirements.txt
⸻

🔐 Authentication

export VERACODE_API_KEY_ID=YOUR_KEY_ID
export VERACODE_API_KEY_SECRET=YOUR_KEY_SECRET

Optional (macOS trust store):

export REQUESTS_CA_BUNDLE=$(python -m certifi)


⸻

🚀 Quick Start

python3 VERACODE_REPORT_FETCH.py \
  --from 2023-01-01 --to 2025-09-15 \
  --report-type FINDINGS --size 200 \
  --out ./out --icons --verify

Outputs:
	•	report_all.jsonl → line-delimited JSON (lossless)
	•	report_all.json → JSON array
	•	report_all.xlsx → Excel export (omit with --no-xlsx)
	•	audit/audit_<report_id>.json → per-window audit (with --verify)

⸻

⚙️ Options

--from YYYY-MM-DD       Start date (inclusive; 00:00:00)
--to YYYY-MM-DD         End date (inclusive; 23:59:59)
--report-type FINDINGS  Report type (default FINDINGS)
--size INT              Page size (default 1000)
--out PATH              Output dir (default ./out)
--filters FILE|<(JSON)  JSON merged into POST body
--sleep FLOAT           Delay after POST (default 0.5s)
--poll-interval FLOAT   Seconds between polls (default 2.0)
--poll-timeout INT      Max wait for COMPLETED (default 600)
--icons                 Show console icons
--no-stamp              Skip provenance stamping
--verify                Verify pages/totals; fetch missing pages
--strict                With --verify, exit on mismatch/dupes
--id-field FIELD        Unique key for duplicate check
--no-xlsx               Skip Excel export


⸻

🔍 Examples

All findings (recommended):

python3 VERACODE_REPORT_FETCH.py \
  --from 2022-01-01 --to 2025-09-15 \
  --report-type FINDINGS --size 200 \
  --out ./out --icons --verify

Strict CI run with duplicate check:

python3 VERACODE_REPORT_FETCH.py \
  --from 2023-01-01 --to 2025-09-15 \
  --report-type FINDINGS --size 200 \
  --out ./out --verify --strict --id-field finding_id

Open-only findings (via filters.json):

{ "status": "open" }

python3 VERACODE_REPORT_FETCH.py \
  --from 2024-01-01 --to 2025-09-15 \
  --report-type FINDINGS --size 500 \
  --out ./out_open --filters filters.json --icons

Skip Excel (JSON only):

python3 VERACODE_REPORT_FETCH.py \
  --from 2023-01-01 --to 2025-09-15 \
  --report-type FINDINGS --size 200 \
  --out ./out --verify --no-xlsx

Gentler polling & longer timeout:

python3 VERACODE_REPORT_FETCH.py \
  --from 2022-01-01 --to 2025-09-15 \
  --report-type FINDINGS --size 200 \
  --poll-interval 3.0 --poll-timeout 1800 \
  --out ./out --verify


⸻

🧾 Verification

Console output:

🧾 running verification …
      ✅ pages: seen=7 reported=7 => OK
      ✅ totals: collected=3002 expected=3002 => OK

Audit JSON:

{
  "report_id": "<uuid>",
  "page_indexes_seen": [0,1,2,3,4,5,6],
  "total_pages_reported": 7,
  "total_elements_reported": 3002,
  "collected_count_after_verify": 3002,
  "duplicate_id_count": 0,
  "strict_ok": true
}


⸻

🔁 Resilient Retries
	•	Retries up to 7 attempts on 5xx / 429 / network errors
	•	Exponential backoff + jitter
	•	Honors Retry-After header on 429
	•	Retries partial JSON decode errors
	•	Fails fast on 401 Unauthorized

⸻

🧰 Post-Run Checks

# Totals consistent
wc -l ./out/report_all.jsonl
jq 'length' ./out/report_all.json

# Provenance fields
jq '.[0] | {source_report_id, window_start, window_end}' ./out/report_all.json

# Duplicate scan
jq -r '.[].finding_id' ./out/report_all.json | sort | uniq -d | head


⸻

🛡️ Best Practices
	•	Large datasets → --size 200..500, --poll-interval 3..5, --poll-timeout 1800..3600
	•	Always use --verify in production
	•	Use --no-xlsx if Excel isn’t needed (lighter, faster)
	•	Leave status unset in filters to capture all findings

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
  JSONL : out/report_all.jsonl
  JSON  : out/report_all.json
  XLSX  : out/report_all.xlsx
📊 Grand total items: 10126

---
