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
