# Tenable Security Center — System Log Client

A Python CLI tool for fetching and filtering login/authentication events from the [Tenable Security Center](https://www.tenable.com/products/security-center) REST API (`/system/log`). Supports date-range queries, severity and module filters, automatic pagination, keyword search, and export to CSV or JSON.

## Requirements

- Python 3.10+
- `requests` and `urllib3`

```bash
pip install requests urllib3
```

## Installation

No package installation is required — download the script and run it directly:

```bash
chmod +x system_logs
./system_logs --host sc.corp --access-key ABC --secret-key XYZ
# or
python system_logs --host sc.corp --access-key ABC --secret-key XYZ
```

## Quick Start

```bash
# Default: last 100 auth events across all time
python system_logs --host 192.168.1.100 --access-key ABC --secret-key XYZ

# Use environment variables to avoid repeating credentials
export SC_HOST=sc.internal
export SC_ACCESS_KEY=abc
export SC_SECRET_KEY=xyz
python system_logs --limit 50
```

---

## Configuration

### Credentials

Credentials can be supplied as CLI flags or environment variables. CLI flags take priority.

| CLI flag | Environment variable | Description |
|---|---|---|
| `--host` | `SC_HOST` | Security Center hostname or IP |
| `--access-key` | `SC_ACCESS_KEY` | API access key |
| `--secret-key` | `SC_SECRET_KEY` | API secret key |

> **Generating API keys in SC:** Log in to Security Center → **Users → API Keys → Generate**.

### SSL Verification

SSL certificate verification is **disabled by default** — common for on-premises SC installs with self-signed certificates. Enable it when a CA-signed certificate is in place:

```bash
python system_logs --verify-ssl --host sc.corp --access-key A --secret-key B
```

---

## CLI Reference

```
usage: system_logs [-h]
                   [--host HOST] [--access-key KEY] [--secret-key KEY] [--verify-ssl]
                   [--date-from YYYYMM] [--date-to YYYYMM]
                   [--limit N] [--page-size N]
                   [--module MODULE] [--severity {INFO,WARNING,CRITICAL}]
                   [--keyword KEYWORD] [--filter KEY=VALUE]
                   [--output FILE] [--display-limit N] [--no-stats] [--quiet]
                   [--list-modules]
```

### Connection

| Flag | Default | Description |
|---|---|---|
| `--host` | `$SC_HOST` | SC hostname or IP address |
| `--access-key` | `$SC_ACCESS_KEY` | SC API access key |
| `--secret-key` | `$SC_SECRET_KEY` | SC API secret key |
| `--verify-ssl` | `false` | Verify the server's SSL certificate |

### Date Range

| Flag | Default | Description |
|---|---|---|
| `--date-from` | `all` | Start month as `YYYYMM`, or `all` to query across all time |
| `--date-to` | same as `--date-from` | End month as `YYYYMM`; triggers month-by-month iteration |

When a range is given, the tool queries each month in sequence and merges the results into a single list before applying `--limit`.

### Volume / Pagination

| Flag | Default | Description |
|---|---|---|
| `--limit` | `100` | Maximum total entries to retrieve; `0` = unlimited |
| `--page-size` | `100` | Entries per API request (max recommended: `500`) |

### Filters

All active filters are AND-ed together before being sent to the API.

| Flag | Default | Description |
|---|---|---|
| `--module` | `auth` | Log module to query: `auth`, `apikey`, `admin`, etc. |
| `--severity` | _(all)_ | One of `INFO`, `WARNING`, or `CRITICAL` |
| `--keyword` | _(none)_ | Free-text keyword — matches usernames, IP addresses, etc. |
| `--filter KEY=VALUE` | _(none)_ | Raw SC filter pair; flag is repeatable for multiple filters |

Use `--list-modules` to print all available module names and exit.

### Output

| Flag | Default | Description |
|---|---|---|
| `--output FILE` | _(none)_ | Save results to file; extension sets format (`.csv` or `.json`) |
| `--display-limit` | `50` | Max rows shown in the terminal table; `0` = show all |
| `--no-stats` | _(off)_ | Skip the severity / user / type statistics summary |
| `--quiet` | _(off)_ | Suppress progress output; print only the final entry count |

---

## Usage Examples

```bash
# ── Basic ────────────────────────────────────────────────────────────────────

# All auth events, last 100 entries (default)
python system_logs --host sc.corp --access-key A --secret-key B

# Failed logins only
python system_logs --host sc.corp --access-key A --secret-key B \
    --severity WARNING

# ── Date filters ─────────────────────────────────────────────────────────────

# Single month
python system_logs --host sc.corp --access-key A --secret-key B \
    --date-from 202505

# Date range (iterates month by month, merges results)
python system_logs --host sc.corp --access-key A --secret-key B \
    --date-from 202501 --date-to 202505

# ── Keyword & module ─────────────────────────────────────────────────────────

# All events for a specific user
python system_logs --host sc.corp --access-key A --secret-key B \
    --keyword jdoe

# Query the admin module
python system_logs --host sc.corp --access-key A --secret-key B \
    --module admin --limit 200

# ── Volume / pagination ───────────────────────────────────────────────────────

# 1 000 events fetched in batches of 200
python system_logs --host sc.corp --access-key A --secret-key B \
    --limit 1000 --page-size 200

# Unlimited — retrieve everything
python system_logs --host sc.corp --access-key A --secret-key B \
    --limit 0

# ── Export ────────────────────────────────────────────────────────────────────

# Save to CSV
python system_logs --host sc.corp --access-key A --secret-key B \
    --output events.csv

# Save to JSON
python system_logs --host sc.corp --access-key A --secret-key B \
    --output events.json

# ── Combined example ─────────────────────────────────────────────────────────

# Failed logins for jdoe in May 2025, exported to CSV
python system_logs --host sc.corp --access-key A --secret-key B \
    --date-from 202505 --date-to 202505 \
    --severity WARNING \
    --keyword jdoe \
    --output failed_logins_may2025.csv

# ── Utility ───────────────────────────────────────────────────────────────────

# List all available log modules
python system_logs --host sc.corp --access-key A --secret-key B \
    --list-modules
```

---

## Output Format

### Terminal Table

A colour-coded table is printed after each fetch. Severity values are highlighted (green / yellow / red).

```
────────────────────────────────────────────────────────────────────────────────────────────
TIMESTAMP                           USER            SEVERITY    MESSAGE
────────────────────────────────────────────────────────────────────────────────────────────
2024-05-10 09:23:45 UTC             jdoe            WARNING     Failed login attempt from 10.0.0.1
2024-05-10 09:45:00 UTC             admin           INFO        Successful login from 10.0.0.5
────────────────────────────────────────────────────────────────────────────────────────────
```

### Statistics Summary

Printed after the table:

- **Severity breakdown** — bar chart of INFO / WARNING / CRITICAL counts
- **Top 10 users** — ranked by number of events
- **Event types** — count per type string

Suppress with `--no-stats`.

### CSV / JSON Schema

Both file formats include the six raw log fields returned by the SC API:

| Field | Description |
|---|---|
| `timestamp` | Event timestamp |
| `user` | Username associated with the event |
| `module` | Log module name (e.g. `auth`) |
| `severity` | `INFO`, `WARNING`, or `CRITICAL` |
| `type` | Event type string |
| `message` | Full log message text |

---

## Troubleshooting

**`Missing required argument(s): --host`**
Set `SC_HOST` as an environment variable or pass `--host` explicitly.

**SSL certificate errors**
Omit `--verify-ssl` (the default) for self-signed certificates. Add `--verify-ssl` only when the SC server has a valid CA-signed certificate.

**HTTP 401 / 403**
Verify the access key and secret key. In SC, API keys are generated under **Users → API Keys**.

**Empty results / fewer rows than expected**
- Try `--limit 0` to remove the row cap.
- Use `--date-from all` to widen the time window.
- Run `--list-modules` to confirm the module name is spelled correctly.
- Check that your API key has permission to read system logs in SC.
