# GHASTriage

A command-line tool that walks every repository owned by a GitHub user and emits
a prioritized **GitHub Advanced Security (GHAS)** triage report — both Markdown
and HTML — so you can see at a glance where to focus your remediation effort
across all your repos.

It scans for open alerts from the three GHAS sources:

- **Dependabot** — vulnerable dependencies
- **Code scanning** — CodeQL / SARIF findings
- **Secret scanning** — leaked credentials

Repos are ranked by a severity-weighted **risk score**, so the worst offenders
surface at the top of the report. Sources that are disabled or unavailable on a
repo (e.g. code scanning not enabled, GHAS not licensed on a private repo) are
listed in a separate **Coverage gaps** section, so they don't drown out real
findings.

---

## Requirements

- **Python 3.9+** (stdlib only — no `pip install` needed).
- **[GitHub CLI](https://cli.github.com/) (`gh`)** authenticated with sufficient
  scopes. `gh auth login` with the default `repo` scope is enough for personal
  repos. For organization repos add `read:org`.

The tool shells out to `gh api`, so authentication is whatever `gh` already has
configured — no tokens to plumb through environment variables.

## Install

```bash
git clone https://github.com/rod-trent/GHASTriage.git
cd GHASTriage
```

That's it. The script is a single file with no dependencies.

## Usage

Scan all repos you own and write `security-report.md` + `security-report.html`
to the current directory:

```bash
python triage.py
```

Scan a different user or organization:

```bash
python triage.py --owner some-org
```

### Flags

| Flag | Default | Description |
|---|---|---|
| `--owner` | `@me` (authenticated user) | GitHub user or org to scan |
| `--output-dir` | `.` | Directory to write the two reports into |
| `--include-archived` | off | Include archived repos |
| `--include-forks` | off | Include forks |
| `--workers` | `6` | Parallel repo fetches (raise for speed, lower if rate-limited) |

Output is written to:

- `<output-dir>/security-report.md`
- `<output-dir>/security-report.html`

The HTML report is self-contained (embedded CSS, no JavaScript, no external
fetches) — open it directly in a browser or share the file.

## What the report contains

**Summary cards** — total repos scanned, repos with open alerts, total open
alerts, and counts by severity.

**Coverage by source** — how many of your repos have each GHAS source enabled
vs. disabled. This is the quickest way to spot where you should be turning on
secret scanning or code scanning.

**Top focus areas** — top 20 repos ranked by risk score, with a severity
breakdown per repo. This is the "where do I start" table.

**Per-repo detail** — collapsible section per affected repo, listing every open
Dependabot advisory (with package + GHSA link), every code-scanning finding
(with rule + path + message), and every secret-scanning hit (with type +
created date + link to the alert).

**Coverage gaps** — every (repo, source) pair where the source is not enabled
or returned an error. Useful for driving "enable Dependabot on these 15 repos"
follow-up work.

## How the risk score works

Each open alert contributes points based on severity:

| Severity | Points |
|---|---:|
| critical | 10 |
| high | 5 |
| medium | 2 |
| low | 1 |
| secret (no GitHub severity) | 5 (treated as high) |

A repo's risk score is the sum of its alerts' points. Repos are sorted
descending by risk score (then by total alert count, then alphabetically).

This weighting is opinionated but easy to change — see `SEVERITY_WEIGHTS` near
the top of [triage.py](triage.py).

## Notes on GHAS availability

- **Public repos** — Dependabot, code scanning, and secret scanning are all
  available for free. They still need to be **enabled** on each repo (most are
  one-click in repo settings).
- **Private repos** — Dependabot alerts are free. Code scanning and secret
  scanning require a paid **GitHub Advanced Security** license. On private
  repos without GHAS, those sources will appear under **Coverage gaps** with a
  "disabled" / 404 / 403 status rather than as actionable alerts.
- **Organizations** — your `gh` token needs `read:org` to enumerate
  organization-owned repos via `--owner some-org`. The tool only reads alerts
  it has permission to see.

## Privacy

The generated `security-report.md` and `security-report.html` files contain
real vulnerability details from your repositories — including names of private
repos and the specific advisories affecting them. Treat them like secrets:

- They are gitignored in this repo by default. **Do not commit them.**
- Don't paste the HTML into a public chat or issue tracker.
- If you share the report, share it with the same audience you'd share the
  underlying repo access with.

## Example output

A scan of an account with 73 repos and 47 open alerts produces output like:

```text
Listing repos for rod-trent...
Found 73 repos. Scanning alerts...
  [1/73] rod-trent/AzureSentinelMisc - 0 alerts
  [2/73] rod-trent/AgentPlatform - 1 alerts
  ...
  [73/73] rod-trent/WeatherData - 0 alerts

Wrote security-report.md
Wrote security-report.html
```

The Markdown report's "Top focus areas" table:

```text
| # | Repo                          | Risk | Total | Crit | High | Med | Low | Secrets |
|---|-------------------------------|-----:|------:|-----:|-----:|----:|----:|--------:|
| 1 | rod-trent/JunkDrawer          |   50 |    26 |    0 |    0 |  24 |   2 |       0 |
| 2 | rod-trent/GithubCopilot       |   43 |    18 |    0 |    4 |   9 |   5 |       0 |
| 3 | rod-trent/AgentPlatform       |    2 |     1 |    0 |    0 |   1 |   0 |       0 |
```

— tells you exactly where to start.

## License

MIT — see [LICENSE](LICENSE).
