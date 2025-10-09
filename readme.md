# URL Inspection Exporter (Google Search Console)

Bulk-inspect a list of URLs with the Google Search Console **URL Inspection API** and export rich index status details to Excel. The script:

- Authenticates with OAuth (stores/refreshes `token.json`)
- Verifies access to your GSC property
- Reads URLs from an Excel file (`urls.xlsx`)
- Calls the URL Inspection API in parallel with retries + backoff
- Writes a structured Excel report (`export.xlsx`)

---

## Table of contents
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Input file format](#input-file-format)
- [Output columns](#output-columns)
- [How it works](#how-it-works)
- [Tuning performance & reliability](#tuning-performance--reliability)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [FAQ](#faq)
- [License](#license)

---

## Prerequisites

- Python 3.9+ recommended  
- A **verified** Google Search Console property for your site  
- A Google Cloud project with **OAuth client credentials** (Desktop app)  
- The URL Inspection API enabled in the project  

### Install Python dependencies

```bash
pip install -U pandas requests google-auth google-auth-oauthlib google-api-python-client openpyxl
```

> `openpyxl` is needed for reading/writing `.xlsx`.

---

## Quick start

1. **Create OAuth client credentials**
   - In Google Cloud Console: APIs & Services → Credentials → **Create Credentials → OAuth client ID**.
   - Application type: **Desktop app**.
   - Download the JSON and save it as `client_secret.json` next to the script (or adjust the path in config).

2. **Enable APIs**
   - APIs & Services → Library → enable **Search Console API** (a.k.a. Webmasters API).

3. **Verify your site in GSC**
   - Ensure the site you plan to query is verified under your Google account.
   - The script checks this automatically and warns if it’s not.

4. **Prepare `urls.xlsx`**
   - Create an Excel file with a single column named **`URL`** (see [Input file format](#input-file-format)).

5. **Run**
   ```bash
   python your_script.py
   ```
   - A browser window will open the first time to complete OAuth.
   - On success, results are exported to **`export.xlsx`**.

---

## Configuration

Edit these values at the top of the script to fit your environment:

```python
URLS_XLSX = '/urls.xlsx'          # path to input Excel with a column named "URL"
CLIENT_SECRET = "/client_secret.json"
TOKEN_JSON = 'token.json'         # will be created & refreshed automatically
GSC_SITE_URL = 'https://www.figma.com/'  # must match a verified property in your GSC
EXPORT_XLSX = 'export.xlsx'
SCOPES = ['https://www.googleapis.com/auth/webmasters.readonly']
REQUEST_TIMEOUT_S = 60
WORKERS = 20                      # 10–30 is typical; lower if you hit rate limits
MAX_RETRIES = 3
PRINT_EVERY = 10                  # progress heartbeat cadence
VERBOSE_URL_LOGS = True           # per-URL logs on/off
```

**Notes**
- `GSC_SITE_URL` must exactly match a verified property shown in GSC (including scheme and trailing slash).
- If your files live elsewhere, provide absolute paths or run from the directory that contains them.

---

## Input file format

An Excel workbook (default: `urls.xlsx`) with a sheet that contains a **`URL`** column:

| URL                              |
|----------------------------------|
| https://www.example.com/         |
| https://www.example.com/page-a   |
| https://www.example.com/page-b?x |
| …                                |

- Blank cells are ignored.  
- Additional columns are ignored.

---

## Output columns

The report (default: `export.xlsx`) contains one row per URL with:

| Column | Description |
|--------|--------------|
| `URL` | the inspected URL |
| `inspectionResultLink` | GSC link to the inspection result |
| `verdict` | overall verdict (`PASS`, `FAIL`, `NEUTRAL`, or `No Data`) |
| `coverageState` | coverage label (e.g. `Submitted and indexed`) |
| `robotsTxtState` | robots.txt evaluation |
| `indexingState` | canonical/indexing evaluation |
| `lastCrawlTime` | timestamp of last crawl |
| `pageFetchState` | fetch result |
| `crawledAs` | crawler type (`MOBILE`, `DESKTOP`) |
| `userCanonical` | user-declared canonical |
| `googleCanonical` | Google-selected canonical |
| `sitemaps` | semicolon-separated list of sitemaps |
| `referringUrls` | semicolon-separated list of discovered referring URLs |
| `mobileUsabilityVerdict` | mobile usability verdict |
| `_status_code` | raw HTTP status from API call |
| `_error` | error message if any |

> Missing fields are filled with `"No Data"` for easier analysis.

---

## How it works

1. **Auth & token management**
   - Loads `token.json` if present, otherwise runs a local OAuth flow.
   - Refreshes expired tokens automatically and persists the new token.

2. **Property verification**
   - Lists all verified GSC properties.
   - Confirms that `GSC_SITE_URL` is one of them.

3. **Parallel inspection**
   - Uses `ThreadPoolExecutor` to run inspections concurrently.
   - Retries on 401/429/5xx with exponential backoff + jitter.
   - On 401, refreshes the token once globally and retries.

4. **Normalization & export**
   - Extracts a consistent set of fields.
   - Writes to Excel for easy filtering and reporting.

---

## Tuning performance & reliability

- **`WORKERS`** – start around 10–20. Lower if you see many `429` errors.  
- **`MAX_RETRIES`** – default 3; increase if network errors persist.  
- **`REQUEST_TIMEOUT_S`** – raise for slower networks.  
- **`PRINT_EVERY`** – set higher for quieter logging on large lists.  
- **`VERBOSE_URL_LOGS`** – set `False` for less console spam.

---

## Troubleshooting

- **Unverified property warning** – log into GSC with the same account; verify the property.  
- **401 Unauthorized** – delete `token.json` and re-run to re-authenticate.  
- **429 Too Many Requests** – lower concurrency.  
- **Quota errors** – respect API usage limits.  
- **Input Excel missing `URL` column** – ensure exact header name.  
- **Non-JSON response / `_error` populated** – inspect `_status_code` and `_error` fields.

---

## Security notes

- Don’t commit `client_secret.json` or `token.json`.  
- Store credentials securely.  
- The script requests **read-only** Search Console scope.

---

## FAQ

**Q:** Can I use multiple properties?  
**A:** Yes—run separate passes with different `GSC_SITE_URL` values.

**Q:** Must URLs match my verified property?  
**A:** Yes, inspected URLs must belong to a verified domain.

**Q:** How many URLs can I inspect?  
**A:** As many as your quota allows. Run in chunks (5–10k/day) if large.

**Q:** Can I export CSV instead?  
**A:** Replace the final export with:
```python
df.to_csv('export.csv', index=False)
```

---

## License

MIT (or adapt to your repo’s standard license).

---

### Example run

```bash
# 1) Install dependencies
pip install -U pandas requests google-auth google-auth-oauthlib google-api-python-client openpyxl

# 2) Place client_secret.json and urls.xlsx next to the script

# 3) Edit configuration if needed

# 4) Run
python url_inspect_export.py
```

**Output:**  
`export.xlsx` with one row per URL and inspection fields listed above.
