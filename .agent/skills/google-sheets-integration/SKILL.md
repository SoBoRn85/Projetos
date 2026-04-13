---
name: google-sheets-integration
description: Google Sheets API v4 integration patterns. Service Account vs OAuth2 authentication, batchUpdate for high-performance writes, batch reads, typed column mapping, rate limit handling, and formula preservation. Use when reading from or writing to Google Sheets, syncing data between Sheets and databases, performing bulk operations on spreadsheets, or setting up Google Sheets API authentication. Use PROACTIVELY whenever the project involves Google Sheets as a data source or destination. Do NOT use for Excel files (use openpyxl/xlsxwriter), or for Google Docs/Slides (different APIs).
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Google Sheets Integration

> **Google Sheets is a database for humans — treat it accordingly.**
> High-performance, type-safe integration with the Sheets API v4.

---

## ⚠️ Core Principle

- Sheets is NOT a database — but users treat it like one
- `batchUpdate` is always better than cell-by-cell writes
- Respect rate limits (60 requests/min for reads, 60 for writes)
- Service Accounts for server-to-server; OAuth2 for user-facing
- Always preserve formulas — never overwrite what the user built

---

## 1. Authentication Strategy

### Decision Tree

```
Who is accessing the spreadsheet?
│
├── Server/Backend (no user interaction)
│   └── Service Account
│       ├── Create in Google Cloud Console
│       ├── Share spreadsheet with SA email
│       └── Use JSON key file (stored securely)
│
├── User-facing app (user's own sheets)
│   └── OAuth2
│       ├── User grants permission via consent screen
│       ├── Store refresh token securely
│       └── Use access token for requests
│
└── Hybrid (server reads, user writes)
    └── Service Account for reads + OAuth2 for writes
```

### Service Account Setup

```python
from google.oauth2.service_account import Credentials
from googleapiclient.discovery import build

SCOPES = ["https://www.googleapis.com/auth/spreadsheets"]

def get_sheets_service(credentials_path: str):
    """Create Sheets API service with Service Account."""
    creds = Credentials.from_service_account_file(
        credentials_path, scopes=SCOPES
    )
    return build("sheets", "v4", credentials=creds)

# Usage
service = get_sheets_service("service-account.json")
sheets = service.spreadsheets()
```

### OAuth2 Setup (Next.js / Supabase)

```python
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request

def get_user_sheets_service(
    access_token: str,
    refresh_token: str,
    client_id: str,
    client_secret: str,
):
    """Create Sheets API service with user OAuth2 credentials."""
    creds = Credentials(
        token=access_token,
        refresh_token=refresh_token,
        token_uri="https://oauth2.googleapis.com/token",
        client_id=client_id,
        client_secret=client_secret,
        scopes=SCOPES,
    )
    if creds.expired and creds.refresh_token:
        creds.refresh(Request())
    return build("sheets", "v4", credentials=creds)
```

### Security Checklist

| Practice | Why |
|----------|-----|
| Store SA key in Supabase Vault or env | Never commit to repo |
| Use minimum scopes | `spreadsheets` only, not `drive` |
| Share sheet with SA email, not public | Principle of least privilege |
| Encrypt OAuth tokens at rest | pgcrypto for refresh tokens |
| Rotate SA keys periodically | Limit blast radius |

---

## 2. Reading Data

### Single Range Read

```python
async def read_range(
    service,
    spreadsheet_id: str,
    range_name: str,
    value_render: str = "FORMATTED_VALUE",
) -> list[list[str]]:
    """Read a range from a spreadsheet.

    value_render options:
    - FORMATTED_VALUE: As displayed (with formatting)
    - UNFORMATTED_VALUE: Raw values (numbers, dates as serial)
    - FORMULA: Shows formulas instead of values
    """
    result = (
        service.spreadsheets()
        .values()
        .get(
            spreadsheetId=spreadsheet_id,
            range=range_name,
            valueRenderOption=value_render,
        )
        .execute()
    )
    return result.get("values", [])
```

### Batch Read (Multiple Ranges)

