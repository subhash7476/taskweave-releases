# TaskWeave

TaskWeave runs long tasks in the background using the Claude Code you already have signed in on your computer. You describe the task, walk away, and collect the finished report later. By default it can only *read* the folder you choose: it cannot change files or run programs.

**You need:** Claude Code installed and signed in, and a folder you want it to look at. Python is optional: without Python 3.10 or newer, the installer sets up a private copy for TaskWeave.

**New here?** The [user guides](guides/README.md) take you from no install to a real task. Start with [Start here](guides/00-start-here.md); it explains the Claude Desktop and ChatGPT paths. The rest of this README is the reference. User-visible changes are in [CHANGELOG.md](../CHANGELOG.md).

**Works today:** Claude Code, with a choice of model and effort per job, follow-up turns grouped into threads, text files sent back as artifacts (CSV, Markdown, JSON, diagrams), read-only attachment input (text, Markdown, CSV, PDF and common images) sent by the client or uploaded in the control center while the job waits (for ChatGPT, which cannot pass a file on), numbered files in each job folder (`prompt-artifacts/01.pdf`, `reply-artifacts/01.csv`), and a local control center for jobs, settings and health checks. **Included but switched off:** OpenAI's Codex CLI ([docs/providers/codex.md](providers/codex.md)): the adapter is complete, and it offers permission modes once the real-CLI permission check passes for them. **Not available yet:** Gemini, attachments for jobs that can write, and binary files such as images back from Claude. **Planned:** OAuth sign-in on the HTTP server, so ChatGPT on any system and Claude's custom connectors can connect by URL, and a team mode for one TaskWeave shared by several people.

