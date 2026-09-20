# surfacediff

**Snapshot and diff your external attack surface.**

ProjectDiscovery gives you recon tools — `subfinder`, `httpx`, `naabu`, `nuclei`.
surfacediff is the layer none of them ship: pipe any tool output in, get immutable
content-addressed snapshots back, and diff them **field-by-field** — new hosts,
dead hosts, status changes, title changes, new ports. Recon becomes monitoring.

Zero dependencies. stdlib only. One `pip install`, no supply chain to babysit.

## The pipelines it was built for

```bash
# subdomains: what's new since last run?
subfinder -d example.com -silent | surfacediff snap -l subs --source subfinder
surfacediff diff -l subs

# live web surface: what changed?
httpx -l subs.txt -json -silent | surfacediff snap -l web --source httpx
surfacediff diff -l web

# ports: did something new open?
naabu -host example.com -json -silent | surfacediff snap -l ports --source naabu
surfacediff diff -l ports
```

`diff` exits **1 when the surface changed** — so your nightly cron becomes an alert:

```bash
# crontab: nightly surface watch
0 6 * * * subfinder -d example.com -silent | surfacediff snap -l subs --source subfinder && surfacediff diff -l subs || notify "attack surface changed"
```

Commit the `.surfacediff/` directory to a private repo and you have versioned
attack-surface history for free — snapshots are plain JSONL, sorted, deterministic,
diffable with plain `git diff`.

## Output

```
label web: +1 -0 ~2
20260916T013000Z.jsonl -> 20260916T020000Z.jsonl
  + url|https://staging.example.com
  ~ url|https://mail.example.com
      status_code: 200 -> 403
      title: removed
```

Machine output with `--json`. Change detection is content-hashed — identical
assets never produce false diffs, reordered attrs don't either.

## Commands

| Command | What it does |
|---|---|
| `surfacediff snap -l LABEL [-i FILE] [--tag K=V] [--source httpx]` | store a snapshot (stdin default), update `current` |
| `surfacediff diff -l LABEL [--json] [--from REF]` | diff last two snapshots; **exit 1 = changes** |
| `surfacediff show -l LABEL` | list snapshot history with timestamps + counts |

Global: `--dir` (store location, default `./.surfacediff`, env `SURFACEDIFF_DIR`), `--version`.

## Input adapters (auto-detected, per line)

| Input | Becomes |
|---|---|
| `subfinder` / `dnsx` / `amass` plain lines | `host` records |
| `httpx -json` lines | `url` records + attrs (status, title, webserver, tech, cdn, asn) |
| `naabu -json` lines | `port` records |
| raw URLs | `url` records (canonicalized: default ports dropped, fragments stripped) |

Malformed lines are skipped with a warning to stderr — never silently.

## What v0 is honest about

- **Pure diffing.** surfacediff makes zero network calls — it shapes and compares
  whatever your recon tools found. The scanning stays with the tools that own it.
- **First snapshot = baseline.** Everything is "added" against an empty store;
  changes appear from the second snapshot onward.
- Store is local filesystem. Remote sync (S3/GCS backends) is future work —
  until then, git is the sync.

## Install

```bash
pip install surfacediff     # or: pip install git+https://github.com/cyeezy08/surfacediff
```

## Part of the Leviathan build

This is the connective tissue of a bigger build-in-public track: continuous
external validation for teams that can't afford six figures. See
[leviathan-core](https://github.com/cyeezy08/leviathan-core) for the exposure
correlation engine.


## The Leviathan stack

| tool | layer |
|---|---|
| surfacediff | what changed on my surface since last run (snapshot/diff) |
| hostage | which of my subdomains are dangling or claimable |
| leviathan-core | which CVEs actually hit my assets, with reasons |

Recon feeds the surface, takeover scanning tests it, the exposure queue
explains it. One coherent pipeline, three focused tools.

## License

MIT