```python
async def batch_read(
    service,
    spreadsheet_id: str,
    ranges: list[str],
) -> dict[str, list[list[str]]]:
    """Read multiple ranges in a single API call."""
    result = (
        service.spreadsheets()
        .values()
        .batchGet(
            spreadsheetId=spreadsheet_id,
            ranges=ranges,
            valueRenderOption="FORMATTED_VALUE",
        )
        .execute()
    )

    data = {}
    for value_range in result.get("valueRanges", []):
        range_name = value_range["range"]
        data[range_name] = value_range.get("values", [])
    return data
```

### Typed Data Mapping

```python
from pydantic import BaseModel, Field
from typing import Optional
from datetime import date

class SheetRow(BaseModel):
    """Map spreadsheet columns to typed Python objects."""
    protocolo: str = Field(description="Column A: Protocol number")
    nome: str = Field(description="Column B: Full name")
    cpf: str = Field(description="Column C: CPF")
    data_nascimento: Optional[date] = Field(description="Column D: Birth date")
    status: str = Field(default="pendente", description="Column E: Processing status")
    observacoes: Optional[str] = Field(default=None, description="Column F: Notes")

    @classmethod
    def from_row(cls, row: list[str], header_map: dict[str, int]) -> "SheetRow":
        """Create from spreadsheet row using header-to-index mapping."""
        def get_col(name: str) -> Optional[str]:
            idx = header_map.get(name)
            if idx is not None and idx < len(row):
                val = row[idx].strip()
                return val if val else None
            return None

        return cls(
            protocolo=get_col("protocolo") or "",
            nome=get_col("nome") or "",
            cpf=get_col("cpf") or "",
            data_nascimento=get_col("data_nascimento"),
            status=get_col("status") or "pendente",
            observacoes=get_col("observacoes"),
        )

def build_header_map(header_row: list[str]) -> dict[str, int]:
    """Create mapping from column name to index."""
    return {
        col.strip().lower().replace(" ", "_"): idx
        for idx, col in enumerate(header_row)
    }

# Usage
rows = await read_range(service, sheet_id, "Sheet1!A:F")
header_map = build_header_map(rows[0])
data = [SheetRow.from_row(row, header_map) for row in rows[1:]]
```

---

## 3. Writing Data

### batchUpdate — The Right Way

```python
async def batch_write(
    service,
    spreadsheet_id: str,
    updates: list[dict],
    value_input: str = "USER_ENTERED",
) -> dict:
    """Write multiple ranges in a single API call.

    value_input options:
    - RAW: Values stored exactly as provided
    - USER_ENTERED: Parsed as if user typed them (formulas work)
    """
    body = {
        "valueInputOption": value_input,
        "data": [
            {
                "range": update["range"],
                "values": update["values"],
            }
            for update in updates
        ],
    }

    return (
        service.spreadsheets()
        .values()
        .batchUpdate(spreadsheetId=spreadsheet_id, body=body)
        .execute()
    )

# Usage: Update status and observations for multiple rows
updates = [
    {"range": "Sheet1!E2", "values": [["concluído"]]},
    {"range": "Sheet1!F2", "values": [["Processado em 2025-01-15"]]},
    {"range": "Sheet1!E3", "values": [["erro"]]},
    {"range": "Sheet1!F3", "values": [["CPF já cadastrado no SIC"]]},
]
await batch_write(service, sheet_id, updates)
```

### Append Rows

```python
async def append_rows(
    service,
    spreadsheet_id: str,
    range_name: str,
    rows: list[list],
) -> dict:
    """Append rows to the end of a sheet."""
    body = {"values": rows}

    return (
        service.spreadsheets()
        .values()
        .append(
            spreadsheetId=spreadsheet_id,
            range=range_name,
            valueInputOption="USER_ENTERED",
            insertDataOption="INSERT_ROWS",
            body=body,
        )
        .execute()
    )
```

### Update Specific Cells Efficiently

```python
async def update_row_status(
    service,
    spreadsheet_id: str,
    row_number: int,
    status: str,
    message: str = "",
    timestamp: str = "",
):
    """Update status columns for a specific row."""
    updates = [
        {
            "range": f"Sheet1!E{row_number}",
            "values": [[status]],
        },
        {
            "range": f"Sheet1!F{row_number}",
            "values": [[message]],
        },
    ]
    if timestamp:
        updates.append({
            "range": f"Sheet1!G{row_number}",
            "values": [[timestamp]],
        })

    await batch_write(service, spreadsheet_id, updates)
```

