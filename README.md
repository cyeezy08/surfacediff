```text
  LEVIATHAN X / SURFACEDIFF
  snapshot  [  before  ] ------- delta -------> [  after  ]
                   + added | - removed | ~ changed
```

# surfacediff

**Reproducible snapshots and field-level changes for authorized asset inventories.**

Part of the **Leviathan X** tooling suite. `surfacediff` normalizes supported structured and text inputs, records versioned snapshots and compares field-level differences. It does not discover or scan internet assets by itself.

[![CI](https://github.com/cyeezy08/surfacediff/actions/workflows/ci.yml/badge.svg)](https://github.com/cyeezy08/surfacediff/actions/workflows/ci.yml)
[Python packaging](pyproject.toml) | [Tests](tests) | [MIT license](LICENSE)

> Operate only on inventory data collected with appropriate authorization. No live scanning or automated provider integration is implied by this package.

---

## Real CLI capture

The [offline demo workflow](https://github.com/cyeezy08/surfacediff/actions/workflows/offline-demo.yml) runs the installed Python CLI without network input and publishes its actual help output as a terminal SVG and raw text. The diagram below describes the architecture; it is not a capture of a live scan. No fabricated before/after data is shown in the capture.

---

## Why surfacediff?

Discovery and monitoring tools answer different questions. A source may identify an asset today, but analysts also need a reproducible record of what changed between observations.

`surfacediff` is the missing bridge:
* **Pipe any recon tool straight in:** Automatically detects JSON and text streams from `subfinder`, `httpx`, `naabu`, and raw URL lists.
* **Immutable, content-addressed snapshots:** Stored as deterministic, sorted JSONL in `./.surfacediff/`. 100% git-diffable.
* **Granular field-by-field diffing:** Detects not just new/dead hosts, but status code flips (`200 -> 403`), title updates, added tech tags, and newly exposed ports.
* **Alerting by design:** `surfacediff diff` exits with **code 1 when changes are detected** — instant exit-code alerting for cron jobs, CI/CD, and bash pipelines.
* **Standard-library runtime:** No third-party runtime dependencies are declared. Deployment and input provenance still need review; no software is risk-free.

---

## Architecture & Pipeline Flow

```mermaid
flowchart LR
    subgraph Inputs ["Recon Inputs (STDOUT)"]
        S["subfinder"]
        H["httpx -json"]
        N["naabu -json"]
        D["dnsx / amass / raw"]
    end

    subgraph Core ["surfacediff Engine"]
        AD["Auto-Detect Adapters"]
        NORM["Canonicalization & Content Hashing"]
        STORE[(".surfacediff/ Snapshots (JSONL)")]
        DIFF["Field Diff Engine"]
    end

    subgraph Outputs ["Downstream Actions"]
        OUT["Human / JSON Diff Report"]
        EXIT["Exit Code 1 (Alert Trigger)"]
        GIT["Git Commit / History"]
    end

    Inputs -->|Piped stream| AD
    AD --> NORM
    NORM --> STORE
    STORE --> DIFF
    DIFF --> OUT
    DIFF --> EXIT
    STORE -.-> GIT
```

---

## Quickstart

### Installation

```bash
# Direct install from GitHub
pip install git+https://github.com/cyeezy08/surfacediff.git

# Or clone and install editable
git clone https://github.com/cyeezy08/surfacediff.git
cd surfacediff && pip install -e .
```

### 1. Snapshot your live attack surface

```bash
# Capture subdomains
subfinder -d example.com -silent | surfacediff snap -l subs --source subfinder

# Capture full HTTP web assets (status, title, technology headers)
httpx -l subs.txt -json -silent | surfacediff snap -l web --source httpx

# Capture open ports
naabu -host example.com -json -silent | surfacediff snap -l ports --source naabu
```

### 2. Diff against your baseline

```bash
# Diff the last two snapshots
surfacediff diff -l web
```

**Terminal output:**
```text
label web: +1 -0 ~2
20260928T120000Z.jsonl -> 20260929T120000Z.jsonl
  + url|https://internal-dev.example.com
  ~ url|https://admin.example.com
      status_code: 403 -> 200
      title: "Forbidden" -> "Dashboard Login"
  ~ url|https://api.example.com
      tech: added [FastAPI, Swagger UI]
```

### 3. Inspect history

```bash
surfacediff show -l web
```

---

## Input Adapters (Zero Configuration)

`surfacediff` parses line-delimited streams automatically:

| Tool / Format | Auto-Detected Record | Tracked Attributes |
|---|---|---|
| `subfinder`, `dnsx`, `amass` | `host` | FQDN, root domain |
| `httpx -json` | `url` | URL, status code, title, webserver, tech stack, CDN, ASN |
| `naabu -json` | `port` | IP/Host, port number, protocol |
| Raw URLs (`https://...`) | `url` | Canonicalized URLs (stripped fragments, default ports normalized) |

---

## Automated Cron & Alerting Workflows

Because `surfacediff diff` exits with code **1** on changes and code **0** when nothing changed, continuous monitoring is a 1-liner:

### Nightly Bash Cron Job

```bash
#!/usr/bin/env bash
# Runs at midnight: snap, diff, and notify only on genuine delta
subfinder -d target.com -silent | surfacediff snap -l target-subs --source subfinder

if ! surfacediff diff -l target-subs --json > /tmp/delta.json; then
    # Surface changed! Send alert
    curl -X POST -H "Content-Type: application/json" \
         -d @/tmp/delta.json \
         "$SLACK_OR_DISCORD_WEBHOOK_URL"
fi
```

### GitHub Actions Scheduled Watcher (`.github/workflows/surface-watch.yml`)

```yaml
name: Nightly Attack Surface Delta
on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:

jobs:
  recon-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Tools
        run: |
          go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
          pip install git+https://github.com/cyeezy08/surfacediff.git

      - name: Snapshot & Diff
        run: |
          subfinder -d example.com -silent | surfacediff snap -l prod-subs
          surfacediff diff -l prod-subs || echo "CHANGES_DETECTED=true" >> $GITHUB_ENV

      - name: Commit Updated Snapshot State
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .surfacediff/
          git commit -m "chore(surface): update snapshots [skip ci]" || exit 0
          git push
```

---

## Command Reference

```text
surfacediff [OPTIONS] COMMAND [ARGS]...

Commands:
  snap   Capture a snapshot from stdin or file and update 'current'
  diff   Diff the two most recent snapshots (or between specific refs)
  show   Display snapshot history, timestamps, and record counts

Global Options:
  --dir TEXT      Directory for snapshot storage [default: ./.surfacediff]
  --version       Show version and exit
  --help          Show this message and exit
```

### `surfacediff snap` Options
* `-l, --label TEXT` **(Required)** Logical category (e.g., `prod-web`, `api-hosts`, `corp-dns`).
* `-i, --input PATH` Path to input file (defaults to STDIN).
* `--source TEXT` Optional source tag (e.g., `httpx`, `subfinder`, `naabu`).
* `--tag KEY=VALUE` Custom key-value metadata tags.

### `surfacediff diff` Options
* `-l, --label TEXT` **(Required)** Label to diff.
* `--from REF` Compare against a specific snapshot timestamp rather than the immediate predecessor.
* `--json` Emit structured JSON for machine parsing / webhook feeds.

---

## Leviathan X tooling

These tools remain independent, reusable executables. They are not claimed to already operate as a unified hosted service.

* **[surfacediff](https://github.com/cyeezy08/surfacediff):** normalized snapshots and change reports.
* **[HostageLVX](https://github.com/cyeezy08/HostageLVX):** dangling DNS and provider fingerprint classification.
* **[FenrirLVX](https://github.com/cyeezy08/FenrirLVX):** WordPress component evidence and offline advisory correlation.

### Data quality boundaries

A missing snapshot, malformed input, a source outage or absent history is **unknown**, not a clean asset. Every comparison is limited by the provided observations. Treat example outputs as illustrative unless backed by actual fixtures and tests.

---

## License

Licensed under the **MIT License**. Published as **Leviathan X** tooling.