> **Renamed.** This project was previously called `claude-research-bridge`. The old command names, Python module and `CLAUDE_BRIDGE_*` settings still work for now but are deprecated. See [Upgrading](#upgrading-from-claude-research-bridge).

Technically, TaskWeave is an MCP server that lets any MCP client (ChatGPT through a tunnel, Claude Desktop, other agents) run **long tasks** (research, reviews, reports, code changes) through the **Claude Code CLI already installed and signed in on your machine**.

Long tasks are handled as background jobs, so every tool call returns in seconds and never hits a tunnel or client response deadline:

| Tool | Older name | Purpose |
|---|---|---|
| `start_task(prompt)` | `start_research_job` | Queue a job and return a `job_id` and `thread_id` immediately. Optional: `permission_mode`, `model`, `effort`, `thread_id` to add a follow-up turn to the same thread, `carry_context` to prepend a capped tail of the previous turn's report, `attachments` for read_only jobs, and `wait_for_files` to have a read_only job wait for files you upload in the control center (see [Files you upload in the control center](#files-you-upload-in-the-control-center)). |
| `check_task(job_id)` | `check_research_job` | Status: `waiting_for_files`, `queued`, `running`, `completed`, `failed`, `timed_out`, `cancelled`. |
| `get_task_result(job_id)` | `get_research_result` | Final report, stderr, return code and metadata. Works after failure or timeout. |
| `cancel_task(job_id)` | `cancel_research_job` | Cancel a waiting, queued or running job (the whole process tree is killed). Finished jobs are left unchanged. |
| `list_tasks()` | `list_research_jobs` | Known jobs, newest first, with status, thread and timestamps. Pass `thread_id` to list one thread. |
| `list_task_artifacts(job_id)` | `list_job_artifacts` | Artifact metadata for the report and the files Claude returned. No file content. |
| `get_task_artifact(job_id, artifact_id)` | `get_job_artifact` | Small text/base64 artifact content; larger files return `too_large`. |
| `list_providers()` | | Configured providers, the capabilities implemented in this release, and the models, effort levels and working folders a job may choose (`job_choices`). |

**Tool names.** By default the server offers both names of each tool (`TASKWEAVE_TOOL_NAMES=both`, fifteen tools): the task names above with their full descriptions, and the older research names with a one-line description that points to the new name. Both names run the same code and return the same result, so existing setups and saved prompts keep working. Set `TASKWEAVE_TOOL_NAMES=task` to offer only the eight task names, or `research` for only the eight older names (exactly the tools of 0.1). A client may need to reconnect to see a change. Replies point to the names the server offers (for example the `next` hint after starting a job).

Every tool error body keeps the human-readable `error` string and adds a stable
`error_code` that clients can branch on without matching English text:

| `error_code` | Meaning |
|---|---|
| `invalid_request` | The request is well-formed JSON but not acceptable. |
| `invalid_job_id`, `job_not_found` | The job ID is malformed, or no such job is retained. |
| `invalid_thread_id`, `thread_not_found`, `thread_expired`, `carry_context_unavailable` | The thread ID is malformed, was never seen, or its jobs were removed by retention; or carry_context needs a completed previous turn in that thread. |
| `invalid_prompt`, `prompt_too_long` | The prompt is empty or over `TASKWEAVE_MAX_PROMPT_CHARS`. |
| `invalid_provider_options` | `provider_options` is not an object. |
| `queue_full`, `too_many_jobs` | The waiting queue or the retained-job cap is full. |
| `budget_exceeded` | The daily or monthly budget (`TASKWEAVE_DAILY_BUDGET_USD`, `TASKWEAVE_MONTHLY_BUDGET_USD`) is used up; the message says when it resets. |
| `permission_mode_rejected` | The requested mode is unknown, above the server's ceiling, or unsupported by the provider. |
| `model_not_allowed`, `invalid_model`, `invalid_effort`, `cost_cap_not_enforceable` | A per-job choice is outside the profile, malformed, or cannot be enforced by the provider. |
| `unknown_provider`, `unsupported_provider_option` | The provider is not configured here, or does not accept the option. |
| `unknown_folder`, `thread_folder_mismatch` | The folder is not one of this server's working folders (a path is never accepted), or a follow-up turn named a different folder than its thread's; start a new thread for another folder. See [Working folders](#working-folders). |
| `folder_ambiguous`, `folder_conflict` | The prompt tags two different folders (`#alpha` and `#beta`) or names paths in two unrelated working folders, or a `#name` tag disagrees with the `folder` argument. Keep one folder per job. See [Working folders](#working-folders). |
| `unsupported_attachment` | An attachment is malformed, too large, of an unsupported type, or the job's permission mode cannot receive attachments (also for `wait_for_files` outside `read_only`). A job whose uploaded files fail the server's check ends with this reason. |
| `invalid_artifact_id`, `artifact_not_found`, `invalid_artifact_encoding`, `artifact_error` | An artifact lookup failed validation, was not found, or could not be read safely. |
| `filesystem_error` | A local file operation failed; details stay in the local log (never host paths). |
| `provider_error` | A provider adapter raised an error without a public code. |

Finished jobs also carry a provider-neutral `error_code` next to the historical
`reason` (for example `provider_reported_error` for either provider's error
envelope; other failures keep their existing reason). Successful jobs report
`error_code: null`.

```
MCP client ──► (optional tunnel / reverse proxy, TLS) ──► taskweave ──► claude -p ... (fixed workspace)
```

## ⚠️ Security, read this first

- **Three per-job permission modes, `read_only` by default** (see [Permission modes](#permission-modes)). A job that does not ask for a mode gets `read_only`: Claude can read, list and search files and nothing else. No web access, no edits, no commands, no MCP connectors.
- **The server operator sets a ceiling.** Callers can only pick a mode up to `TASKWEAVE_MAX_PERMISSION_MODE` (default `read_only`). Writing and shell access are opt-in on the server, so a remote caller cannot grant themselves more.
- **`full_repository` runs repository-defined code** (tests, build scripts, git hooks) with your OS user's rights. It is not contained. Read the limits in [docs/permission-modes.md](permission-modes.md) before enabling it.
- **CLI tool restrictions are not an OS sandbox.** The bridge limits Claude through CLI flags. That reduces risk; it does not isolate Claude from your machine. For untrusted callers, run the bridge in a container ([docs/docker.md](docker.md) has a ready image) or as a dedicated low-privilege OS user, with a workspace that holds nothing sensitive.
- **Fixed workspace.** Claude always runs with its working directory set to `TASKWEAVE_WORKSPACE`. A prompt cannot choose it. Claude can still reach files outside it if the OS allows (see the point above).
- **Data folders are read-only.** Folders in `TASKWEAVE_DATA_DIRS` are added only to `read_only` jobs, which have no tool that can write. `read_write` and `full_repository` jobs never get them. A data folder must not overlap the jobs folder. See [Read-only data folders](#read-only-data-folders).
- **Job data is outside the workspace.** `TASKWEAVE_JOBS_DIR` must not be the workspace or nested inside it; startup rejects that configuration. Prompts, reports, artifacts and attachment bytes therefore do not appear in Claude's working tree, and a workspace scan cannot turn them into provider-visible files.
- **The control center can change settings.** `taskweave-control` edits the settings file and runs the Claude CLI for health checks, so it listens on `127.0.0.1` only and needs a per-run token for every change and every check. Never expose it (or `taskweave-viewer`) through a tunnel.
- **HTTP is always authenticated.** `--transport http` refuses to start without `TASKWEAVE_AUTH_TOKEN` (16+ characters). Use TLS (a tunnel or reverse proxy) for anything beyond localhost.
- **Prompts cost money.** Every job is a real Claude session billed to the signed-in account. Even a trivial prompt can cost $0.20 or more on a large model, because the CLI sends and caches a large system prompt. Set a default model, a cost cap per job (`max_cost_usd`) and a daily or monthly budget for all jobs (`daily_budget_usd`, `monthly_budget_usd`) in the profile; the installer's starter profile sets $2 per job and $10 a day.
- Secrets are redacted from logs and returned stderr. Prompts are never logged. Claude receives a scrubbed environment: `TASKWEAVE_*`, `CLAUDE_BRIDGE_*`, `MODEL_CLI_*`, `GIT_*`, cloud/provider credentials and anything named `*_KEY`, `*_TOKEN`, `*_SECRET` or `*_PASSWORD` are removed (`ANTHROPIC_*` and `CLAUDE_CODE_*` are kept; add others with `TASKWEAVE_PASS_ENV`). Every other provider gets the same scrubbing, keeps only its own CLI's variables, and never sees another provider's (see [docs/providers.md](providers.md)).
- **The Claude CLI must support the restriction flags** (`--restricted`, `--strict-mcp-config`, `--permission-prompts`, ...). If it does not, jobs fail with `cli_missing_required_flags` instead of running unrestricted. Verified with CLI 2.1.286.

## Requirements

- Python 3.10+, or nothing: when the installer finds no Python 3.10 or newer, it downloads Astral's [uv](https://docs.astral.sh/uv/) (pinned by version and SHA-256, from PyPI) into the install folder, and uv installs a private Python 3.12 there. PATH, the Windows registry and any other Python are left alone, and removing the install folder removes it.
- [Claude Code CLI](https://docs.claude.com/en/docs/claude-code) installed and signed in. Check with `claude -p "say hi"`.
- **Windows:** the native installer (`irm https://claude.ai/install.ps1 | iex` in PowerShell; `claude.exe` lands in `%USERPROFILE%\.local\bin`) and npm (`npm install -g @anthropic-ai/claude-code`) both work. TaskWeave never runs a `.cmd` file, because `cmd.exe` truncates multi-line prompts and allows command injection; for npm's `claude.cmd` it reads which `claude.exe` the shim starts (inside npm's `node_modules`, checked to be a real Windows program) and runs that directly. If npm skipped Claude Code's install script (`--ignore-scripts`), that program is missing and the checks say how to fix it.
- **Claude Code 2.1.317 and newer** print `--output-format json` as one JSON array of every session event when `--restricted` is passed, which TaskWeave always passes. TaskWeave reads only the final `result` element and never returns the other events (they hold host paths and tool output). TaskWeave 0.2.0rc1 and older do not read this shape: every job's report is the raw array, cut short, and the setup test fails with *The reply did not contain the expected text*.

## Install

**TaskWeave is proprietary and free to use under its [End User License Agreement](../EULA.md). Its releases are a public download from the [TaskWeave releases page](https://github.com/subhash7476/taskweave-releases/releases/latest): no GitHub account, invite or `gh` is needed.** The source repository is private. It is not on PyPI or any other public package index, so `pip install taskweave-mcp` and `uvx taskweave-mcp` do not work; the installer puts the wheel (`taskweave_mcp-<version>-py3-none-any.whl`) into a virtual environment of its own. The installer asks you once to accept the EULA (`-AcceptEula` / `--accept-eula` answers for unattended installs) and installs nothing without a yes.

**The easy way: one download.** Open the [releases page](https://github.com/subhash7476/taskweave-releases/releases/latest) and download the one zip for your computer: `TaskWeave-<version>-windows-x64.zip` (most Windows PCs; there is no Windows on ARM zip yet), `-macos-arm64` (Apple silicon), `-macos-x64` (Intel Macs), `-linux-x64` or `-linux-arm64`. Extract it (on Windows: right-click, **Extract All**). In the extracted folder, double-click `install.cmd` on Windows, or run `sh install.sh` in Terminal on macOS and Linux, and type `yes` for the licence agreement. The zip holds the installer, the wheel, `constraints.txt`, `EULA.md`, `SHA256SUMS.txt` and a `wheelhouse` folder with every dependency for that system, so the install downloads nothing when a Python 3.10+ is present. You need nothing else installed: without Python 3.10+, the installer sets up a private one (see [Requirements](#requirements)), which needs the internet once.

**The separate files (Windows).** The same files are also attached to the release one by one:

1. Open the [releases page](https://github.com/subhash7476/taskweave-releases/releases/latest). A release candidate is marked *Pre-release*; use one only when a guide tells you to.
2. Download these files into **one folder**:
   - `taskweave_mcp-<version>-py3-none-any.whl` (exactly one wheel);
   - `install.ps1` and `taskweave_install.py`;
   - `install.cmd`;
   - `constraints.txt` (the dependency versions this release was tested with);
   - `EULA.md` (the licence agreement the installer shows);
   - `SHA256SUMS.txt`.
3. Double-click `install.cmd`. If Windows says it protected your PC, choose **More info**, then **Run anyway**. Type `yes` when it asks about the licence agreement.

The installer finds the wheel in its own folder and checks it against `SHA256SUMS.txt`; no command is typed and no file name is needed. That check protects against a wrong, partial or corrupted download. It does not protect against a malicious release: whoever can replace the wheel can replace the checksum file next to it. If the browser renamed a repeated download to `... (1).whl`, delete it and download the wheel again: the installer ignores it and says so.

**Let the installer download.** With only `install.cmd`, `install.ps1` and `taskweave_install.py` in the folder (or `install.sh` and `taskweave_install.py` on macOS and Linux), the installer downloads the latest release from the public releases page over HTTPS, with `constraints.txt` and `EULA.md`, and checks each file against the SHA-256 GitHub reports for it; a file that does not match is refused. It honours `HTTPS_PROXY` and sends no credentials. Pass `-Tag v0.2.0rc4` / `--tag v0.2.0rc4` for a release candidate.

**The `gh` way.** If you already use the [GitHub CLI](https://cli.github.com), you can download the same files with it. On Windows:

```powershell
gh release download --repo subhash7476/taskweave-releases --pattern "install.ps1" --pattern "taskweave_install.py" --pattern "install.cmd" --pattern "*.whl" --pattern "constraints.txt" --pattern "EULA.md" --pattern "SHA256SUMS.txt"
.\install.ps1
```

On macOS and Linux download `install.sh` instead of `install.ps1` and run `sh install.sh`. Keep `SHA256SUMS.txt` in the pattern list: the installer checks the wheel against it, and without it the wheel is installed unchecked. To install a release candidate, put its tag right after `download` (for example `gh release download v0.2.0rc3 --repo ...`; `gh` has no `--tag` flag), then run the installer as usual. Testers of a candidate in the private source repository pass `-Gh -Repo subhash7476/TaskWeave` (`--gh --repo subhash7476/TaskWeave`) so the installer downloads with their `gh` sign-in. The installer:

1. finds Python 3.10 or newer, or, when there is none, downloads uv into `<install folder>/uv`, checks it against its pinned SHA-256 (a mismatch stops everything) and lets it install Python 3.12 into `<install folder>/python`;
2. asks once for the licence agreement (the version accepted is kept in `install.json` in the install folder, so a re-run or an update with the same agreement does not ask again);
3. uses the one TaskWeave wheel next to it (checked against `SHA256SUMS.txt` when that file is present), or downloads the latest release over HTTPS and checks its SHA-256 against the one GitHub reports (if GitHub reports none it says so and carries on);
4. creates a virtual environment (Windows: `%LOCALAPPDATA%\TaskWeave\venv`; macOS and Linux: `~/taskweave/venv`) and installs the wheel, with the dependency versions the release was tested with when `constraints.txt` is next to it (if pip cannot install those, it tries once more with the newest versions and says so). When a `wheelhouse` folder of dependency wheels is next to it (the install bundles carry one), it first installs from that folder with no network at all, and uses PyPI only if those wheels do not fit the machine's Python;
5. writes a starter `taskweave` profile and a workspace folder (`~/TaskWeaveWorkspace`), **never overwriting an existing settings file**;
6. runs `taskweave doctor` and tells you what to fix, for example a missing or signed-out Claude Code, or an npm install whose `claude.exe` is missing;
7. puts the `taskweave`, `taskweave-control` and `taskweave-viewer` commands on PATH, so they work in a new terminal: on Windows it copies them into `<install folder>\bin` and adds that folder to your user PATH; on macOS and Linux it links them into `~/.local/bin` and, when that folder is not on PATH, adds one marked line to your shell's start-up file (`~/.zshrc`, `~/.bashrc` or `~/.profile`). Another program's command of the same name is never replaced. `-NoPath` / `--no-path` skips this;
8. creates two shortcuts, **TaskWeave Control Center** (runs `taskweave-control --background`: opens the running control center, or starts one without a window first) and **Stop TaskWeave Control Center** (`--stop`), in the Start menu on Windows, `~/Applications` on a Mac and the app menu on Linux, so the control center is one click away after a restart (`-NoShortcuts` / `--no-shortcuts` skips this);
9. starts the control center and opens its **Setup** page in your browser (see [Control center](#control-center-and-settings-profiles); it keeps running in the background; `-NoOpen` / `--no-open` skips this).

It does not start the MCP server or run a job. The settings file is the usual per-user one (or `TASKWEAVE_CONFIG`, if set); an existing file is never edited, so if it has no profile with the chosen name the installer says so, skips `doctor`, and tells you to pass `-Profile` with one of your profiles or to create one in the control center. Running it again is safe: the same version changes nothing, a newer release upgrades the package, and your settings are kept. Options: `-Wheel <file>` / `--wheel <file>` installs a wheel from another folder (no download happens), `-Tag v0.1.0` / `--tag v0.1.0` picks a release, `-AcceptEula` / `--accept-eula` accepts the licence agreement without asking, and `-Root`, `-Workspace`, `-Profile` (`--root`, `--workspace`, `--profile`) change the folders and profile name. `install.cmd` accepts the same options and passes them to `install.ps1`. The installer needs `taskweave_install.py` next to it. From a checkout, run `.\scripts\install.ps1` or `sh scripts/install.sh` directly.

**macOS and Linux.** Download `install.sh`, `taskweave_install.py`, the wheel, `constraints.txt`, `EULA.md` and `SHA256SUMS.txt` into one folder, open Terminal in that folder, and run one line:

```bash
sh install.sh
```

A browser download loses the executable bit, which is why the line starts with `sh`; a double-click installer is Windows-only.

Manual install, if you prefer to do each step yourself. Get the wheel from the [releases page](https://github.com/subhash7476/taskweave-releases/releases/latest), or with the GitHub CLI:

```bash
gh release download vX.Y.Z --repo subhash7476/taskweave-releases --pattern "*.whl"
```

Installing by hand skips the licence prompt; the [EULA](../EULA.md) still applies.

Windows, step by step (PowerShell; the folder and version are examples, use the release you downloaded):

```powershell
py -3.12 -m venv C:\TaskWeave\venv
C:\TaskWeave\venv\Scripts\python.exe -m pip install C:\path\to\taskweave_mcp-X.Y.Z-py3-none-any.whl
C:\TaskWeave\venv\Scripts\taskweave.exe --version
C:\TaskWeave\venv\Scripts\taskweave.exe doctor
```

`taskweave doctor [--profile NAME] [--config FILE] [--json]` runs the same checks as the
control center's Health section: settings, each enabled CLI, the workspace, data folders and
the jobs folder. It prints `ok`, `warn` or `fail` for each, then `Overall`. Exit codes: `0`
ok or warnings only, `1` at least one failure, `2` the settings file or profile could not be
read. `--json` prints the full report for scripts. `--support` prints a support report to paste into an issue: versions, every check and hint, with secret values and key-shaped strings, paths in the home folder, the user name, e-mail addresses and tunnel IDs removed (the control center's Health section has the same as **Copy diagnostics**).

macOS / Linux:

```bash
python3 -m venv ~/taskweave/venv
~/taskweave/venv/bin/python -m pip install /path/to/taskweave_mcp-X.Y.Z-py3-none-any.whl
~/taskweave/venv/bin/taskweave --version
```

`--version` prints `taskweave X.Y.Z`. This installs the commands `taskweave` (also available as `taskweave-mcp`), `taskweave-control` (the control center) and `taskweave-viewer` (the read-only session viewer).

To build the wheel from a checkout instead: `python -m pip install build`, then `python -m build` (the wheel lands in `dist/`). To run straight from a checkout without building: `uvx --from . taskweave --help`.

**Install report.** Every install writes `install-report.json` in the install folder: the operating system, the Python and TaskWeave versions, which install steps worked and, when `doctor` found a problem, each check's name and status. It holds no paths, user names, file contents or prompts. When an install fails, the installer names the file; attach it if you ask for help. It is sent to the maintainer only when you say yes to the installer's question, and that question is asked only when the release has a report address (this release has none, so nothing is ever sent). `-SendReport` / `--send-report` and `-NoSendReport` / `--no-send-report` answer for unattended installs; your answer is kept in `install.json`, so an update does not ask again. `taskweave uninstall` removes the file.

**Update.** `taskweave update` installs the newest public release with that release's own installer (checked against GitHub's checksum); `--check` only says whether one exists, and `--tag v0.2.0rc4` picks a release. Settings and jobs are kept. The control center also shows a notice when a newer release exists (see `TASKWEAVE_UPDATE_CHECK`).

**Uninstall.** `taskweave uninstall` (or, on Windows, **Settings, Apps, Installed apps, TaskWeave, Uninstall**) stops the control center and removes the shortcuts, the PATH entry, the program and a private Python if the installer set one up. It asks first (`--yes` skips the question). Your settings file and jobs stay unless you add `--remove-data`, and the workspace is never removed. It runs the installer's own copy, kept in `<install folder>/installer`, with the Python that built the program, because Windows cannot delete a program while it runs.

## Upgrading from claude-research-bridge

TaskWeave was previously known as `claude-research-bridge`, and some local setups still use that name. Nothing breaks immediately: the old names keep working for at least one release, and the old commands print a one-line notice on stderr (never on stdout, so MCP clients are unaffected) before running as usual.

| Before | Now |
|---|---|
| the earlier install name, `uvx claude-research-bridge` or `pip install claude-research-bridge` | the `taskweave-mcp` wheel (see [Install](#install)) |
| command `claude-research-bridge` | `taskweave` (or `taskweave-mcp`) |
| command `claude-research-bridge-viewer` | `taskweave-viewer` |
| `python -m claude_research_bridge`, `import claude_research_bridge` | `python -m taskweave_mcp`, `import taskweave_mcp` (the old names still work; importing the old module name raises a `DeprecationWarning`) |
| settings `CLAUDE_BRIDGE_*`, `CLAUDE_BIN`, `MODEL_CLI_*` | `TASKWEAVE_*` (see [Configuration](#configuration)) |
| job folder `claude-research-bridge` | new installs use `taskweave`; an existing folder is kept (see "Where jobs are stored") |
| MCP client config entry | use any name you like for the entry; point its command at the installed `taskweave` command (the server reports its name as "TaskWeave" and its own version) |

Update your MCP client configuration and scripts when convenient. The deprecated names will be removed in a later release, which will be announced in the release notes first.

## Configuration

Settings come from environment variables, or from a named profile in a settings file (see [Control center and settings profiles](#control-center-and-settings-profiles)); environment variables override profile values. A `.env.example` is provided; TaskWeave does not load `.env` itself, so export the variables or set them in your MCP client config.

| Variable | Default | Meaning |
|---|---|---|
| `TASKWEAVE_WORKSPACE` | **required** | Directory Claude runs in. Must exist. |
| `TASKWEAVE_CLAUDE_BIN` | `claude` from `PATH` | Path to the Claude CLI. |
| `TASKWEAVE_PROVIDERS` | `claude` | Providers this server runs, comma-separated: `claude`, and `codex` once verified ([docs/providers/codex.md](providers/codex.md); until then start-up refuses it and says how to verify it). See [docs/providers.md](providers.md) for how providers are configured. |
| `TASKWEAVE_PROVIDER` | the first of `TASKWEAVE_PROVIDERS` | Provider for jobs whose client does not name one. Must be one of `TASKWEAVE_PROVIDERS`. |
| `TASKWEAVE_DATA_DIRS` | none | Extra folders that `read_only` jobs may read, as absolute paths separated like `PATH` (`;` on Windows, `:` elsewhere). Never added to `read_write` or `full_repository` jobs. See [Read-only data folders](#read-only-data-folders). |
| `TASKWEAVE_DEFAULT_PERMISSION_MODE` | `read_only` | The mode a job gets when the client does not ask for one (ChatGPT and Claude Desktop usually do not). Never above the permission cap or the default folder's own mode. See [Working folders](#working-folders). |
| `TASKWEAVE_DEFAULT_FOLDER` | `workspace` | The working folder a job runs in when nothing else picks one: `workspace` or the name of a folder in `TASKWEAVE_FOLDERS`. |
| `TASKWEAVE_FOLDER_ROUTING` | `prompt` | `prompt`: a `#name` tag or a working folder's full path in the prompt also picks the folder. `explicit`: only the `folder` argument, the thread and the default do. Either way only working folders can be used. See [Working folders](#working-folders). |
| `TASKWEAVE_FOLDER_DETAILS` | none | Optional aliases and a description per folder, as `name\|alias,alias\|description` entries separated by `;`. A `#tag` or the `folder` argument may use an alias; clients see the description. See [Working folders](#working-folders). |
| `TASKWEAVE_SHOW_FOLDER_PATHS` | `false` | Also list each working folder's path in `folders_available`. Off: names only. |
| `TASKWEAVE_FOLDER_SETTINGS` | `environment` | For a server with a settings profile: `profile` makes the profile's folder settings win over the same settings in the environment, which are then used only while the settings file cannot be read. The Connect page's launcher sets it. See [Working folders](#working-folders). |
| `TASKWEAVE_FOLDERS` | none | Named working folders a job can be run in, as `name\|mode\|path` entries separated like `PATH`. A job picks one by name; its mode is limited by the folder's mode and the permission cap. See [Working folders](#working-folders). |
| `TASKWEAVE_FOLDER_GROUPS` | none | Parent folders whose immediate subfolders each become a working folder named after the subfolder, as `mode\|filter\|path` entries separated like `PATH` (`filter`: `all`, or `git` for subfolders with `.git`). See [Folder groups](#folder-groups). |
| `TASKWEAVE_JOBS_DIR` | per-user data dir (see "Where jobs are stored") | Where job folders (`prompt.txt`, `stdout.txt`, `report.txt`, `metadata.json`, artifacts, ...) are stored. It **must not** be inside the workspace; startup rejects that configuration. |
| `TASKWEAVE_VIEWER_URL` | unset | Loopback address of the session viewer or control center, for example `http://127.0.0.1:8765`. When set, `start_task` adds a `watch_url` deep link to the returned turn. Only `http(s)` on `localhost`, `127.0.0.1` or `::1` is accepted, with no path, query, fragment or credentials. |
| `TASKWEAVE_MAX_CONCURRENT` | `1` | Jobs running at once. Others wait as `queued`. Two jobs that can write never run in the same working folder at once, whatever this is; see [Working folders](#working-folders). |
| `TASKWEAVE_MAX_QUEUE` | `20` | Jobs allowed to wait; further submissions are rejected. |
| `TASKWEAVE_JOB_TIMEOUT_SECONDS` | `1800` | Hard limit per job, enforced by a background watchdog. |
| `TASKWEAVE_MAX_PERMISSION_MODE` | `read_only` | Highest mode any caller may request: `read_only`, `read_write` or `full_repository`. **See Permission modes.** |
| `TASKWEAVE_ALLOW_WEB` | `false` | `true` adds `WebSearch` and `WebFetch` to every mode. Web fetches can carry data out in URLs, so it is off by default. |
| `TASKWEAVE_DESKTOP_NOTIFY` | `false` | `true` shows a desktop notification when a job finishes. Only the job ID and terminal status are sent; a missing tool or failed notification is logged and never affects the job. |
| `TASKWEAVE_OPEN_UPLOAD_PAGE` | `true` | When a job waits for files you upload (`wait_for_files`), open the control center at that job, starting it if needed, unless a control center page is open. Never in a container or a Linux session without a display. |
| `TASKWEAVE_UPLOAD_WAIT_SECONDS` | `1800` | How long such a job waits for the upload before it ends as `failed` with reason `upload_timeout` (at least 60). |
| `TASKWEAVE_STREAM_OUTPUT` | `false` | `true` makes Claude jobs write their events to `stdout.txt` as they happen (`--output-format stream-json`) instead of one result document at the end, so the control center can show a live tail of a running job. The report, `get_task_result` and the size limit (`TASKWEAVE_MAX_STDOUT_BYTES`, which now bounds the whole stream) are unchanged. The stream is several times larger than the result, and its first event holds host details (workspace, home folder, user name), so the viewer and the live tail show only assistant text and tool names, never the raw events. Off by default. |
| `TASKWEAVE_TOOL_NAMES` | `both` | Which MCP tool names to offer: `both` (the task names such as `start_task`, plus the older `start_research_job` names so existing setups keep working), `task` (only the task names) or `research` (only the older names). See **Tool names** under the tool table at the top. |
| `TASKWEAVE_PASS_ENV` | empty | Comma-separated environment variable names to pass to Claude despite scrubbing (for example Bedrock/Vertex credentials). |
| `MODEL_CLI_MAX_PERMISSION_MODE`, `CLAUDE_BRIDGE_MAX_PERMISSION_MODE` | unset | **Deprecated.** Older names for the permission ceiling; must agree with every other permission setting supplied. |
| `MODEL_CLI_ALLOW_EDITS`, `CLAUDE_BRIDGE_ALLOW_EDITS` | unset | **Deprecated boolean aliases.** `true` means `read_write`; `false` means `read_only`. |
| `TASKWEAVE_AUTH_TOKEN` | unset | Bearer token. Required for `--transport http`. |
| `TASKWEAVE_MAX_PROMPT_CHARS` | `12000` | Longest accepted prompt. |
| `TASKWEAVE_MAX_RESULT_CHARS` | `30000` | Report size returned through MCP; longer reports are cut and end with `[TRUNCATED: showing <N> of <M> characters. Full report: report.txt in the job directory.]`. Clients must treat that marker as literal text, not report content. |
| `TASKWEAVE_MAX_INLINE_ARTIFACT_BYTES` | `1000000` | Raw artifact bytes eligible for inline retrieval. |
| `TASKWEAVE_MAX_INLINE_ARTIFACT_RESPONSE_BYTES` | `2000000` | Maximum actual returned JSON bytes, including Base64 expansion and metadata. |
| `TASKWEAVE_MAX_JOBS` | `200` | Jobs retained on disk. |
| `TASKWEAVE_RETENTION_DAYS` | `14` | Finished jobs older than this are deleted (`0` keeps everything). |
| `TASKWEAVE_MODELS` | empty (any model) | Comma-separated models a client may choose per job, for example `claude-opus-5-5,claude-sonnet-5-5`. With a list and no default, the first one is used. |
| `TASKWEAVE_DEFAULT_MODEL` | unset (the CLI's default) | Model used when the client does not choose one. Must be in `TASKWEAVE_MODELS` when that is set. |
| `TASKWEAVE_EFFORTS` | empty (any level) | Comma-separated effort levels a client may choose: `low`, `medium`, `high`, `xhigh`, `max`. |
| `TASKWEAVE_DEFAULT_EFFORT` | unset (the CLI's default) | Effort level used when the client does not choose one (passed as `--effort`). |
| `TASKWEAVE_MAX_COST_USD` | unset (no cap) | Spending cap per job in US dollars, passed to Claude as `--max-budget-usd`. The installer's starter profile sets $2. |
| `TASKWEAVE_DAILY_BUDGET_USD` | unset (no budget) | Budget for all jobs together per local day, in US dollars. Once jobs finished today have cost this much, new jobs are refused with `budget_exceeded` (and a queued job that would start fails with that reason) until midnight. A running job is never stopped; the per-job cap bounds how far one job can go over. Costs are what the CLI reports, kept in `spend.jsonl` in the jobs folder (also after retention removes a job). The starter profile sets $10. |
| `TASKWEAVE_MONTHLY_BUDGET_USD` | unset (no budget) | The same for the calendar month. |
| `TASKWEAVE_FALLBACK_MODEL` | unset | Model to fall back to when the chosen one is overloaded (`--fallback-model`). |
| `TASKWEAVE_<ID>_BIN`, `TASKWEAVE_<ID>_PASS_ENV`, `TASKWEAVE_<ID>_MODELS`, ... | unset | The same settings for another provider listed in `TASKWEAVE_PROVIDERS` (for example `TASKWEAVE_CODEX_BIN`). The unprefixed model settings above are Claude's. A cost cap the provider's CLI cannot enforce is refused unless `TASKWEAVE_<ID>_ALLOW_UNCAPPED=true`. See [docs/providers.md](providers.md#enabling-and-configuring-providers). |
| `TASKWEAVE_UPDATE_CHECK` | `true` | The control center checks the public releases page for a newer TaskWeave at most once a day and shows a notice (`taskweave update` installs it). `false` turns the check off. Only GitHub's public release information is fetched; nothing about you or your jobs is sent. The MCP server never checks. |
| `TASKWEAVE_PROFILE` | unset | Settings profile to start with (same as `--profile`). |
| `TASKWEAVE_CONFIG` | per-user `taskweave.toml` | Settings file to read profiles from (same as `--config`). |

**Older setting names.** Every setting above except the permission ones was previously `CLAUDE_BRIDGE_<NAME>` (for example `CLAUDE_BRIDGE_WORKSPACE`), and `TASKWEAVE_CLAUDE_BIN` was `CLAUDE_BIN`. The old names still work for now. `CLAUDE_BIN` is accepted without a warning; every other old name prints a deprecation warning at startup. If both an old and a new name are set:

- for ordinary limits (concurrency, timeouts, sizes, retention), the `TASKWEAVE_` value is used;
- for settings that control access or what runs (`WORKSPACE`, `JOBS_DIR`, `AUTH_TOKEN`, `PASS_ENV`, `ALLOW_WEB`, `PROVIDER`, `CLAUDE_BIN`), different values are an error and startup stops. Equal values are accepted. Remove the old name to fix it. Values are never printed in these messages. A `TASKWEAVE_` name from this list that is present but empty (for example `TASKWEAVE_ALLOW_WEB=`) is also an error, so that an empty value can never quietly fall back to a leftover old name; remove the variable or give it a value. For ordinary limits, an empty value still counts as unset.

All explicitly supplied permission settings are constraints, not precedence rules: `TASKWEAVE_MAX_PERMISSION_MODE` and the four deprecated names above must resolve to the same mode or startup fails closed. Explicit empty or whitespace-only permission values are errors; remove unused legacy variables rather than leaving them blank.

TaskWeave's own settings (anything starting with `TASKWEAVE_`, `CLAUDE_BRIDGE_` or `MODEL_CLI_`) are never passed to Claude, and `TASKWEAVE_PASS_ENV` cannot re-enable them.

Generate a token with `python -c "import secrets; print(secrets.token_urlsafe(32))"`.

**Where jobs are stored.** If you do not set `TASKWEAVE_JOBS_DIR`, jobs go in the per-user data folder `taskweave` (on Windows `%LOCALAPPDATA%\taskweave\jobs`). Earlier versions used a folder named `claude-research-bridge` instead. To keep your existing jobs visible, the rules are:

| `taskweave\jobs` | `claude-research-bridge\jobs` | Used |
|---|---|---|
| missing | missing | `taskweave` (created when the server starts) |
| exists | missing | `taskweave` |
| missing | exists | the original folder, with a notice |
| exists, holds no jobs | exists, holds jobs | the original folder, with a notice |
| exists (with or without jobs) | exists, holds no jobs | `taskweave` |
| exists, holds jobs | exists, holds jobs | **startup stops**: set `TASKWEAVE_JOBS_DIR` to the one you want |

A folder "holds jobs" if it contains at least one job folder (a 32-character hexadecimal name). TaskWeave never moves, merges or deletes job folders, and an explicit `TASKWEAVE_JOBS_DIR` (or the older `CLAUDE_BRIDGE_JOBS_DIR`) is always used as given. The viewer follows the same rules.

Resource limits are configurable with `TASKWEAVE_MAX_STDOUT_BYTES` (default 10,000,000), `TASKWEAVE_MAX_STDERR_BYTES` (1,000,000), `TASKWEAVE_MAX_ARTIFACTS` (50), `TASKWEAVE_MAX_ARTIFACT_BYTES` (25,000,000), `TASKWEAVE_MAX_TOTAL_ARTIFACT_BYTES` (50,000,000), `TASKWEAVE_MAX_INLINE_ARTIFACT_BYTES` (1,000,000 raw bytes), and `TASKWEAVE_MAX_INLINE_ARTIFACT_RESPONSE_BYTES` (2,000,000 bytes of the actual returned JSON, including Base64 expansion and response metadata). Attachment limits are `TASKWEAVE_MAX_ATTACHMENTS` (10), `TASKWEAVE_MAX_ATTACHMENT_BYTES` (10 MiB), `TASKWEAVE_MAX_TOTAL_ATTACHMENT_BYTES` (25 MiB), and `TASKWEAVE_MAX_ATTACHMENT_FILENAME_CHARS` (120).

These output limits are intentional. `stdout` or `stderr` above its capture limit fails the job with `output_limit_exceeded`; the file is truncated with a visible marker and the job is never reported as a success. If the file cannot be inspected or truncated (for example it is locked), the job records `output_truncation_error` / `output_inspection_failed` and a completed run is downgraded rather than reported successful. An artifact over either inline limit returns the documented `too_large` shape with `content: null`; it is not silently shortened. If `report.txt` cannot be written, the job ends `failed` with reason `report_write_failed` and reports `report_persisted: false` / `report_persistence_error` instead of a false success.

Command line: `taskweave --transport stdio|http [--host 127.0.0.1] [--port 8080] [--profile NAME] [--config FILE]` (stdio is the default; the HTTP endpoint is `/mcp`). `taskweave doctor [--profile NAME] [--config FILE] [--json]` checks a setup without starting the server. `taskweave --transport stdio --setup-check` validates the whole configuration and serves over a temporary jobs folder that is deleted on exit, so a launcher can be proved without touching your real jobs, heartbeat or recovery. `taskweave run-test --workspace DIR --jobs-dir DIR [--claude-bin PATH] [--timeout SECONDS] [--max-cost USD]` runs the first-run test task (below) and prints one JSON line; the exit code is 0 only when it passed.

## Control center and settings profiles

`taskweave-control` opens a local page (`http://127.0.0.1:8765/`) with five sections (`--setup` opens the first one you need on a new machine):

- **Jobs**: the session viewer (see below). A picker in the header switches between the default jobs folder and each profile's jobs folder.
  In the control center a job waiting for files has an **Upload files** button (and every control center page shows a banner for each waiting job; see [Files you upload in the control center](#files-you-upload-in-the-control-center)), and a waiting, queued or running turn has a **Cancel job** button. The control center is a separate process from the MCP server, so the click writes a `cancel.request` file in the job's folder; the running server's watchdog notices it within about a second and cancels the job exactly as `cancel_task` does (the job ends `cancelled` with reason `cancelled_by_user`, and `metadata.json` records `cancel_source`: `control_center` or `mcp`). The server writes a small `server.json` heartbeat in the jobs folder every few seconds; if it is missing or stale the page says TaskWeave is not running and the job will be cancelled when it next starts. The read-only viewer has no cancel button. The control center cannot stop a process on its own, so a job whose server is down keeps running until the server returns.
  With `TASKWEAVE_STREAM_OUTPUT=true` a running turn also has a **Live** fold that shows the assistant's text and the names of the tools it calls, refreshed every two seconds (`GET /api/jobs/{id}/live?after=<byte offset>`, loopback only, at most a few KB per request). Tool inputs, tool results, reasoning and the start-up event are never sent.
- **Setup**: a four-step first-run wizard that opens by itself when no profile exists (and from the installer). It reuses the installer's starter profile if there is one.
  1. **Folders**: the workspace and the jobs folder, each typed, browsed (a server-side list of subfolders: directories only, no links, bounded, loopback and token only) or created for you, plus an optional Claude CLI path.
  2. **Checks**: the same cards as Health, run on the values before anything is saved.
  3. **Test task**: one button that sends a tiny request to your signed-in Claude Code and expects the exact reply `TASKWEAVE_TEST_OK`. It runs as `taskweave run-test` in its own process (the control center never runs jobs itself). The test is **read-only with no tools at all** (`--tools ""`; if the command cannot be built that way it is not run), uses only the folders and CLI path from the wizard (no model, effort, cost cap or data folder from your settings or environment), stops after 120 seconds, is capped at **$0.50** of model usage (`--max-budget-usd`, stated on the page before the click; one tiny request can cost about $0.25 because Claude Code creates its prompt cache, and a lower cap stops it with `error_max_budget_usd` before it replies), runs one at a time, and returns a short redacted result. It works in its own `onboarding-test` folder inside the jobs folder, which the Jobs list, retention and the restart scan ignore; the folder is deleted when the test passes and kept when it fails. A sign-in or rate-limit error says to try again. It needs a signed-in CLI: with an expired sign-in it reports the CLI's own message.
  4. **Connect**: saves the profile and shows the command that starts the server, then the registration snippet for **Claude Desktop** (`mcpServers` JSON with the installed `taskweave` path) or the command for **ChatGPT**. ChatGPT still needs a tunnel and a connector that you create yourself (see [Connect TaskWeave to ChatGPT](chatgpt-tunnel.md)); the wizard does not automate that part.
- **Connect** (Windows): the machine side of [connecting ChatGPT through the OpenAI tunnel client](chatgpt-tunnel.md). From Connect you can write the launcher, prove it, check the tunnel client, create the tunnel profile (tunnel-client 0.0.19 or newer, in a `tunnel-profiles` folder next to the launcher) and write the start script (Step 3f) that asks for your key in its own window. Its last step shows an advisory readiness light (the tunnel client's local `/readyz`, loopback only) and a first-job watcher: press Watch, send a prompt from ChatGPT, and a job created after that moment appears (only its short id, status, mode and time are shown, never the prompt or the report). A job in the folder shows that something submitted one; it counts as ChatGPT through the tunnel only if you sent the prompt from ChatGPT. The first-job check also compares a launcher that holds the settings with the saved profile (workspace, data folders, working folders, permission cap, default mode and folder) and says when it is out of date and must be written again. It cannot create the tunnel, supply your runtime key (it is never typed into the page) or create the connector in ChatGPT, and a green check here proves none of those.
  1. **Launcher**: generates `taskweave-stdio.cmd` from a saved profile (settings written into it, or `--profile`/`--config`; the profile type refuses a settings file under `%LOCALAPPDATA%`), refuses paths a launcher cannot hold safely (quotes, line breaks, and non-ASCII characters when Windows has no plain short (8.3) name for the folder; with one, the short name is written instead), never overwrites without asking (the old file is kept as `.bak`), and shows the tunnel command with the forward slashes it needs. **Prove the launcher** starts it exactly as the tunnel client will, with `--setup-check`, and requires exactly two MCP replies and every TaskWeave tool.
  2. **Tunnel client**: you give the path of the `tunnel-client.exe` you downloaded (TaskWeave never downloads or starts it by itself); the page runs `--version` and applies the version rules: below 0.0.17 the automated steps are refused, a newer version than the verified ones (0.0.17, 0.0.18, 0.0.19; for 0.0.19 only `--version` and `dev proxy`) is allowed with a note. **Offline check** runs `tunnel-client dev proxy` in front of the launcher (no key, no tunnel, no network) and sends an MCP `initialize`.
  Only a short list of commands is ever run, never through a shell, with keys and tokens removed from their environment; the control center makes no network request other than to the loopback address the offline proxy reports. The tunnel profile, starting the tunnel and the connector are still the manual steps below.
- **Settings**: create, edit and delete **profiles**. A profile holds a root folder, the workspace, read-only data folders, the jobs folder, the Claude CLI path, the permission cap, web access, concurrency, timeout, retention, and the model policy (allowed models, default model, allowed effort levels, default effort, cost cap per job, fallback model).
- **Health**: checks whether a profile (or the `TASKWEAVE_*` variables of the window that started the control center) is ready to run jobs. See [Health checks](#health-checks).

```bash
taskweave-control                       # settings file: $TASKWEAVE_CONFIG or the per-user taskweave.toml
taskweave-control --background          # open the running one, or start one without a window first
taskweave-control --stop                # stop the one running for this settings file
taskweave-control --config D:/tw/taskweave.toml --port 8765
```

Profiles are saved in a TOML file (on Windows `%LOCALAPPDATA%\taskweave\taskweave.toml` by default; the previous version is kept as `taskweave.toml.bak`). For a server started by another program, such as a tunnel client, prefer a settings file next to its launcher, passed with `--config` to both `taskweave` and `taskweave-control`; see [Windows: a profile or settings file is not found](#windows-a-profile-or-settings-file-is-not-found) for why. Hand edits are fine:

```toml
[profiles.nifty]
root = 'G:\TaskWeave'
workspace = 'G:\Nifty'
jobs_dir = "jobs-nifty"            # relative to root
data_dirs = ['G:\MarketData']     # read_only jobs may also read these
max_permission_mode = "read_only"
models = ["claude-opus-5-5", "claude-sonnet-5-5"]
default_model = "claude-opus-5-5"
default_effort = "high"
max_cost_usd = 5.0
```

Start the server with a profile, for example in a tunnel launcher: `taskweave --profile nifty --transport stdio`. The Settings page shows this line for each profile.

- **Checks.** Saving runs the same checks as server start-up (the workspace must exist, the jobs folder must not be inside it, the default model must be an allowed one, and so on), and shows each problem next to its field. Nothing is saved until the profile is valid.
- **When changes apply.** Folders and server limits apply when the server next starts, so restart the tunnel client (it starts the server). Model, effort and cost-cap changes apply to the **next job** without a restart: the server re-reads them from its profile before each job. If the file has become unreadable, the values from start-up stay in force.
- **Environment wins.** Any `TASKWEAVE_*` variable set in the server's environment overrides the profile's value.
- **Per-job choices.** `start_task` accepts `model` and `effort`; values outside the profile's lists are rejected before a job is created. `list_providers` reports the allowed values and defaults under `job_choices`, and each job records its model, effort and cost cap in `metadata.json`.
- **Safety.** The control center listens on `127.0.0.1` only and must never be exposed through a tunnel. Unlike the viewer it can write the settings file and run the Claude CLI, so every change and every health check needs a per-run token embedded in the page and must come from the page itself (`Origin` check), and changes must be JSON; requests with a foreign `Host` header are refused. The HTTP auth token is never shown or stored in the settings file; keep it in the environment.

### Health checks

The **Health** section checks one setup at a time and marks each part *OK*, *Needs a look* or *Problem*, with what to do when something is wrong:

| Check | What it looks at |
|---|---|
| Profile settings | Whether the server would start with these settings (the same checks as start-up), and a summary of the permission cap, limits, models and effort. |
| Claude Code CLI (one card per enabled provider) | The CLI is found, answers `--version`, has the flags TaskWeave needs to enforce permission modes, and is signed in (`claude auth status`; only the signed-in state and method are shown, never the account). |
| Workspace | The folder exists and is not empty (Claude can only read files that are in it). |
| Data folders | Each read-only data folder exists and can be listed (the number of items is shown). None set is fine. |
| Jobs folder | The folder exists (or can be created), is outside the workspace, is writable (a temporary file is created and removed) and has at least 1 GB free. |
| Jobs | Running and queued jobs, jobs marked running whose process has gone (the server that ran them stopped; they are marked failed when it next starts), and failures in the last 24 hours. Each running or queued job has an *Open* button that shows it in Jobs. |

Checks run when you open the section, pick a setup or press *Check again*; they take a few seconds because they run the CLI. A profile is checked with its own values: variables set in a launcher can override them, and whether the tunnel client is running is not checked here (use its own health endpoint for that).

## Read-only data folders

Claude works in its workspace, but you can also let it read other folders, for example a market data folder you don't want to copy into the workspace. List them in `TASKWEAVE_DATA_DIRS`, or as *Read-only data folders* in a settings profile (one per line; a relative folder is resolved against the profile's root):

```text
TASKWEAVE_DATA_DIRS="G:\MarketData;G:\Filings"     # Windows: separated by ";"
TASKWEAVE_DATA_DIRS="/srv/market:/srv/filings"      # macOS and Linux: separated by ":"
```

- **Only `read_only` jobs get them.** A `read_only` job has no write or command tools at all, so a data folder cannot be changed. `read_write` and `full_repository` jobs run without the data folders: with Claude Code 2.1.286, rules that deny writes to an added folder did not stop `Write`, so TaskWeave never adds one to a job that can write.
- **How it works.** Each folder is passed to Claude Code with `--add-dir`. `--restricted` still applies: files outside the workspace and the data folders are refused, and so is a junction or symlink in a data folder that points elsewhere. Only Claude Code supports data folders in this release; jobs on other providers get none.
- **What clients see.** `start_task` returns the folder names under `data_folders` (not full paths), and the job's prompt lists the full paths so Claude can find them. Each job records the folders it got in `metadata.json`, and the Jobs view shows them.
- **Rules.** Each folder must exist and be an absolute path (after a profile's root is applied). A data folder must not contain, or be inside, the jobs folder, since a job could otherwise read other jobs' prompts and reports. A folder inside the workspace is ignored with a warning, because jobs can already read it. At most 20 folders. Changes apply when the server next starts.

## Working folders

The workspace is the one folder Claude works in by default. To work in other folders, each with its own highest permission mode, list them as **named working folders**. A job chooses one **by name** (never by path), and Claude then runs with that folder as its working directory, so write access exists only there.

```text
TASKWEAVE_FOLDERS="nifty|read_only|G:\Nifty;site|read_write|G:\Site"     # Windows: entries separated by ";"
TASKWEAVE_FOLDERS="nifty|read_only|/srv/nifty:site|read_write|/srv/site"  # macOS and Linux: separated by ":"
```

Each entry is `name|mode|path`. In a settings profile, use the *Working folders* rows in the control center (Settings): a name, the highest mode, and the path. A client passes `folder="nifty"` to `start_task`, or writes `#nifty` in the prompt (below); with neither, the job runs in the default folder, which is the workspace unless you change it.

**How a job's folder is chosen.** The first of these that applies wins:

1. the `folder` argument of `start_task`;
2. a `#name` tag in the prompt, such as `#nifty` (case does not matter);
3. for a follow-up turn (`thread_id`), its thread's folder;
4. the full path of a working folder written in the prompt, such as `G:\Nifty\prices.csv`;
5. the default folder (below).

Tags and paths (rules 2 and 4) are for clients such as ChatGPT that usually send only a prompt. They only select among the working folders listed here, so they grant nothing a client could not already ask for with `folder`; a path outside every working folder never picks or widens anything. Ambiguity is refused, never guessed: two different tags, or paths in two unrelated working folders, are refused with `folder_ambiguous`, and a tag that disagrees with `folder` with `folder_conflict`. When folders nest, a path picks the innermost folder that holds every path the prompt names. The prompt reaches Claude unchanged, tag included. Each response says which rule applied in `folder_source`.

- **Tags.** `#` followed straight away by a folder name, at the start of the prompt or after a space, bracket or quote, and ending at a space, punctuation or the end. `C#`, `issue#12`, a Markdown heading (`# Title`) and `#nifty/src` are never tags; `#42` or `#todo` count only if a folder has exactly that name, and otherwise stay ordinary text. An unknown tag that looks like a folder name (`#nifyt`) adds a warning when the job falls back to the default folder. The tag is `#`, not `@`, because typing `@` in ChatGPT opens its own menu of apps and GPTs.
- **Paths.** Compared as text and never opened. On Windows, case and `/` versus `\` do not matter, and both the long and the 8.3 short form of a folder's path match. A folder path with spaces works. A folder's bare name in ordinary text ("look at nifty") never picks it: a word is too easily ordinary text.
- **Turning it off.** `TASKWEAVE_FOLDER_ROUTING=explicit` (*Folder from the prompt* in the control center) keeps only rules 1, 3 and 5; a tag that names another folder then only adds a warning. The default is `prompt`.

A job cannot reach a path outside its folder (and, for `read_only` jobs, the data folders), so asking a workspace job to list a path elsewhere fails with Claude's "outside the working directory" message. To catch this at once, `start_task` checks the prompt for absolute paths the job cannot reach and returns a `warnings` list: one sentence per path, naming the working folder that holds it when there is one ("start a new job with folder='nifty' or with #nifty in the prompt"). The check only reads the prompt as text and never opens a path.

**A thread keeps its folder.** A follow-up turn (`thread_id`) without a folder or tag runs in its thread's folder, even if the default has changed since; a path in its prompt does not move it (rule 3 comes first) and is flagged in `warnings` instead. A follow-up whose `folder` or tag names a different folder is refused with `thread_folder_mismatch`; start a new thread for the other folder. If the operator has since removed the thread's folder, the follow-up is refused with `unknown_folder`.

**Making a folder the default.** ChatGPT and Claude Desktop usually send only a prompt, so by default every job runs `read_only` in the workspace and cannot reach a working folder. To change that, set `TASKWEAVE_DEFAULT_FOLDER` (the folder's name) and `TASKWEAVE_DEFAULT_PERMISSION_MODE`; in the control center these are *Default working folder* and *Default permission mode* in Settings. A client that names a folder or a mode still gets what it asked for, within the limits below. The default mode must fit under the permission cap and the default folder's own mode, or TaskWeave refuses to start and says which. When a job's folder comes from somewhere else (the argument, a tag, a path or the thread) and the client names no mode, the default mode is lowered to that folder's own mode if it is lower, never raised.

### Folder groups

When you work in many repositories, register the folder that holds them once instead of listing each one. A **folder group** (`TASKWEAVE_FOLDER_GROUPS`, *Folder groups* in the control center) is a parent folder with one highest mode; each immediate subfolder becomes a working folder named after it, so `E:\Repos\alpha` is `#alpha`. A repository cloned into the parent later is usable for the next job, with no settings change and no restart.

```text
TASKWEAVE_FOLDER_GROUPS="read_write|git|E:\Repos"   # Windows; macOS and Linux: read_write|git|/home/me/repos
```

Each entry is `mode|filter|path`: the highest mode for every subfolder, `all` or `git` (only subfolders that contain `.git`, a folder or a worktree's file), and the parent folder. Skipped, with a note in Health: links and junctions (they could point anywhere, and you approved only the parent), names that are not valid folder names (letters, digits, `-` and `_`, up to 40), the jobs folder, and names already taken. A folder listed by name always wins over a subfolder with the same name, and an earlier group over a later one; Health warns about each clash. Hidden folders (a leading dot) are skipped silently. The parent itself is not a working folder, and at most 10 groups and 200 subfolders per group are used. The group's folders are looked at again before each job and every few seconds while clients poll.

**Changes apply to the next job.** A server started with a settings profile re-reads its folder settings before each job: the working folders, the folder groups, their aliases and descriptions, the default folder and mode, folder routing and `TASKWEAVE_SHOW_FOLDER_PATHS`. Adding a folder in the control center (Settings, *Working folders*) therefore needs no restart and no new launcher. The edit is checked as at startup; a broken one (a folder that does not exist, a name used twice) is refused, the server keeps its last good folder list, and Health says why. The workspace, jobs folder, permission cap, CLI paths and transport still need a restart, so a live edit can never raise a job above the cap the server started with. A folder setting given in the environment pins its value, like every setting, with one exception: the launcher written by the Connect page points at its profile and sets `TASKWEAVE_FOLDER_SETTINGS=profile`, so the profile wins and the launcher's own copies of the folder settings are used only while the settings file cannot be read (a tunnel client started from the Microsoft Store version of PowerShell cannot see `%LOCALAPPDATA%`; see [below](#windows-a-profile-or-settings-file-is-not-found)). Health shows which settings file the running server reads, and warns when the server reads another copy of it than the one the control center edits.

**Aliases and descriptions.** *Folder aliases and descriptions* (`TASKWEAVE_FOLDER_DETAILS`) gives a folder other names, which work as `#tags` and as the `folder` argument (`#nse` for `nifty`), and a one-line description that clients see in `folders_available` and the `start_task` description. Aliases follow the folder name rule and must not clash with any folder name or other alias.

**Telling ChatGPT about your folders.** When a profile has working folders, the control center's Settings page shows a short text with a **Copy ChatGPT instructions** button, for example: "TaskWeave folders: #workspace (read only), #nifty (read only: NSE market data scripts). When you start a TaskWeave task about one of them, put its #name at the start of the prompt." Paste it into ChatGPT's custom instructions or a project's instructions. It names folders, modes and descriptions, never paths.

- **Two limits apply.** A job's mode must not exceed the folder's mode **and** the server's permission cap (`TASKWEAVE_MAX_PERMISSION_MODE`). A `read_only` folder can never be written, whatever the client asks. A request above either limit is rejected before a job starts, and the mode is checked again when a queued job launches.
- **One writing job per folder.** At most one job that can write (`read_write` or `full_repository`) runs in a folder at a time; a folder nested inside another counts as the same folder. `read_only` jobs run alongside it. A writing job for a busy folder waits as `queued`, and the queue starts the first job whose folder is free, so jobs in other folders are not held up. `check_task` on a waiting job adds `queued_reason`, a sentence saying which job holds the folder or that every job slot is in use. To run jobs in different folders side by side, raise `TASKWEAVE_MAX_CONCURRENT` (*Jobs at a time*) above 1; Health suggests it when you have working folders.
- **One working folder per job.** Mixed access inside one job (write here, read-only there) is not offered: Claude Code 2.1.286 did not enforce deny rules for extra folders, so only the working folder can be writable. Read-only reference folders stay available through `TASKWEAVE_DATA_DIRS` to `read_only` jobs.
- **What clients see.** Every `start_task`, `check_task` and `get_task_result` response carries `folder` (where the job runs), `folder_source` (`argument`, `prompt_tag`, `thread`, `prompt_path` or `default`; `null` for jobs started by an older version) and `folders_available` (each folder's name and the highest mode a job may use there: the folder's own mode or the permission cap, whichever is lower; its aliases and description when set; its path only when `TASKWEAVE_SHOW_FOLDER_PATHS` is on), plus `warnings` when the prompt named a path the job cannot reach or a tag that did not pick the folder. `list_tasks` shows each job's folder, and `list_providers` lists the same folders under `job_choices` > `folders`. The `start_task` tool description names this server's folders (with their descriptions), the default and the `#name` rule, as they were when the server started; `folders_available` is always current. ChatGPT keeps the tool list it saved when the connector was set up or last refreshed, so refresh the connector in ChatGPT after updating TaskWeave or changing folders. Each job records `folder` (the workspace included) and `folder_source` in `metadata.json`.
- **Rules.** Names use letters, digits, `-` and `_` (up to 40) and are unique; `workspace` is taken. Each path must exist and be absolute (after a profile's root is applied) and must not contain, or be inside, the jobs folder. At most 20 folders. The Health section lists them. With a settings profile, changes apply to the next job (above); a server configured only by environment variables reads them when it starts.

## Permission modes

Every job runs in exactly one mode, chosen when it starts and **fixed for the life of the job** (it cannot be changed while queued or running, and a new job never inherits a previous job's mode).

```text
start_task(prompt, permission_mode="read_only")   # permission_mode is optional
```

| Mode | Claude can | Claude cannot |
|---|---|---|
| `read_only` (default) | Read files, list (`Glob`) and search (`Grep`) inside the workspace; write a report. | Create, edit, delete, rename or move files; run any command; use git; use the web or MCP connectors. |
| `read_write` | Everything in `read_only`, plus create new files and edit existing ones (`Write`, `Edit`) inside the workspace. | Run any command or git; delete, rename or move files; write to `.git/` or `.claude/`. Note: `Write` can overwrite and `Edit` can empty an existing file, which the CLI cannot tell apart from a normal edit. |
| `full_repository` | Everything in `read_write`, plus a fixed allow-list of shell commands: local git (`status diff log show branch add commit rm mv checkout switch restore stash clean`) and test/build tools (`pytest`, `npm test/run build/run lint`, `cargo test/build/check`, `go test/build/vet`, `dotnet test/build`). Delete, rename and move tracked files with `git rm` / `git mv`. | Push, fetch, pull, clone, change git config, install dependencies, use the network, run `rm`/`mv`/`cp`, shells or any command not on the list. |

A request is rejected before anything starts when the mode is not one of the three exact values, or when it is above `TASKWEAVE_MAX_PERMISSION_MODE`. The chosen mode is returned by `start_task`, `check_task`, `get_task_result` and `list_tasks`, and stored in `metadata.json`.

How it is enforced (details, full tool matrix and limits: [docs/permission-modes.md](permission-modes.md)):

| Control | `read_only` | `read_write` | `full_repository` | Enforced by |
|---|---|---|---|---|
| Tools that exist at all (`--tools`) | Read, Glob, Grep | + Edit, Write | + Bash | **Server builds the flags; the CLI removes everything else.** Verified with the real CLI. |
| Everything not pre-approved is denied (`dontAsk`, no prompts) | yes | yes | yes | CLI |
| File tools confined to the workspace, symlinks/junctions leaving it refused | yes | yes | yes | CLI (`--restricted`), verified |
| User/project settings files, hooks and MCP servers ignored | yes | yes | yes | CLI (`--restricted`, `--strict-mcp-config`), verified |
| Writes to `.git/` and `.claude/` refused | n/a | yes | yes | CLI (`--restricted`), verified for `Write` |
| Shell commands limited to an allow-list; push/config/network/install denied | n/a | n/a | yes | CLI pattern rules. **Best effort, not containment.** |
| Credentials removed from the environment | yes | yes | yes | Server |
| Mode above the operator's ceiling refused (also re-checked when a queued job starts) | yes | yes | yes | Server |
| Prompt text describing the mode | yes | yes | yes | **Advisory only.** Never relied on. |

`full_repository` is **not** a sandbox. Test and build commands run code from the repository, and git runs repository hooks, with your user's OS rights. Use it only on repositories you trust, ideally in a container or a low-privilege account.

## Sign in with an API key, Bedrock or Vertex

TaskWeave runs the Claude Code on your computer, so jobs use whatever that Claude Code signs in with. By default that is your Claude subscription (run `claude` once and sign in). Claude Code can also sign in with an Anthropic API key, through Amazon Bedrock, or through Google Cloud's Vertex AI, which suits teams, companies with a cloud agreement, and servers. Anthropic's guides: [Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock), [Google Vertex AI](https://code.claude.com/docs/en/google-vertex-ai), [authentication](https://code.claude.com/docs/en/iam).

TaskWeave gives Claude Code a cleaned environment (see **Security, read this first** above): `ANTHROPIC_*` and `CLAUDE_CODE_*` variables pass, cloud credentials such as `AWS_*` and `GOOGLE_*` are removed unless you name them in `TASKWEAVE_PASS_ENV`.

| Sign-in | Set these where TaskWeave starts | Reaches Claude Code |
|---|---|---|
| Anthropic API key (Claude Console) | `ANTHROPIC_API_KEY` | by itself |
| Subscription token from `claude setup-token` (for servers without a browser) | `CLAUDE_CODE_OAUTH_TOKEN` | by itself |
| Amazon Bedrock | `CLAUDE_CODE_USE_BEDROCK=1`; `AWS_REGION` unless your AWS profile sets one; credentials as one of: `AWS_PROFILE` (after `aws sso login` or `aws configure`), `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` (+ `AWS_SESSION_TOKEN`), or `AWS_BEARER_TOKEN_BEDROCK` | `CLAUDE_CODE_USE_BEDROCK` by itself; each `AWS_*` name only when listed in `TASKWEAVE_PASS_ENV` |
| Google Vertex AI | `CLAUDE_CODE_USE_VERTEX=1`, `CLOUD_ML_REGION` (for example `global` or `us-east5`), `ANTHROPIC_VERTEX_PROJECT_ID`; credentials from `gcloud auth application-default login`, or `GOOGLE_APPLICATION_CREDENTIALS` | the first three by themselves; `GOOGLE_APPLICATION_CREDENTIALS` only when listed in `TASKWEAVE_PASS_ENV` |

For example, Bedrock with an AWS SSO profile, starting the server from a terminal on macOS or Linux:

```bash
aws sso login --profile my-bedrock
export CLAUDE_CODE_USE_BEDROCK=1 AWS_PROFILE=my-bedrock AWS_REGION=us-east-1
export TASKWEAVE_PASS_ENV=AWS_PROFILE,AWS_REGION
taskweave doctor --profile taskweave
```

On Windows, set them as user environment variables (`setx CLAUDE_CODE_USE_BEDROCK 1`, and so on), then open a new window.

- **Where they go.** They must be in the environment of the process that starts TaskWeave. In a terminal or a service, export them. In Docker, pass them with `-e` ([docs/docker.md](docker.md)). Claude Desktop starts TaskWeave itself: add them to the `env` object of the `taskweave` entry in `claude_desktop_config.json`, which keeps them in plain text, so prefer `AWS_PROFILE` or gcloud's application-default sign-in there over keys. The Claude Desktop extension (`.mcpb`) has no field for them in this release. A settings profile holds only TaskWeave's own settings, never credentials; the environment is applied on top of the profile, so `TASKWEAVE_PASS_ENV` from the environment works with any profile.
- **Which one wins.** When several are present, Claude Code chooses by its own order: a cloud provider (`CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`) first, then `ANTHROPIC_API_KEY`, then `CLAUDE_CODE_OAUTH_TOKEN`, then the subscription you signed in with. TaskWeave jobs are non-interactive, so an `ANTHROPIC_API_KEY` left in the environment is used without asking and bills that API account; remove it if you meant to use your subscription.
- **Check it.** `taskweave doctor` and Health show the method Claude Code reports (`claude.ai` for a subscription, `api_key`, `oauth_token`, `third_party` for Bedrock and Vertex). That does not prove the cloud credentials work: run the setup test task (Setup, step 3, **Test task**) or one small read-only job.
- **Models and cost.** Bedrock and Vertex may name models differently; set `TASKWEAVE_MODELS` and `TASKWEAVE_DEFAULT_MODEL` to names your account offers, or leave them empty for Claude Code's default (which follows `ANTHROPIC_MODEL`). With an API key or a cloud account each job is billed by usage. The cost Claude Code reports, which the per-job cap and the budgets use, is its own estimate; the bill from Anthropic or your cloud provider is what counts.
- **Terms.** The terms of the account you sign in with apply to every job TaskWeave runs: Anthropic's terms for a subscription or an API key, your cloud agreement for Bedrock or Vertex.

## Use with Claude Desktop (stdio)

**The easiest way: the extension.** Download `TaskWeave-<version>.mcpb` from the [releases page](https://github.com/subhash7476/taskweave-releases/releases/latest) and double-click it, or drag it into Claude Desktop's **Settings, Extensions**. Claude Desktop installs it (no Python and no TaskWeave installer needed; it fetches the tested dependency versions with uv) and asks for the workspace folder and, optionally, the Claude Code program, a jobs folder and the spending limits. The control center and the other commands are not part of the extension; install TaskWeave too if you want them. Jobs run read-only, as everywhere.

**The easy way with the full install.** In the control center's Setup wizard, open **Connect** and choose **Claude Desktop**. The page finds `claude_desktop_config.json`, shows a preview (the command and arguments it will add, and the names of the other servers, never their values), and **Add to Claude Desktop** writes the entry after keeping a byte-for-byte backup. It only ever changes Claude Desktop's own config file in the places Claude Desktop keeps it; any other file is refused, even one with the same name. Quit Claude Desktop completely first, including from the system tray, and start it again afterwards. The page shows the backup name; put it back to undo.

The button writes the entry for the saved profile, so the workspace and Claude CLI path in that profile are used:

```json
{
  "mcpServers": {
    "taskweave": {
      "command": "/absolute/path/to/venv/bin/taskweave",
      "args": ["--profile", "taskweave", "--transport", "stdio"]
    }
  }
}
```

Add `"--config", "/absolute/path/to/taskweave.toml"` after the profile name when the profile is in a non-default settings file. `command` is the `taskweave` command from your [install](#install) (on Windows, `C:\\TaskWeave\\venv\\Scripts\\taskweave.exe`). Set the Claude CLI path in the profile if Claude Desktop starts with a smaller `PATH` than your shell. Adding the entry by hand is the fallback: edit the file Claude Desktop opens from **Settings, Developer, Edit Config**. On Linux, Claude Desktop is not an official application, so the button is hidden and this snippet is the only way.

## Use remotely (HTTP behind a tunnel)

```bash
export TASKWEAVE_WORKSPACE=/absolute/path/to/workspace
export TASKWEAVE_AUTH_TOKEN=<long random token>
taskweave --transport http --host 127.0.0.1 --port 8080
```

Safe exposure checklist:

1. Keep the bridge bound to `127.0.0.1` and let the tunnel/reverse proxy connect to it locally.
2. The tunnel or proxy must provide HTTPS. The bearer token is only as private as the connection carrying it.
3. Your MCP client must send `Authorization: Bearer <token>` on every request. If a client cannot send a custom header, do not expose the bridge without another authentication layer in front of it.
4. Leave `TASKWEAVE_MAX_PERMISSION_MODE` at `read_only` unless you fully trust every caller.
5. Rotate the token if it may have leaked. Binding to a non-loopback host is allowed (with a token), but prefer the loopback-plus-tunnel setup.

To run the HTTP server in a container, with Claude Code inside it and an API key, see [docs/docker.md](docker.md).

The bridge does not include or configure any tunnel. Any tunnel client or reverse proxy that forwards HTTPS to `http://127.0.0.1:8080/mcp` works; treat tunnel IDs and keys as your own secrets and never commit them.

## Connect TaskWeave to ChatGPT using the OpenAI tunnel client (Windows)

ChatGPT reaches TaskWeave on your Windows PC through the OpenAI tunnel client, without an inbound port:

```text
ChatGPT ──► OpenAI ◄── tunnel-client (on your PC, outbound HTTPS only) ──stdio──► taskweave ──► claude.exe
```

You need an OpenAI tunnel, its runtime key, the *Tunnels* roles and a ChatGPT plan that allows custom connectors. The control center's **Connect** page writes and proves the launcher, creates the tunnel profile and writes the start script. Start with the guides [Prepare ChatGPT on OpenAI](guides/05-prepare-chatgpt-on-openai.md) and [Connect ChatGPT](guides/06-connect-chatgpt.md); the complete step-by-step walkthrough, the troubleshooting table by layer and what has been verified on which tunnel-client version are in [docs/chatgpt-tunnel.md](chatgpt-tunnel.md).

## Job lifecycle and restarts

- Job state is written to `<jobs dir>/<job_id>/metadata.json` on every change and reloaded at startup.
- On restart: jobs waiting for files keep waiting (an upload saved meanwhile is used); queued jobs are re-queued; a running job whose process is still alive and verified (same PID and start time) is re-attached and keeps its timeout; any other running job becomes `failed` with reason `server_restarted`.
- If a recovery outcome cannot be written to `metadata.json`, `check_task` and `get_task_result` expose `recovery_pending: true`, `recovery_desired_status`, `recovery_desired_reason`, and `recovery_persistence_error`. The degraded recovery state exists only in memory: the old on-disk `metadata.json` is left unchanged as the basis for retry on a later restart. Recovery is not durably recorded while filesystem writes fail, and the job is not reported as successfully recovered.
- On timeout the whole process tree is killed, the job becomes `timed_out`, and partial output is kept.
- If the tree cannot be confirmed stopped (on timeout or cancel), the job stays `running` with `termination_pending: true` and a `termination_error`. For a job this server started, TaskWeave then waits up to 10 seconds for the job's own process to exit, kills it if needed, and on the next watchdog pass gives the job its intended status (`timed_out`, `cancelled` or `failed`) with `termination_pending: false`. The finished job keeps `termination_error`, because only the job's own process was confirmed gone: a child process it started may still be running, so check for leftover processes when you see it.
- Output is written when Claude exits unless `TASKWEAVE_STREAM_OUTPUT=true` is set; with it, `stdout.txt` grows while the job runs and the control center shows a live tail. MCP clients still get the report only when the job ends.

## Threads

Every job belongs to a **thread**: the turns of one client conversation. `start_task` returns a `thread_id`; a client that passes it back on the next call adds that job to the same thread as the next turn (`turn` 2, 3, …). Omit it to start a new thread. An unknown or malformed `thread_id` is rejected before a job is created; when retention has removed every job of a known thread, the error code is `thread_expired` and the message names the retention period, while a never-seen id gets `thread_not_found`. The `thread_id` and `turn` are stored in `metadata.json` and returned by `check_task`, `get_task_result` and `list_tasks`.

A thread only groups jobs for review. Each job still starts a fresh Claude session that does not remember earlier turns, so a follow-up prompt must carry the context it needs. MCP clients do not tell the server which conversation a call came from, so grouping relies on the client passing `thread_id` back; the tool description asks it to.

**Opt-in carry-over.** `start_task(..., thread_id=..., carry_context=true)` prepends the tail of the previous completed turn's report from the same thread (up to a few thousand characters). The block is clearly fenced, labelled as untrusted prior model output, redacted, and shortened further if needed to respect `TASKWEAVE_MAX_PROMPT_CHARS`. It is off by default. If the previous turn is still queued or running, failed, timed out, cancelled, has no saved report, or the thread has expired, the request is rejected with `error_code: carry_context_unavailable` before a job is created; the server never silently sends empty context. If the new job's `permission_mode` is broader than the one that produced the report (for example a `read_write` follow-up after a `read_only` analysis), the context is still carried and the response adds `carry_context_warning`: the report is untrusted model output, so check it before relying on it for changes. The new job records `carry_context_from`, `carry_context_from_mode` and `carry_context_chars` in `metadata.json`. A design note comparing this with CLI session resume lives with the other architecture notes.

## Session viewer (read-only web UI)

Every job folder holds `prompt.txt`, `stdout.txt`, `stderr.txt`, `metadata.json`, `artifacts.json`, and optionally the files sent with the prompt under `prompt-artifacts/` and the files returned with the reply under `reply-artifacts/`. Both are numbered in order (`01.pdf`, `02.png`, ...); job folders from 0.2.1 and earlier use `attachments/` and `artifacts/` instead and are still read. To read them as conversations without opening each folder:

```bash
taskweave-viewer                       # uses TASKWEAVE_JOBS_DIR or the default jobs dir
taskweave-viewer --jobs-dir /path/to/jobs --port 8765
```

The same view is the **Jobs** section of the [control center](#control-center-and-settings-profiles) (`taskweave-control`), which adds a picker for each profile's jobs folder; `taskweave-viewer` is the read-only version on its own.

It opens `http://127.0.0.1:8765/` with a thread list on the left (latest activity first, status of the latest turn, turn count, total cost, first prompt). The selected thread is shown as a timeline: for each turn, the client's request (**ChatGPT → Claude**, with its permission mode and any attachment names) followed by **Claude**'s reply (status, duration, model, effort, cost, the report formatted or raw, and the files Claude returned, each of which opens in place to show its text). stderr and the job's `metadata.json` are folded under each reply. Jobs without a `thread_id` (from older releases or the original prototype layout, `job.json`) appear as single-turn threads. Search covers job and thread IDs, prompt, report and stderr text, and a link ending in `#<job_id>` opens that job's thread scrolled to the turn.

The viewer is strictly read-only, listens on loopback only, rejects foreign `Host` headers, and shows files as stored (no redaction through the viewer). MCP result and text-artifact retrieval apply best-effort redaction. Job files can contain your prompts and Claude's findings, so do not expose the viewer port.

## Artifacts and attachments

See [docs/providers.md](providers.md) for the provider contract and capability discovery.

### Files Claude returns

When a prompt asks for a file back (a CSV table, a Markdown document, JSON, HTML, SVG, or a Mermaid or DOT diagram), Claude does not write it to disk. The prompt wrapper tells it to put each file in its report between marker lines:

```text
<taskweave-artifact filename="rate-limits.csv">
endpoint,limit_per_min
/search,60
</taskweave-artifact>
```

When the job finishes, TaskWeave saves each complete block to `jobs_dir/<job_id>/reply-artifacts/` under the next number (`01.csv`, `02.md`, ...; the extension is the one Claude gave) and replaces it in `report.txt` with a note such as `[Artifact saved: rate-limits.csv as 01.csv (31 bytes). ...]`. The artifact keeps the name Claude gave as its `filename`; `stored_as` says where it is inside the job folder. The client then sees the file in `get_task_result`'s `artifacts` (with a `next` hint) and fetches it with `get_task_artifact`. Because Claude never writes the file itself, this works in every permission mode, including `read_only`.

- Allowed extensions: `.md`, `.markdown`, `.txt`, `.csv`, `.tsv`, `.json`, `.mmd`, `.mermaid`, `.dot`, `.html`, `.svg`. Text only; binary files such as PNG charts are not produced this way.
- The filename must be a plain name (no folders, no Windows device names). A repeated name gets a `-2`, `-3`… suffix; an existing file is never overwritten.
- The usual artifact count and size limits apply. A block that is refused (unsafe name, other extension, over a limit, or never closed) stays in the report unchanged, and the reason is recorded in `metadata.json` under `artifact_extraction_notes`. If saving fails, the job still completes with its full report, and `artifact_extraction_error` says why.
- `stdout.txt` keeps Claude's original output; `metadata.json` lists the names Claude gave under `extracted_artifacts` and the numbered names they were saved as under `extracted_artifact_files`.
- The session viewer lists these files under Claude's reply. Opening one shows its text (Markdown formatted, everything else as plain text; HTML and SVG are never rendered).

Every terminal job attempts to write a report artifact and a job-local `artifacts.json` manifest. The manifest is written after the process exits and records stable IDs, sanitized names, detected media types, sizes, SHA-256 hashes, kinds, timestamps, and `complete`/`partial`/`unknown` status. If manifest finalization fails, the job outcome is preserved but `get_task_result` reports `artifacts_finalized: false` with `artifact_finalization_error`; artifact listings then fall back to the synthetic report entry. The server never claims a finalized manifest that was not written. Only `report.txt` and `jobs_dir/<job_id>/reply-artifacts/` (`artifacts/` in job folders from 0.2.1 and earlier) are scanned; the workspace is not scanned for artifacts. Use `list_task_artifacts` and `get_task_artifact`; larger files return metadata with `too_large` instead of an inline blob. `artifacts_directory` is an opaque `job:<id>` reference, not a host path.

HTML, XHTML and SVG artifacts are returned only as inert plain text (`representation: "plain_text"`, `trusted_renderable: false`) or, when Base64 is requested, as download-only bytes (`representation: "base64_bytes"`). The server does not render them, and the response says so; this is not a claim that the content is safe. The viewer shows them as source text and never renders them. CSV files are returned unmodified with `csv_formula_risk: true` when risk detection finds a cell beginning with `=`, `+`, `-`, `@`, tab or carriage return, plus a warning to neutralize the file before opening it in a spreadsheet. Detection checks comma-, semicolon- and tab-delimited forms conservatively; it is risk detection, not sanitization. In this batch, files created elsewhere in the workspace are **not** automatically copied into the job artifact directory; only `report.txt` and files already under `jobs_dir/<job_id>/reply-artifacts/` are discovered.

`list_task_artifacts` and `get_task_artifact` reject symlinks and Windows junctions, including junctioned `reply-artifacts/` (or older `artifacts/`) directories and subdirectories. The scanner never hashes or retrieves a path that resolves outside those directories (or `report.txt`).

Text artifact retrieval applies best-effort redaction to the returned text and reports `redacted: true|false`. Base64 retrieval returns the original artifact bytes encoded as Base64; it does not apply text redaction to the payload. CSV `csv_formula_risk` metadata and warnings are advisory risk detection, not sanitization. Consumers must treat retrieved artifacts as untrusted content.

Attachment input is staged under `jobs_dir/<job_id>/prompt-artifacts/`, numbered in the order sent (`01.pdf`, `02.png`, ...; the extension comes from the checked media type), with MIME/signature checks, size/count limits, safe filenames, SHA-256 verification, and symlink/junction rejection. **Read-only Claude jobs receive the staged directory as a read-only folder** through the same `--add-dir` mechanism used for data folders, and the wrapped prompt lists each file by position, stored name and media type, for example `attachment 1: 01.pdf (application/pdf)` (the client-supplied filename is kept in `attachments.json` for the viewer and is never put in the prompt). A job takes files once, so nothing is overwritten or renumbered. `read_write` and `full_repository` requests with attachments are rejected before a job is created; those modes can modify or delete files, so they never get the folder. Attachments are never passed as client filesystem paths and no copy is made into the workspace. The client sees only a count and the display names it supplied; the staged path stays on the server.

Treat attached documents as untrusted data, not instructions. The wrapper prompt says so, but the real control is the permission mode: a read-only job cannot write, delete, run commands or widen its mode, whatever a document says. There is no malware scan; PDFs and images are parsed by the provider. The staged bytes live in the job directory and are removed by the same retention as the report; they are never logged. A real-CLI check that the folder is readable and not writable is required before releasing this to untrusted callers. The full threat model is recorded with the other architecture notes in the working tree.

### Files you upload in the control center

ChatGPT cannot pass a file you dropped into the chat on to TaskWeave: it would have to type the whole file out as Base64. For that case a job can wait for you to upload the files on the computer that runs TaskWeave:

1. ChatGPT calls `start_task` with `wait_for_files: true` (the tool description tells it to whenever you have files to send). The job gets its `job_id` at once, with status `waiting_for_files` and an `upload_deadline`; it is not queued and takes no slot from running jobs.
2. TaskWeave opens the control center at that job, starting the control center when none is running, unless a control center page is already open (`TASKWEAVE_OPEN_UPLOAD_PAGE=false` turns this off, and it never happens in a container or a Linux session without a display). Every open control center page shows a banner for each waiting job.
3. Click **Upload files**, choose or drop the files (the same types and limits as `attachments`), and press **Upload and start**. The control center checks them and saves them in the job's `prompt-artifacts` folder as `01.pdf`, `02.png`, ...; the server checks them again with its own limits and starts the job straight away. A job takes one upload.
4. ChatGPT keeps polling `check_task` and fetches the result as usual.

A waiting job can be cancelled from ChatGPT or the control center. One that gets no files within `TASKWEAVE_UPLOAD_WAIT_SECONDS` (default 1800) ends as `failed` with reason `upload_timeout`. Waiting jobs survive a server restart, and an upload saved while the server was stopped is used when it starts. Like attachments, this is for `read_only` jobs only; asking for it with another mode, or together with `attachments`, is rejected before a job is created. You must be at the computer running TaskWeave: the file you gave ChatGPT does not travel, so you upload it again from this computer's disk.

### Compatibility changes from the pre-release behavior

- The project, Python package, commands and default job folder were renamed from `claude-research-bridge` to TaskWeave (`taskweave-mcp` / `taskweave_mcp`); see [Upgrading](#upgrading-from-claude-research-bridge). The old names keep working.
- Settings now use the `TASKWEAVE_` prefix. The `CLAUDE_BRIDGE_*` names (and `CLAUDE_BIN`) are still accepted; see "Older setting names" above for how conflicts are handled. Permission configuration uses `TASKWEAVE_MAX_PERMISSION_MODE` as the canonical setting. The deprecated `MODEL_CLI_MAX_PERMISSION_MODE`, `CLAUDE_BRIDGE_MAX_PERMISSION_MODE`, `MODEL_CLI_ALLOW_EDITS`, and `CLAUDE_BRIDGE_ALLOW_EDITS` settings are constraints, not overrides: all supplied settings must agree or startup fails closed. Explicit empty or whitespace-only permission values are rejected; remove unused legacy variables or set them consistently.
- The child-process environment filter now also removes `TASKWEAVE_*` and `MODEL_CLI_*` (as well as `CLAUDE_BRIDGE_*`), and `PASS_ENV` can no longer re-allow any of them.
- `artifacts_directory` in `get_task_result` is now the opaque reference `job:<id>`, not an absolute host path. This is a deliberate breaking semantic change: the server no longer discloses host paths to MCP clients. Use `list_task_artifacts` / `get_task_artifact` with the job ID instead. There is no compatibility switch that re-exposes host paths.
- A truncated MCP report now ends with `[TRUNCATED: showing <N> of <M> characters. Full report: report.txt in the job directory.]`. Clients that previously received an untruncated report, or that parse report text exactly, should expect this literal suffix; read `report.txt` for the full text.
- A job whose process cannot be confirmed stopped stays `running` with `termination_pending: true` (and `pending_terminal_status` / `pending_terminal_reason`) instead of being reported terminal. It is finalized only after the job's own process is confirmed gone, and keeps `termination_error` (see [Job lifecycle and restarts](#job-lifecycle-and-restarts)).
- Recovered jobs with an unknown, non-empty `provider` value fail with reason `unknown_provider_after_restart`; missing/empty provider metadata still means Claude for legacy jobs.
- Results now include a provider-neutral `error_code` (`provider_reported_error` for a provider's own error envelope; the failure reason otherwise). `reason` is unchanged, including `claude_reported_error`. `session_id` is canonical; `claude_session_id` remains a deprecated alias written only for Claude jobs and is planned for removal in the release after the second provider ships.

These changes are not claimed to be fully backward compatible. See [docs/providers.md](providers.md) for capability details.

## Troubleshooting

Work one layer at a time and do not move on until the current layer works: **CLI → server → transport/tunnel → client discovery → job lifecycle.**

Start with the control center's **Health** section (`taskweave-control`): it checks the settings, CLI and sign-in, workspace, jobs folder and stuck jobs in one go. Then:

1. **Claude CLI.** Run `claude --version` and the Step 1 command above. Fix sign-in or PATH here first. On Windows `TASKWEAVE_CLAUDE_BIN` may be `claude.exe` or npm's `claude.cmd` (TaskWeave runs the `claude.exe` it starts); a message about a missing `claude.exe` behind the `.cmd` means npm skipped Claude Code's install script.
2. **Server.** Run `taskweave --transport stdio` with your env vars (or with `--profile <name> --config <file>`). Configuration errors list every problem at once (missing workspace, missing CLI, bad numbers) and exit with code 2; so does a jobs directory that cannot be created or read. For HTTP, a missing or short token stops startup. stdout carries only MCP messages; all diagnostics go to stderr.
3. **Transport/tunnel.** Locally: `curl -i -X POST http://127.0.0.1:8080/mcp` should return `401`; with `-H "Authorization: Bearer <token>"` it should no longer be `401`. Then test the same through the tunnel. `401` through the tunnel means the header is being dropped or the token is wrong.
4. **Client discovery.** With the default `TASKWEAVE_TOOL_NAMES=both` the client should list fifteen tools: the eight task names (`start_task`, `check_task`, `get_task_result`, `cancel_task`, `list_tasks`, `list_task_artifacts`, `get_task_artifact`, `list_providers`) and the seven older names (`start_research_job`, `check_research_job`, `get_research_result`, `cancel_research_job`, `list_research_jobs`, `list_job_artifacts`, `get_job_artifact`). With `task` or `research` it lists eight. If it shows stale tools, disconnect and reconnect the MCP server in the client to refresh its cached schema.
5. **Job lifecycle.** Use `get_task_result` even after a failure; it reports status, reason, return code, stderr and the artifacts directory. Inspect `<jobs dir>/<job_id>/` (`metadata.json`, `stderr.txt`, `stdout.txt`).
   - Stuck in `queued`: another job is running and `TASKWEAVE_MAX_CONCURRENT` is reached, or another job that can write is using the same folder. `check_task`'s `queued_reason` says which.
   - Stuck in `waiting_for_files`: the job waits for you to upload its files. Open `taskweave-control` on the computer running TaskWeave and use the job's banner, or cancel the job. See [Files you upload in the control center](#files-you-upload-in-the-control-center).
   - `failed` with `upload_timeout`: no files were uploaded within `TASKWEAVE_UPLOAD_WAIT_SECONDS`. Start the task again.
   - `failed` with `unsupported_attachment` after an upload: the server's own check refused the uploaded files (type, size or count, or a file changed after the control center checked it).
   - `failed` with `failed_to_start`: the CLI path or workspace is wrong.
   - `timed_out`: raise `TASKWEAVE_JOB_TIMEOUT_SECONDS` or narrow the task.
   - `Queue is full`: wait, or raise `TASKWEAVE_MAX_QUEUE`.
   - **`permission_mode ... is not permitted on this server`**: the operator's ceiling is lower than the request. Raise `TASKWEAVE_MAX_PERMISSION_MODE` (only if you accept the risk) and restart the bridge. Note that a job already queued under a mode above a lowered ceiling fails with `permission_mode_rejected` when it would start.
   - **`Invalid permission_mode`**: only `read_only`, `read_write` and `full_repository` are accepted, exactly as written.
   - **A job reports that a tool or command was denied or unavailable**: that is the mode working. The report usually says which one. Start a new job in the mode you need (a running job's mode cannot be changed). In `read_write`, shell commands never exist; in `full_repository`, anything off the allow-list (see docs/permission-modes.md) is denied, including `rm`, `mv`, `git push` and installs.
   - **Claude cannot see a file you expect**: file tools only reach the workspace, and a symlink or junction that points outside it is refused.
   - **`web` research fails**: web tools are off unless `TASKWEAVE_ALLOW_WEB=true`.
   - **A needed environment variable is missing for Claude** (for example Bedrock or Vertex credentials): credentials are scrubbed by default. List the names in `TASKWEAVE_PASS_ENV` (see [Sign in with an API key, Bedrock or Vertex](#sign-in-with-an-api-key-bedrock-or-vertex)).
   - `failed` with `cli_missing_required_flags`: the installed Claude CLI is too old for the restriction flags. Upgrade it. The bridge refuses to run without them.
   - **`Model '...' is not allowed here`** or **`Effort '...' is not allowed here`**: the server restricts models or effort levels (`TASKWEAVE_MODELS`, `TASKWEAVE_EFFORTS`, or the profile's lists). `list_providers` shows the allowed values under `job_choices`.
   - **A file you asked for is not in `artifacts`**: Claude must return it in a `<taskweave-artifact filename="...">` block with an allowed extension. A refused block stays in the report, and `metadata.json` says why under `artifact_extraction_notes`.
   - **A follow-up became a new thread**: the client did not pass the earlier `thread_id`. Ask it to. `thread_expired` means the thread's jobs were removed by retention (the message names the period); start a new thread. `thread_not_found` means the ID was never seen or is wrong.

### Windows: a profile or settings file is not found

**Symptom:** TaskWeave stops at start-up with `Profile '<name>' was not found in ...\AppData\Local\taskweave\taskweave.toml (profiles: none)` although the file exists and contains that profile (the control center shows it). Started by the OpenAI tunnel client, this looks like the tunnel stopping right after it starts, with `stdio MCP command exited ... exit status 2` in its log.

**Cause:** the program that started TaskWeave runs inside a packaged Windows app, typically the **Microsoft Store version of PowerShell** (`C:\Program Files\WindowsApps\Microsoft.PowerShell_...\pwsh.exe`, check with `$PSHOME`). Windows gives programs started from it a private view of `%LOCALAPPDATA%`, so files created there by other programs (the control center, or a regular PowerShell) are invisible to them. `Test-Path` on the file returns `False` from that window. The default jobs folder `%LOCALAPPDATA%\taskweave\jobs` can be affected the same way (jobs written there would not show up in a viewer started elsewhere).

**Fix (recommended):** keep the settings file outside `%LOCALAPPDATA%`, next to your launcher, and pass it explicitly to both commands:

```bat
"%ROOT%venv\Scripts\taskweave.exe" --profile chatgpt --config "%ROOT%taskweave.toml" --transport stdio
"%ROOT%venv\Scripts\taskweave-control.exe" --config "%ROOT%taskweave.toml"
```

Set `TASKWEAVE_JOBS_DIR` (or the profile's jobs folder) to a folder outside `%LOCALAPPDATA%` too. Alternatively, start the tunnel client from the regular PowerShell (`C:\Program Files\PowerShell\7\pwsh.exe`) or Windows PowerShell. Check the result by running the launcher by hand from the same window: `$null | cmd /d /c <your launcher>` should exit without printing `Invalid TaskWeave configuration`.

## Writing good prompts

Ready-to-use prompts for common jobs (a codebase tour, a bug review, a security pass, document summaries, data questions, documentation, tests, release notes), each with the permission mode it needs: [docs/task-templates.md](task-templates.md).

State the task and its boundaries, which checks must not be done (the permission mode, not the prompt, decides what Claude is allowed to do; the prompt cannot widen it), and ask Claude to separate verified facts from inferences and list what it did and did not inspect. A zero exit code does not mean the conclusions are correct.

## License

TaskWeave is proprietary. Using it is licensed only under the [End User License Agreement](../EULA.md), which the installer shows and asks you to accept. See `LICENSE`.