---

## 4. Rate Limit Handling

### Google Sheets API Quotas

| Quota | Limit | Per |
|-------|-------|-----|
| Read requests | 60 | minute per user |
| Write requests | 60 | minute per user |
| Total requests | 300 | minute per project |

### Retry with Exponential Backoff

```python
import asyncio
from googleapiclient.errors import HttpError

async def sheets_api_call_with_retry(
    api_call,
    max_retries: int = 5,
    base_delay: float = 1.0,
):
    """Execute Sheets API call with exponential backoff for rate limits."""
    for attempt in range(max_retries + 1):
        try:
            return api_call.execute()
        except HttpError as e:
            if e.resp.status == 429:  # Rate limited
                if attempt == max_retries:
                    raise
                delay = base_delay * (2 ** attempt)
                await asyncio.sleep(delay)
            elif e.resp.status in (500, 503):  # Server error
                if attempt == max_retries:
                    raise
                await asyncio.sleep(base_delay)
            else:
                raise
```

### Batch Strategy for High Volume

```python
async def process_large_dataset(
    service,
    spreadsheet_id: str,
    all_updates: list[dict],
    batch_size: int = 50,
    delay_between_batches: float = 1.0,
):
    """Process large number of updates in controlled batches."""
    for i in range(0, len(all_updates), batch_size):
        batch = all_updates[i : i + batch_size]
        await batch_write(service, spreadsheet_id, batch)

        # Respect rate limits
        if i + batch_size < len(all_updates):
            await asyncio.sleep(delay_between_batches)
```

---

## 5. Formula Preservation

### Reading Without Destroying Formulas

```python
async def read_preserving_formulas(
    service,
    spreadsheet_id: str,
    range_name: str,
) -> tuple[list[list[str]], list[list[str]]]:
    """Read both values and formulas for safe editing."""
    # Get displayed values
    values = await read_range(
        service, spreadsheet_id, range_name,
        value_render="FORMATTED_VALUE"
    )
    # Get formulas
    formulas = await read_range(
        service, spreadsheet_id, range_name,
        value_render="FORMULA"
    )
    return values, formulas

async def safe_write(
    service,
    spreadsheet_id: str,
    range_name: str,
    values: list[list],
    formula_ranges: list[str],
):
    """Write values but skip cells that contain formulas."""
    # Read existing formulas first
    _, formulas = await read_preserving_formulas(
        service, spreadsheet_id, range_name
    )

    # Filter out cells with formulas
    safe_updates = []
    for row_idx, row in enumerate(values):
        for col_idx, value in enumerate(row):
            cell_formula = (
                formulas[row_idx][col_idx]
                if row_idx < len(formulas) and col_idx < len(formulas[row_idx])
                else ""
            )
            if not cell_formula.startswith("="):
                # Safe to write — no formula here
                col_letter = chr(65 + col_idx)  # A, B, C...
                row_num = row_idx + 1
                safe_updates.append({
                    "range": f"Sheet1!{col_letter}{row_num}",
                    "values": [[value]],
                })

    if safe_updates:
        await batch_write(service, spreadsheet_id, safe_updates)
```

---

## 6. Spreadsheet URL Parsing

```python
import re

def parse_spreadsheet_url(url: str) -> dict:
    """Extract spreadsheet ID and optional gid from a Google Sheets URL.

    Supports:
    - https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit#gid=0
    - https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit
    - Plain spreadsheet ID
    """
    # Full URL pattern
    match = re.match(
        r"https://docs\.google\.com/spreadsheets/d/([a-zA-Z0-9-_]+)",
        url,
    )
    if match:
        spreadsheet_id = match.group(1)
        gid_match = re.search(r"gid=(\d+)", url)
        return {
            "spreadsheet_id": spreadsheet_id,
            "gid": int(gid_match.group(1)) if gid_match else 0,
        }

    # Maybe it's already just the ID
    if re.match(r"^[a-zA-Z0-9-_]+$", url):
        return {"spreadsheet_id": url, "gid": 0}

    raise ValueError(f"Cannot parse spreadsheet URL: {url}")
```

---

## 7. Sheet Metadata

