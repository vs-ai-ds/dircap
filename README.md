# dircap

[![PyPI](https://img.shields.io/pypi/v/dircap)](https://pypi.org/project/dircap/)
![CI](https://github.com/vs-ai-ds/dircap/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/pypi/pyversions/dircap)

Set folder size caps and get warned when you exceed them.  
Works on **Windows / macOS / Linux**.

`dircap` is a small, automation-friendly CLI that checks configured folders against size budgets and reports:

- **OK** — under warn threshold
- **WARN** — above warn threshold
- **OVER** — at/over 100% of limit

It is designed to be:
- simple to configure
- safe to run daily
- reliable in schedulers / cron
- easy to extend via scripts (not plugins)

---

## Why dircap exists

Disk usage grows quietly until something breaks:
- downloads pile up
- build artifacts accumulate
- logs grow without limits

`dircap` solves this with:
- explicit per-folder caps
- clear WARN / OVER states
- exit codes for automation
- optional notifications via scripts

No daemon. No GUI. No background watcher.

---

## Installation

### Option A — pipx (recommended)

```bash
pipx install dircap
```

### Option B — pip

```bash
pip install dircap
```

### Option C — from source (development / customization)

Use this option when you have the repo locally (downloaded/checked out from GitHub).

```bash
cd <path-to-repo>/dircap
python -m venv .venv
# or on Windows:
py -m venv .venv
```

Activate the venv:

**Windows (PowerShell)**

```powershell
.venv\Scripts\Activate.ps1
```

**Windows (cmd.exe)**

```bat
.venv\Scripts\activate.bat
```

**macOS / Linux**

```bash
source .venv/bin/activate
```

Install:

Runtime dependencies only:

```bash
pip install -e .
```

Development (runtime + dev deps like pytest/ruff):

```bash
pip install -e ".[dev]"
```

Verify:

```bash
dircap --help
dircap --version
```

## Quick start

Run `dircap init` once to create the default config (safe to run again).

```bash
dircap init
dircap where
dircap validate
dircap report
dircap check
```

Example output:

```bash
$ dircap check
```

```
                                  dircap
┏━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━┳━━━━━━━┳━━━━━┳━━━━━━━━┓
┃ Name      ┃ Path                         ┃ Used  ┃ Limit ┃  %  ┃ Status ┃
┡━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━╇━━━━━━━╇━━━━━╇━━━━━━━━┩
┃ Downloads ┃ /home/user/Downloads         ┃ 4.2GB ┃ 5.0GB ┃ 84% ┃ OK     ┃
┃ Builds    ┃ /home/user/projects/build    ┃ 9.8GB ┃ 10GB  ┃ 98% ┃ WARN   ┃
┃ Logs      ┃ /home/user/app/logs          ┃ 12GB  ┃ 10GB  ┃ 120%┃ OVER   ┃
└───────────┴──────────────────────────────┴───────┴───────┴─────┴────────┘

OK: 1  WARN: 1  OVER: 1
```

## Configuration

### Config file location

dircap init creates the config file if it doesn't exist:

- Windows: `%APPDATA%\dircap\config.toml`
- macOS / Linux: `~/.config/dircap/config.toml`

Running `dircap init` again is safe — it will not overwrite an existing config.

### Example config

```toml
[settings]
default_warn_pct = 85
follow_symlinks = false
max_depth = 50
exclude_dirnames = [".git", "node_modules", "__pycache__", ".venv", "dist", "build"]

[action]
# Actions are intentionally left empty in this project.
# For alerts/notifications, use the scheduler scripts in examples/
# so ONE summary notification per run is sent (not one per folder).
on_warn = ""
on_over = ""

[[budgets]]
name = "Downloads"
path = "~/Downloads"
limit = "5GB"
warn_pct = 80
```

Add more folders by adding more [[budgets]] blocks.

### Notes on paths

`~` and environment variables are expanded.

Paths that don't exist are not a hard error (useful for removable/mounted drives), but `dircap validate` will warn.

```bash
# Windows tip:
# Use forward slashes or single quotes to avoid backslash escaping issues.
path = "C:/Users/usr/Downloads"
# or:
path = 'C:\Users\usr\Downloads'
```

### Notes on excludes

`exclude_dirnames` is a directory-name filter (fast).  
If a folder matches by name, it is skipped everywhere in the scan.

## Commands

**dircap init**  
Creates the config file if it doesn't exist.

**dircap where**  
Prints the config file path.

**dircap validate**  
Validates the config without scanning folders.

**dircap report**  
Scans folders and prints a readable table (sorted by severity: OVER → WARN → OK).

**dircap check**  
Same as report, plus exit codes for automation.

Exit codes:
- `0` — all OK
- `1` — at least one WARN
- `2` — at least one OVER

Common options:
- `--summary` → print only OK/WARN/OVER counts
- `--json <file>` → write results to JSON
- `--json-verbose` → write structured JSON including summary + warnings + results
- `--no-actions` → skip config actions (recommended; notifications are done via scripts)

## Automation (recommended setup)

A clean pattern is to keep scripts + logs in a user folder:

- Windows: `%USERPROFILE%\dircap\`
- macOS / Linux: `~/dircap/`

Typical contents:

```
run-dircap.bat / run-dircap.sh     # copied from examples/
dircap-last.txt                    # generated after running dircap
dircap-last.json                   # generated after running dircap
```

This repository provides ready-to-use scripts in `examples/`.

### Windows Task Scheduler

**1) Copy example scripts (PowerShell)**

```powershell
mkdir "$env:USERPROFILE\dircap" -Force
copy examples\run-dircap.bat "$env:USERPROFILE\dircap\"
copy examples\send-email.py "$env:USERPROFILE\dircap\"
```

Notes:
- `$env:USERPROFILE` is a PowerShell environment variable that points to your home folder.
- In cmd.exe/batch scripts the equivalent is `%USERPROFILE%`.

Edit `run-dircap.bat` and set the `REPO=...` path.

**2) Create scheduled task**

Action → Start a program

Program/script:

```
C:\Windows\System32\cmd.exe
```

Arguments:

```bat
/c "%USERPROFILE%\dircap\run-dircap.bat"
```

Start in (recommended):

```
%USERPROFILE%\dircap
```

Logs will be written to:
- `%USERPROFILE%\dircap\dircap-last.txt`
- `%USERPROFILE%\dircap\dircap-last.json`

### macOS / Linux (cron)

**1) Copy example scripts**

```bash
mkdir -p ~/dircap
cp examples/run-dircap.sh ~/dircap/
cp examples/send-email.py ~/dircap/
chmod +x ~/dircap/run-dircap.sh
```

Edit `run-dircap.sh` and set the `REPO=...` path.

**2) Add cron entry**

```cron
0 9 * * * /bin/sh /home/<you>/dircap/run-dircap.sh
```

Logs will be written to:
- `~/dircap/dircap-last.txt`
- `~/dircap/dircap-last.json`

### Email notifications (optional)

Email sending is handled by `examples/send-email.py` (one summary email per run, only when WARN/OVER).

#### Configure SMTP (environment variables)

**Windows (PowerShell)**

```powershell
setx DIRCAP_EMAIL_TO "you@example.com"
setx DIRCAP_EMAIL_FROM "from@example.com"
setx DIRCAP_SMTP_SERVER "mail.example.com"
setx DIRCAP_SMTP_PORT "465"
setx DIRCAP_SMTP_USER "usr@example.com"
setx DIRCAP_SMTP_PASS "EMAIL_PASSWORD"

# Optional flags
setx EMAIL_USE_SSL "true"
setx EMAIL_USE_TLS "false"
```

After `setx`, close and reopen your terminal (or log out/in) so scheduled tasks can see the new variables.

**macOS / Linux (bash/zsh)**

Add to `~/.bashrc` or `~/.zshrc`:

```bash
export DIRCAP_EMAIL_TO="you@example.com"
export DIRCAP_EMAIL_FROM="from@example.com"
export DIRCAP_SMTP_SERVER="mail.example.com"
export DIRCAP_SMTP_PORT="465"
export DIRCAP_SMTP_USER="usr@example.com"
export DIRCAP_SMTP_PASS="EMAIL_PASSWORD"

# Optional flags
export EMAIL_USE_SSL="true"
export EMAIL_USE_TLS="false"
```

Then reload your shell:

```bash
source ~/.bashrc
# or
source ~/.zshrc
```

## Architecture (high level)

```
src/dircap/
├── cli.py      # CLI commands, output, exit codes
├── config.py   # Config discovery + parsing
├── scan.py     # Directory traversal + size calculation
└── format.py   # Size parsing + formatting
```

### Non-goals

- GUI or tray app
- background daemon
- live file watching
- built-in Slack/Teams integrations (use scripts)

## Development

Create a venv and install editable (with dev dependencies):

```bash
cd dircap
python -m venv .venv
# Windows (PowerShell): .venv\Scripts\Activate.ps1
# macOS/Linux: source .venv/bin/activate
pip install -e ".[dev]"
```

Run tests:

```bash
pytest
```

Lint/format:

```bash
ruff format .
ruff check .
```

## License

MIT
