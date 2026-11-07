# Changelog

All notable changes to TaskWeave. Releases are published on the public TaskWeave releases page (https://github.com/subhash7476/taskweave-releases/releases); TaskWeave is not published to PyPI.

## 0.3.0rc2 (2026-11-07)

Second release candidate for 0.3.0. It adds folder routing to `0.3.0rc1`: a prompt can pick its working folder with a `#name` tag or the folder's full path, every response says where a job runs, two jobs that write never share a folder, folder changes in the control center apply to the next job, and folder groups make every repository in a parent folder a working folder. Install it with `-Tag v0.3.0rc2` (`--tag v0.3.0rc2` on macOS and Linux), and refresh the TaskWeave connector in ChatGPT afterwards, because ChatGPT keeps the tool list it saw last.

The design is in `docs/architecture/folder-routing-plan.md`. In a real ChatGPT test on 0.3.0rc1, two jobs about a working folder ran in the workspace because ChatGPT put the folder's path, then its name, into the prompt instead of passing `folder`.

### Added

- **The prompt can pick the folder.** When the client passes no `folder`, a `#name` tag in the prompt (`#nifty`, any case) or the full path of a working folder now picks that folder, so a client that sends only a prompt, like ChatGPT, still reaches it. The order is: the `folder` argument, a tag, a follow-up's thread, a path, the default folder. Only the operator's working folders can be picked, so this grants nothing a client could not already ask for. Ambiguity is refused, never guessed: two different tags or paths in two unrelated folders return the new error code `folder_ambiguous`, and a tag that disagrees with `folder` returns `folder_conflict`. Paths are compared as text (on Windows without regard to case or slash style, in both long and 8.3 short form) and never opened; bare folder names in ordinary text never count. The tag is `#`, not `@`, because `@` opens ChatGPT's own menu. `TASKWEAVE_FOLDER_ROUTING=explicit` (*Folder from the prompt* in the control center) turns this off; the Connect page's launcher carries the setting.
- **Where a job runs, in every response.** `start_task`, `check_task` and `get_task_result` return `folder`, `folder_source` (`argument`, `prompt_tag`, `thread`, `prompt_path` or `default`) and `folders_available` (each folder's name and the highest mode a job may use there, never a path). `list_tasks` shows each job's folder. Every job now records `folder` in `metadata.json`, the workspace included, plus `folder_source`.
- **Warnings.** When the prompt names an absolute path the job cannot reach, the start, status and result responses carry `warnings`: one sentence per path, naming the working folder that holds it when there is one and saying how to start a job there (`folder=<name>` or `#name`). A tag that did not pick the folder is flagged too: an unknown tag that looks like a folder name (`#nifyt`) on a job left in the default folder, or any folder tag when routing is explicit. The prompt is only read as text; no path in it is opened.
- The `start_task` description names this server's folders, each with the highest mode it allows, the default, and the `#name` rule and order. Refresh the connector in ChatGPT to see it.
- **Folder changes apply to the next job.** A server started with a settings profile re-reads its folder settings before each job (the working folders, the new aliases and descriptions, the default folder and mode, folder routing, and the new `TASKWEAVE_SHOW_FOLDER_PATHS`), so adding a folder in the control center needs no restart and no new launcher. A broken edit is refused and the last good folder list stays in force. The workspace, jobs folder, permission cap, CLI paths and transport still need a restart. The Connect page's launcher now points at its profile and sets the new `TASKWEAVE_FOLDER_SETTINGS=profile`, so its own copies of the folder settings are only a fallback while the settings file cannot be read; its folder copies no longer count as drift. Health shows which settings file the running server reads (from its heartbeat) and warns when the server reads another copy, kept a refused edit, or reads a different file than the control center edits.
- **Folder groups.** `TASKWEAVE_FOLDER_GROUPS` (*Folder groups* in the control center) registers a parent folder once, with one highest mode; each immediate subfolder becomes a working folder named after it, and a repository cloned into it later is usable for the next job without a settings change. `git` takes only subfolders with `.git`. Links, junctions, invalid names, the jobs folder and taken names are skipped with a note in Health; a folder listed by name wins over a subfolder with the same name.
- **Folder aliases and descriptions.** `TASKWEAVE_FOLDER_DETAILS` (*Folder aliases and descriptions*) gives a folder other names, usable as `#tags` and as the `folder` argument, and a description that clients see in `folders_available` and the `start_task` description. The folder argument now also accepts a folder name in another case.
- **Copy ChatGPT instructions.** The control center's Settings page shows a short text naming the profile's folders and how to tag them, with a button to copy it into ChatGPT's custom or project instructions. It never includes paths.
- `TASKWEAVE_SHOW_FOLDER_PATHS` (off by default) adds each folder's path to `folders_available`.
- **One writing job per folder.** At most one job that can write (`read_write` or `full_repository`) runs in a working folder at a time, a folder nested in another counting as the same; `read_only` jobs run alongside it. The queue starts the first job whose folder is free, so a busy folder no longer holds up jobs in other folders. `check_task` on a queued job adds `queued_reason`, saying which job holds the folder or that every job slot is in use. Health suggests raising *Jobs at a time* above 1 when there are working folders.

### Changed

- **A thread keeps its folder.** A follow-up turn without `folder` or a tag runs in its thread's folder instead of the operator's default. A follow-up whose `folder` or tag names a different folder is refused with the new error code `thread_folder_mismatch`, and one whose folder the operator has since removed with `unknown_folder`.
- When the client names no permission mode, the default mode is lowered to fit the job's folder if that folder allows less (never raised). Before, a job sent to a `read_only` folder under a `read_write` default was refused with `permission_mode_rejected`.
- The control center's Settings have a new *Working folders* group (the folders, aliases and descriptions, default folder and mode, folder routing and folder paths), marked as applying to the next job. *Default working folder* and *Default permission mode* were settings already but missing from the form.

### Fixed

- `unknown_folder` is now in the published list of error codes; it was returned before but missing from the list. A test now checks that every code a job can raise is listed.

## 0.3.0rc1 (2026-11-06)

First release candidate for 0.3.0. It adds files you upload in the control center for a ChatGPT job, and numbered files in their own job folders, to `0.2.1`; install it with `-Tag v0.3.0rc1` (`--tag v0.3.0rc1` on macOS and Linux).

### Added

- **Files you upload for a ChatGPT job.** ChatGPT cannot pass on a file you dropped into the chat, so `start_task` takes `wait_for_files: true`: the job is created with the new status `waiting_for_files` and waits, unqueued, while you upload the files in the control center on the computer running TaskWeave. The server opens the control center at the job (starting it if needed) unless a page is already open, and every open page shows a banner with an **Upload files** button. The job starts as soon as the files are saved in its `prompt-artifacts` folder; the server checks them again with its own limits first. Read-only jobs only, one upload per job, the same types and limits as `attachments`. A job with no upload within `TASKWEAVE_UPLOAD_WAIT_SECONDS` (default 1800) fails with reason `upload_timeout`; `TASKWEAVE_OPEN_UPLOAD_PAGE=false` stops the page from opening. Waiting jobs survive a restart and can be cancelled. Not yet tried through a real ChatGPT tunnel.
- The `start_task` description now names the fields of an attachment (`filename`, `media_type`, `data_base64`, optional `sha256`).

### Changed

- **Numbered files in each job folder.** Files sent with a prompt are now saved in the job's `prompt-artifacts` folder and files Claude returns in its `reply-artifacts` folder, numbered in order: `01.pdf`, `02.png`, and so on. The original names are kept: a sent file's name in `attachments.json` (shown in the viewer, never put in the prompt), a returned file's name as the artifact's `filename`, with the new `stored_as` saying where it is in the job folder. The report's note now reads `[Artifact saved: summary.csv as 01.csv (...)]`. Job folders from 0.2.1 and earlier (`attachments`, `artifacts`) are still read.

### Docs

- The user guides explain sending files with a read-only task and uploading them in the control center, with the new troubleshooting entries (`waiting_for_files`, `upload_timeout`, a refused upload).
- The README says what happens when a timed-out or cancelled job's process tree cannot be confirmed stopped: TaskWeave kills the job's own process, finishes the job, and keeps `termination_error`, because a child process the job started may still be running. This is how earlier releases behaved too.

## 0.2.1 (2026-11-04)

Fixes jobs from ChatGPT and Claude Desktop that ignored the working folder set in Settings (see the first entry below). Install it over 0.2.0 with `taskweave update`.

### Added

- **Default permission mode and default working folder.** New settings (`TASKWEAVE_DEFAULT_PERMISSION_MODE`, `TASKWEAVE_DEFAULT_FOLDER`, or *Default permission mode* and *Default working folder* in Settings) say what a job gets when the client asks for nothing. ChatGPT and Claude Desktop usually send only a prompt, so before this every such job ran `read_only` in the workspace, and a working folder you added (for example a drive folder in Full Repository) was never used. A default can never exceed the permission cap or the folder's own mode; TaskWeave refuses to start and says which. The Connect page's launcher carries both settings, and `list_providers` shows them.
- **Stale launcher check.** The Connect page's variables launcher copies the settings into itself when it is written. The first-job check now compares it with the saved profile (workspace, data folders, working folders, permission cap, defaults) and says plainly when it is out of date and must be written again.

### Changed

- `start_task` no longer advertises `permission_mode` as defaulting to `read_only` in its schema. A client fills defaults from the schema, which stopped the operator's default from applying. The server still uses `read_only` unless a default mode is set.

## 0.2.0 (2026-11-04)

The first final release of 0.2.0. It adds everything below to `0.2.0rc3`, which came out of a full review of the release candidates; the install, update and uninstall flow changed the most, so test a new install before relying on it.

### Added

- **Install report.** Every install writes `install-report.json` in the install folder (system, Python and TaskWeave versions, which steps worked, `doctor`'s check names and statuses; no paths, user names or file contents), and a failed install names it so it can be attached to a question. It is sent to the maintainer only with your yes, and the installer asks only when the release has a report address: this release has none, so nothing is sent. `--send-report` / `--no-send-report` (`-SendReport` / `-NoSendReport`) answer for unattended installs.
- **Task names for the tools.** The tools now have names that fit any task: `start_task`, `check_task`, `get_task_result`, `cancel_task`, `list_tasks`, `list_task_artifacts` and `get_task_artifact` (`list_providers` is unchanged). The older `start_research_job` names stay next to them and run the same code, so existing setups and saved prompts keep working; a client now lists fifteen tools. `TASKWEAVE_TOOL_NAMES` (or **Settings** > Tool names) picks `both` (default), `task` or `research`. Replies point to the names the server offers.
- **Codex CLI provider (switched off).** An adapter for OpenAI's Codex CLI ships with this release. It offers no permission mode until `scripts/verify_permissions.py --provider codex --candidate` passes with the real CLI; until then start-up refuses `codex` in `TASKWEAVE_PROVIDERS` and says how to verify it. See `docs/providers/codex.md`.
- **Docker image.** `packaging/docker/Dockerfile` runs TaskWeave's HTTP server with Claude Code inside a container as an unprivileged user, signed in with an API key; see `docs/docker.md`.
- **Claude Desktop extension.** `TaskWeave-<version>.mcpb` installs TaskWeave into Claude Desktop with a double-click: no Python and no installer. Claude Desktop installs the tested dependency versions itself (MCPB `uv` type) and asks for the workspace folder, optionally the Claude Code program and jobs folder, and the spending limits.
- **Spending limits.** New daily and monthly budgets for all jobs together (`TASKWEAVE_DAILY_BUDGET_USD`, `TASKWEAVE_MONTHLY_BUDGET_USD`, or **Settings**). Once a budget is used up, new jobs are refused with `budget_exceeded` until it resets; each job's cost is recorded in its metadata (`cost_usd`) and in `spend.jsonl`. The installer's starter profile now caps a job at $2 and all jobs at $10 a day, and Health shows what was spent.
- **Copy diagnostics.** Health in the control center has **Copy diagnostics**, and `taskweave doctor --support` prints the same support report: versions and every check, with secrets, home-folder paths, the user name, e-mail addresses and tunnel IDs removed, ready for an issue.
- **Updates.** `taskweave update` installs the newest public release with its own installer (checked against GitHub's checksum); `--check` only reports, `--tag` picks a release. The control center shows a notice when a newer release exists, checking at most once a day; `TASKWEAVE_UPDATE_CHECK=false` turns that off.
- **Uninstall.** `taskweave uninstall` removes what the installer added (the program, shortcuts, PATH entries, a private Python) and asks first; settings and jobs stay unless `--remove-data`, and the workspace is never removed. On Windows TaskWeave is also listed under **Settings, Apps** with an **Uninstall** button.
- **Control center shortcuts.** `taskweave-control --background` opens the running control center, or starts one without a window first; `taskweave-control --stop` stops it. The installer adds **TaskWeave Control Center** and **Stop TaskWeave Control Center** shortcuts (Start menu, `~/Applications`, the Linux app menu), so the control center is one click away after a restart and no longer needs Task Manager to stop.
- **Commands on PATH.** After an install, `taskweave`, `taskweave-control` and `taskweave-viewer` work in a new terminal: on Windows through `<install folder>\bin` on the user PATH, on macOS and Linux through `~/.local/bin`. `-NoPath` / `--no-path` skips it. This makes the `taskweave --version` steps in guides 02 and 09 work as written.
- **One download.** Each release has one zip per system (`TaskWeave-<version>-<system>.zip`: Windows x64, macOS Apple silicon and Intel, Linux x64 and arm64; there is no Windows on ARM zip in 0.2.0, because `cryptography` and `cffi` have no Windows ARM64 build at the tested versions) holding the installer, the wheel, the pins, the EULA, checksums and every dependency. Extract it and double-click `install.cmd`, or run `sh install.sh`. `install.cmd` now says what to do when it is run from inside an unextracted zip.
- **Offline install.** A `wheelhouse` folder of dependency wheels next to the installer (the install bundles carry one) is used first, with no network at all; PyPI is used only when those wheels do not fit the machine's Python.
- **No Python needed.** When no Python 3.10 or newer is found, `install.cmd` / `install.ps1` and `install.sh` download Astral's uv (pinned by version and SHA-256, from PyPI) into the install folder, and uv installs a private Python 3.12 there for TaskWeave. PATH, the registry and any other Python are left alone; a download that does not match its pin stops the install.
- **Public download.** Releases are published on the public [TaskWeave releases page](https://github.com/subhash7476/taskweave-releases/releases/latest); no GitHub account, invite or `gh` is needed. With no wheel next to it, the installer downloads the latest release over HTTPS (or `-Tag` / `--tag`) and checks every file against the SHA-256 GitHub reports. `-Gh` / `--gh` keeps the GitHub CLI download for a private repository.
- **Licence agreement.** TaskWeave is free to use under its End User License Agreement (`EULA.md`). The installer shows where it is and asks once per version of the agreement; `-AcceptEula` / `--accept-eula` answers for unattended installs, and nothing is installed without a yes.
- `constraints.txt`: the dependency versions a release was tested with. The installer passes it to pip, so a new install gets those versions instead of whatever PyPI has newest; if pip cannot install them, it tries once more without them and says so. It ships with every release and is checked against `SHA256SUMS.txt`.

### Fixed

- **Claude Code installed with npm on Windows.** `claude.cmd` was refused, which was the most common Windows setup failure. Since Claude Code 2.x, npm's `claude.cmd` only starts a native `claude.exe` inside npm's `node_modules`; TaskWeave now runs that program directly (never `cmd.exe`), and explains when npm skipped its install script and left a placeholder instead.
- **Windows user names with accents or other alphabets.** The Connect page's launcher refused any path with a non-ASCII character, and the default install folder is under the user's profile, so a user named "José" could not use the ChatGPT path. It now writes Windows' plain short (8.3) name for such a folder, and only refuses when the drive has short names turned off, with the fix (`-Root C:\TaskWeave`) in the message.
- **Intel Macs.** cryptography (needed by the MCP SDK) stopped publishing Intel-Mac wheels in 49.0, so pip would try to build it from source there and the install would fail. TaskWeave now requires `cryptography<49` on Intel Macs, and `constraints.txt` pins 48.0.1 everywhere.
- The README's `gh` install command now downloads `SHA256SUMS.txt` too; before, that path installed the wheel without checking it. The README and guide 02 no longer tell you to add `--tag` to `gh release download`, which has no such flag: put the tag right after `download`.
- The starter workspace's `README.txt` now says jobs are read-only by default, instead of saying the agent can edit files there.
- **Add to Claude Desktop** changes only Claude Desktop's own config file in the places Claude Desktop keeps it. Before, the page would preview and write any file named `claude_desktop_config.json`.
- Windows: a job's `metadata.json`, `report.txt`, manifests, cancel requests and the settings file are written even while another program (the viewer, the control center, a virus scanner) is reading the old file. Before, the write could fail with `PermissionError`, which is the likely cause of the occasional failure of the timeout test on Windows.

### Docs

- **Task templates.** `docs/task-templates.md` has eleven ready-to-use prompts (a codebase tour, a bug review, a security pass, uncommitted changes, document summaries, a document comparison, a data question, a research memo, documentation, tests, release notes), each with the permission mode it needs and the files it returns. It is published with the guides and linked from guide 08 and the README.
- **Landing page.** The public page now starts with what TaskWeave is for, a table of the ways in (the Claude Desktop extension, the full install, ChatGPT, Docker, other clients), the sign-ins it works with, spending limits and a short safety summary.
- **Docker from the release files.** Releases now carry the `Dockerfile`, so the image can be built from the release page without the source code (`docs/docker.md`, "Build").
- **Sign in with an API key, Bedrock or Vertex.** A README section lists the variables for each way Claude Code can sign in, which of them reach Claude Code by themselves and which need `TASKWEAVE_PASS_ENV`, where to set them for a terminal, Docker and Claude Desktop, which one Claude Code picks when several are present, and how to check it.

## 0.2.0rc3 (2026-11-03)

Third release candidate for 0.2.0. It adds named working folders and the launcher fix below to `0.2.0rc2`; install it with `-Tag v0.2.0rc3`.

### Added

- **Named working folders**, each with its own highest permission mode (`TASKWEAVE_FOLDERS`, or *Working folders* in a settings profile). `start_research_job` takes `folder="<name>"`; the job runs with that folder as its working directory, so write access exists only there. A job's mode must not exceed the folder's mode or the server's permission cap, and a path is never accepted. `list_providers` shows each folder's name and highest mode (no paths); Health lists the folders. One working folder per job: Claude Code 2.1.286 did not enforce deny rules for extra folders, so mixed access in one job is not offered.

### Fixed

- The Connect page's variables launcher now carries the profile's read-only data folders (`TASKWEAVE_DATA_DIRS`) and working folders. Before this it silently left them out, so a data folder saved in Settings never reached the job. Write the launcher again after changing folders.

## 0.2.0rc2 (2026-11-03)

Second release candidate for 0.2.0. It adds the fixes below to `0.2.0rc1`; install it with `-Tag v0.2.0rc2`.

### Fixed

- Claude Code 2.1.317 prints `--output-format json` as one JSON array of every session event when `--restricted` is passed. TaskWeave now takes only the final `result` element, in jobs, the viewer and the setup test, and never returns the other events (they hold host paths, the user name and tool output). Before this fix every job's report was the raw array, cut short, and the setup test failed with *The reply did not contain the expected text*.
- An error result with no text of its own (for example `error_max_budget_usd`) now has a readable report naming the cause, instead of an empty one.
- The setup test's cost cap is $0.50 instead of $0.05. Creating Claude Code's prompt cache alone can cost about $0.25 on a large model, so $0.05 stopped the CLI before it replied.

### Docs

- Windows: the guides, the README and `taskweave doctor` now explain that an npm install of Claude Code gives a `claude.cmd` shim TaskWeave refuses, how to install the native `claude.exe` (`irm https://claude.ai/install.ps1 | iex`), how to check with `where.exe claude`, and how to remove the npm copy.

## 0.2.0rc1 (2026-11-03)

First release candidate for 0.2.0, and the first release with a browser installer, user guides and a Claude Desktop button. Do not use `0.2.0rc1` with Claude Code 2.1.317 or newer, whose output it cannot read (fixed in `0.2.0rc2`).

### Install

- A browser-first Windows install: download the wheel, `install.ps1`, `taskweave_install.py`, `install.cmd` and `SHA256SUMS.txt` into one folder, then double-click `install.cmd`. The installer finds the wheel itself; no command and no file name are typed.
- `install.cmd` runs `install.ps1` with `-ExecutionPolicy Bypass`, which is needed because a browser-downloaded script carries the mark that the default Windows policy refuses. It forwards arguments and the exit code, and pauses only when `TASKWEAVE_NO_PAUSE` is unset.
- The installer uses the one TaskWeave wheel next to it when neither `--wheel` nor `--tag` is given. Two wheels stop and are named; a browser-renamed `... (1).whl` is ignored and named.
- `scripts/make_checksums.py` writes `SHA256SUMS.txt` in GNU format. The installer checks a local wheel against it and stops on a mismatch with a plain message; an explicit `--sha256` still wins, and the `gh` path still uses GitHub's digest.
- Prerequisite messages name the missing program, its official download page and the command that proves it. The installer still does not install other vendors' software.

### Setup, Claude Desktop and ChatGPT

- The first-run setup wizard: folders, health checks, a capped tool-free test task, and the client connection step (PR 23).
- **Add to Claude Desktop**: the Connect step finds `claude_desktop_config.json`, previews the entry, keeps a byte-exact backup and writes atomically. A file that changes after the preview is refused with 409; a file held by a running Claude Desktop is refused with the quit-Claude-Desktop message. The page never shows other servers' values, and it never creates a missing config file.
- The Connect page: launcher generation and proof, tunnel-client detection and offline check, tunnel profile creation, start-script generation with a masked runtime-key prompt, `/readyz`, the readiness light and the first-job watcher (PRs 24 to 26).
- ChatGPT setup is Windows-only by design and still needs OpenAI's tunnel, a runtime key, the *Tunnels* roles and a plan that allows custom connectors.

### Jobs, permissions and clients

- Stable `error_code` values and provider-neutral result fields (PR 13).
- Atomic report writes, expired-thread errors and the distribution check (PR 14).
- Slow job work outside the manager lock (PR 15).
- `taskweave doctor`, the Windows installer and opt-in desktop notifications (PR 16).
- Opt-in thread carry-over (PR 17).
- Read-only attachments (PR 18).
- Cancel a job from the control center (PR 19).
- A timing-race fix in the termination-failure test (PR 20).
- Opt-in live tail for running jobs (PR 21).
- Threads, returned files as artifacts, the control center with profiles and per-job model/effort, the Health section, read-only data folders, and the provider-neutral layer (PRs 4 to 10).
- Every job is read-only by default. `read_write` and `full_repository` are opt-in and capped by the operator's ceiling; `full_repository` is best effort, not a sandbox.

### Guides and release engineering

- Eleven task-focused guides under `docs/guides/`, linked from the README's first screen. Each has the same shape, a plain-words glossary and an "If it does not match" table.
- `tests/test_guides.py` keeps the guides true: links and anchors, required sections, real command flags, tunnel-client commands tied to the verified-versions manifest, step budgets, the plain-words rule and a hygiene message for the guide folder.
- `scripts/compare_dists.py` makes the candidate-to-final carry-over mechanical: only the version string, `README.md` and `docs/guides/` may differ.
- One release-asset list in `scripts/release_assets.py`, checked against the README's release and checksum commands.
- The verification template and record live under `docs/release/`; they do not ship in the sdist.

### Upgrading from 0.1.0

1. Download the new release's wheel and installer files and run the installer again. Your settings file and jobs are kept; a newer version upgrades the package in place.
2. Run `taskweave doctor --json` and compare the version with `taskweave --version`.
3. If you connected Claude Desktop by hand, either keep the existing entry or use the new **Add to Claude Desktop** button. The button writes the profile-based entry and keeps a backup.
4. If you connected ChatGPT, the Connect page can write the same launcher and start script it writes for a new setup. The runtime key is still entered only in the start script's own prompt.

### Compatibility changes

See the README's [Compatibility changes from the pre-release behavior](docs/reference.md#compatibility-changes-from-the-pre-release-behavior). The old `claude-research-bridge` command names and `CLAUDE_BRIDGE_*` settings still work for now.

### Known limits

- The ChatGPT path is Windows-only.
- Claude Desktop is not an official Linux application; the guides use the snippet on Linux.
- The release is private. There is no PyPI package and no public download.
- The installers are unsigned. Windows may show a download warning; the guides explain the safe two clicks for a file from this private repository.
- Real-credential runs A to I are recorded in the verification record for this release. Items not yet verified are listed there and in the README's checklist; the notes here do not claim more than that record.

### Files in this release

- `taskweave_mcp-0.2.0-py3-none-any.whl`
- `taskweave_mcp-0.2.0.tar.gz`
- `install.ps1`, `install.sh`, `taskweave_install.py`, `install.cmd`
- `SHA256SUMS.txt`

The checksum file protects against a wrong, partial or corrupted download. It does not protect against a malicious release: whoever can replace the wheel can replace the checksum file next to it.

### Pull requests since v0.1.0

| PR | Title |
|---|---|
| 3 | Explain installing, configuring and starting the tunnel client on a new machine |
| 4 | Group jobs into threads and show them as conversations in the viewer |
| 5 | Save text files Claude returns in its report as job artifacts |
| 6 | Add a control center with settings profiles and per-job model and effort |
| 7 | Bring the README up to date with threads, returned files and the control center |
| 8 | Add a Health section to the control center |
| 9 | Make the provider layer provider-neutral (multi-provider groundwork) |
| 10 | Add read-only data folders for read_only jobs |
| 11 | Make CI manual-only: Ubuntu by Run workflow, macOS on Azure, Windows local |
| 12 | Add a manually started Azure Pipelines run for macOS |
| 13 | Add stable error codes, neutral result fields and viewer deep links |
| 14 | Hygiene: atomic report, expired-thread errors, dist check, ruff |
| 15 | Keep slow job work outside the manager lock |
| 16 | Add taskweave doctor, a Windows installer and opt-in desktop notifications |
| 17 | Add opt-in thread carry-over |
| 18 | Deliver attachments to read_only jobs |
| 19 | Cancel a job from the control center |
| 20 | Fix a timing race in the termination-failure test |
| 21 | Add an opt-in live tail for running jobs (TASKWEAVE_STREAM_OUTPUT) |
| 22 | Stage 1: one-command installer (standalone module, thin shims, gh download with SHA-256 check) |
| 23 | Add first-run setup wizard and tool-free taskweave run-test (onboarding stage 2) |
| 24 | Add Connect page, launcher generator and proof, and --setup-check (tunnel step PR A) |
| 25 | Connect: create the tunnel profile and write the start script (Stage 3, PR B1) |
| 26 | Connect: readiness light and first-job watcher (Stage 3, PR B2) |

The 0.2.0 work on this branch (the browser installer, checksums, the Claude Desktop button, the guide checker and guides, the comparison tool, this changelog and the verification template) becomes pull requests before the candidate is tagged; its pull requests will be added to this table when they merge.

## 0.1.0 (2026-10-30)

The first private release: the TaskWeave rename, `TASKWEAVE_*` settings, the private GitHub Release procedure, the MCP tools, background jobs with a fixed workspace, and the first tunnel-client setup notes. Pull requests 1 and 2.