```python
async def get_sheet_metadata(
    service,
    spreadsheet_id: str,
) -> dict:
    """Get spreadsheet metadata: title, sheets, columns."""
    result = (
        service.spreadsheets()
        .get(spreadsheetId=spreadsheet_id)
        .execute()
    )

    return {
        "title": result["properties"]["title"],
        "sheets": [
            {
                "title": sheet["properties"]["title"],
                "sheetId": sheet["properties"]["sheetId"],
                "rowCount": sheet["properties"]["gridProperties"]["rowCount"],
                "colCount": sheet["properties"]["gridProperties"]["columnCount"],
            }
            for sheet in result.get("sheets", [])
        ],
    }
```

---

## 8. Integration with Supabase

### Storing Sheets Config Securely

```python
# Encrypt the Sheets URL and service account in Supabase
async def save_sheets_config(
    supabase_client,
    user_id: str,
    sheets_url: str,
    service_account_json: str,
):
    """Save Sheets configuration with encrypted credentials."""
    parsed = parse_spreadsheet_url(sheets_url)

    await supabase_client.rpc("save_encrypted_config", {
        "p_user_id": user_id,
        "p_sheets_url": sheets_url,
        "p_spreadsheet_id": parsed["spreadsheet_id"],
        "p_sa_credentials": service_account_json,  # Will be pgcrypto encrypted
    }).execute()
```

### Sync Pattern: Sheets → Supabase → Portal

```python
async def sync_workflow(
    sheets_service,
    supabase_client,
    spreadsheet_id: str,
):
    """Read from Sheets, store in Supabase, process via RPA."""
    # 1. Read pending rows from Sheets
    rows = await read_range(sheets_service, spreadsheet_id, "Sheet1!A:F")
    header_map = build_header_map(rows[0])
    records = [SheetRow.from_row(row, header_map) for row in rows[1:]]

    # 2. Filter pending records
    pending = [r for r in records if r.status == "pendente"]

    # 3. Store in Supabase for processing
    for record in pending:
        await supabase_client.table("extractions").insert({
            "spreadsheet_id": spreadsheet_id,
            "protocolo": record.protocolo,
            "data": record.model_dump(),
            "status": "pending",
        }).execute()

    # 4. Update Sheet status to "em processamento"
    row_updates = [
        {
            "range": f"Sheet1!E{i + 2}",  # +2 for header + 0-index
            "values": [["em processamento"]],
        }
        for i, r in enumerate(records)
        if r.status == "pendente"
    ]
    if row_updates:
        await batch_write(sheets_service, spreadsheet_id, row_updates)

    return pending
```

---

## 9. Decision Checklist

Before integrating with Google Sheets:

- [ ] **Auth strategy chosen?** (Service Account vs OAuth2)
- [ ] **Credentials stored securely?** (not in code, encrypted)
- [ ] **Spreadsheet ID parsed correctly?** (URL or direct ID)
- [ ] **Column mapping defined?** (header names → Pydantic model)
- [ ] **Using batchUpdate?** (not cell-by-cell writes)
- [ ] **Rate limits handled?** (retry with backoff)
- [ ] **Formulas preserved?** (not overwriting user formulas)
- [ ] **Error handling for API failures?** (429, 500, 503)
- [ ] **Batch size configured?** (50-100 updates per call)
- [ ] **Status tracking in sheet?** (update rows after processing)

---

## 10. Anti-Patterns

| ❌ Don't | ✅ Do |
|----------|-------|
| Write cell-by-cell | Use `batchUpdate` for all writes |
| Ignore rate limits | Implement exponential backoff |
| Hardcode column positions | Map headers dynamically |
| Store SA key in code | Use env vars or Vault |
| Overwrite formula cells | Read formulas first, skip them |
| Use `RAW` input for user data | Use `USER_ENTERED` (respects formats) |
| Read entire sheet every time | Read specific ranges |
| Process all rows at once | Batch with delays between |
| Skip error handling on API calls | Wrap every call with retry |

---

> **Remember:** Google Sheets is often the user's primary interface. Treat it with respect — don't destroy formulas, don't ignore formatting, and always update status columns so the user knows what happened. The API is powerful but rate-limited — batch everything, retry gracefully, and be a good API citizen.
