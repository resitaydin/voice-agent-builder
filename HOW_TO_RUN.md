# How to run the builder

Everything from a fresh machine to a finished build. Run it from the repository folder.

**In short**

```
git clone https://github.com/resitaydin/voice-agent-builder.git && cd voice-agent-builder
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt        # Windows: python -m venv .venv ; .venv\Scripts\pip install -r requirements.txt
# put FREYA_API_KEY and FREYA_BANK_TEAM_KEY in env.sh (copy kit/env.example), then: source env.sh
# log in to Claude (or set ANTHROPIC_API_KEY), see step 3 below
.venv/bin/python builder/build.py {case_folder} {mock_url}                  # Windows: .venv\Scripts\python ...
```

## Setup from scratch

Tested from a fresh copy of the repository on Windows (`git clone`, Python 3.14), on Linux (Ubuntu 24.04, Python 3.12) and on macOS (Python 3.12).

**1. Install**

| | macOS / Linux (bash, zsh) | Windows (PowerShell) |
|---|---|---|
| Python 3.10 or newer (tested on 3.12 and 3.14) | `python3 --version` (Ubuntu and Debian also need `sudo apt install python3-venv`. The macOS `python3` is 3.9, which is too old: pip then says `No matching distribution found for claude-agent-sdk`. Use `brew install python@3.12` and `python3.12`, or `uv venv -p 3.12 .venv`) | `python --version` |
| Clone | `git clone https://github.com/resitaydin/voice-agent-builder.git && cd voice-agent-builder` | the same |
| Virtual environment | `python3 -m venv .venv` | `python -m venv .venv` |
| Dependencies | `.venv/bin/pip install -r requirements.txt` | `.venv\Scripts\pip install -r requirements.txt` |

Below, `PY` means `.venv/bin/python` (macOS, Linux) or `.venv\Scripts\python` (Windows). No need to activate the environment.

**2. Keys.** Copy `kit/env.example` to `env.sh` (it is git-ignored) and fill in:

| Variable | Where it comes from |
|---|---|
| `FREYA_API_KEY` | Your Freya workspace, under API keys. It starts with `fak_`. |
| `FREYA_BANK_TEAM_KEY` | Make one up: at least 16 characters. Use the same one every time. |
| `FREYA_API_URL` | Leave as in the file (`https://staging-app.freyavoice.ai`). |

- macOS / Linux: `source env.sh`
- Windows: PowerShell does not read `env.sh`. Set the three variables in the same terminal, for example `$env:FREYA_API_KEY = "fak_..."`.

**3. Claude login.** The builder runs the Claude Agent SDK, which needs a Claude account or an API key. Either log in with the `claude` program that comes with the SDK, or set `ANTHROPIC_API_KEY`:

```
PY -c "import claude_agent_sdk, pathlib; print(pathlib.Path(claude_agent_sdk.__file__).parent / '_bundled')"
# in the folder it prints: claude auth login        (Windows: claude.exe auth login)
# check:                   claude auth status       shows "loggedIn": true
```

**4. Check that it works** (all three should succeed, none starts a build):

```
PY -m pytest -q builder/tests eval             # all tests pass (171 today)
PY eval/contamination.py                       # "0 leaks", exit code 0
PY -c "from builder.blocks.check import freya; ws = freya.workspace_id(); print(freya.api('GET', '/api/v2/workflow', workspaceId=ws, limit=100)['total'])"
                                               # prints a number; the workspace limit is 200 workflows
```

## Run

Use a plain terminal, **not** inside a Claude Code session: variables such as `CLAUDECODE` make the SDK start a nested session. In a Claude Code terminal, clear them first (macOS / Linux: `unset ANTHROPIC_API_KEY CLAUDECODE CLAUDE_CODE_ENTRYPOINT CLAUDE_CODE_SESSION_ID`; PowerShell: `Remove-Item Env:CLAUDECODE, Env:CLAUDE_CODE_ENTRYPOINT, Env:CLAUDE_CODE_SESSION_ID -ErrorAction SilentlyContinue`).

```
PY builder/build.py {case_folder} {mock_url}                       # the final case: a folder of documents and our mock
PY builder/build.py {brief_path} {tool_api_path}                   # a practice brief and the kit's tool API file
PY builder/build.py eval/briefs/v1-raw-a.md kit/tools/openapi.json # example: practice brief A
```

`traces/` and `submission/` are created by the run. Only when you pipe the output through `tee traces/final.log` must `traces/` exist first. The final case is run as `python3 builder/build.py {case_folder} {mock_url}`; `python3` and `PY` are the same when the right Python is first on the path.

- **Mock URL:** `https://the-builder-mock.the-builder-freya.workers.dev`. `{mock_url}/health` must answer `{"ok":true}`.
- **Time:** a build takes about 6 to 11 minutes and stops by itself at 25 minutes (28 minutes at the latest for the last deploy). It uses your Claude account and creates or updates one Freya workflow.
- **What a good run prints at the end:** `created workflow_id=... agent_id=... nodes=... tools=...`, then `end seconds=...` (under 1800), and the exit code is 0.
- **Where the results go:** the phone agent is exported to `submission/<slot>.workflow.json` and the trace is written as JSON lines to `traces/<slot>-<time>.jsonl`. The slot is the brief's file name without `.md`, or the case folder's name. A second run with the same slot updates the same workflow instead of making a new one.
- **Case folder:** it becomes one `brief.md` in the run folder: README and BRIEF first, Word and Excel converted to text, PDF and image files listed for the agent to open. The tool API comes from `{mock_url}/openapi.json`.

**If it fails**

| Message or symptom | Meaning |
|---|---|
| `401` and `invalid JWT` from Freya | `FREYA_API_KEY` is missing or wrong in this terminal. Set it again (step 2). |
| 5xx or a timeout from Freya | Not our crash. Wait and run again, with no change. |
| `error code: 1042` from the mock | Cloudflare right after a deploy. Wait one minute. |
| `agent_error` in the trace | The agent run crashed; the message is in the trace. Check the Claude login first (step 3). |
| `gate_refused` | The checks found problems; the trace lists them. The last deployed version is kept. |
| Many workflows in the workspace | The limit is 200, and a run with a new slot makes one. The last command of step 4 prints the count. |

## The final case

The final case is a folder of the bank's documents plus our mock. The mock must already serve the case's tool API (`notes/hidden-runbook.md`, "The final case", step 2). The repository has the folder as `briefs/final-case/`; use whatever folder holds the case, unzipped exactly as received.

```
mkdir traces                                      # only because of the pipe; PowerShell: mkdir traces -Force
PY builder/build.py briefs/final-case https://the-builder-mock.the-builder-freya.workers.dev 2>&1 | tee traces/final.log
```

PowerShell: replace `| tee traces/final.log` with `| Tee-Object traces\final.log`.

**Checks after the run** (`submission/final-case.workflow.json` below is the exported file; the slot is the folder's name)

| Check | bash (macOS, Linux, Git Bash) | PowerShell |
|---|---|---|
| The log ends with `created workflow_id=...`, then `end seconds=...` under 1800; the exit code is 0; the export file exists | `ls submission/` | `ls submission` |
| No URL in the export points anywhere but our mock (must print nothing) | `grep -o 'https\?://[^" ,\\]*' submission/final-case.workflow.json \| grep -v '^https://the-builder-mock.the-builder-freya.workers.dev' \| sort -u` | `Select-String -Path submission\final-case.workflow.json -Pattern 'https?://[^" ,\\]*' -AllMatches \| % { $_.Matches.Value } \| ? { -not $_.StartsWith('https://the-builder-mock.the-builder-freya.workers.dev') } \| Sort-Object -Unique` |
| No key in any trace (every count must be 0) | `grep -c fak_ traces/*; grep -c sk-ant- traces/*; grep -c "$FREYA_BANK_TEAM_KEY" traces/*` | `Select-String -Path traces\* -Pattern 'fak_','sk-ant-',$env:FREYA_BANK_TEAM_KEY -SimpleMatch \| Measure-Object` (must be 0) |
| `builder/` is unchanged (must print nothing) | `git status --porcelain builder/ \| grep -v '^?? builder/runs/'` | `git status --porcelain builder/ \| Select-String -NotMatch '^\?\? builder/runs/'` |
| No leak of the case into `builder/` | `PY eval/contamination.py` (exit code 0) | the same |

If the run failed with a Freya error (5xx, timeout), it is not our crash: wait and run again with no change. A rerun with the same slot updates the same workflow.

Preflight before the build and the submission steps after it: `notes/hidden-runbook.md`. How the parts fit together and which operating systems are tested: `docs/architecture.md`.
