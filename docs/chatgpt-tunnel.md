# Connect TaskWeave to ChatGPT using the OpenAI tunnel client (Windows)

This is the full Windows walkthrough for connecting ChatGPT to TaskWeave through the OpenAI tunnel client. The [user guides](guides/README.md) ([Prepare ChatGPT on OpenAI](guides/05-prepare-chatgpt-on-openai.md) and [Connect ChatGPT](guides/06-connect-chatgpt.md)) are the shorter path; the control center's Connect page automates most of the machine side. The rest of the reference is in the [README](reference.md).

This connects ChatGPT to TaskWeave on your own Windows PC without opening an inbound port:

```text
ChatGPT ──► OpenAI ◄── tunnel-client (on your PC, outbound HTTPS only) ──stdio──► taskweave ──► claude.exe
```

The OpenAI tunnel client starts TaskWeave itself and talks to it over **stdio**, so you do not need `--transport http` or `TASKWEAVE_AUTH_TOKEN` for this setup. There are two separate pieces, and you need both:

1. **The tunnel runtime on your PC** (`tunnel-client run`). A healthy tunnel does **not** create anything in ChatGPT by itself.
2. **A custom MCP connector in ChatGPT** that uses that tunnel. You create it once, by hand, while the tunnel is running.

These steps were tested on Windows 11 with tunnel-client 0.0.17 and 0.0.18. Version 0.0.19 (released 2026-10-30) restarts a stdio child that exits instead of shutting the client down. Offline only, these were re-run on it (the zip's SHA-256 matched what GitHub reports): `--version`, `init`, `profiles list`, `doctor` (exit code 2 on failure), the generated profile's `tunnel_id`, `api_key` and `listen_addr` lines, `dev proxy` in front of a launcher, and `MCP_STDIO_FAILURE_POLICY=exit` through `dev proxy`. Note that 0.0.19 keeps profiles in `~/.config/tunnel-client` by default (0.0.17 and 0.0.18 used `%APPDATA%\tunnel-client`); Connect always passes `--profile-dir`. Not re-run on it: `/readyz` and a real `run` (they need a key and a tunnel); the rest of this guide was not re-tested on 0.0.19, and where it describes the old behavior it says so. OpenAI's pages and ChatGPT's menu labels change over time, so when they differ from this guide, follow the official documentation: [Secure MCP Tunnel guide](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels) and the [openai/tunnel-client](https://github.com/openai/tunnel-client) repository. `tunnel-client.exe help quickstart` and `tunnel-client.exe help troubleshooting` print the current links and commands for the version you have.

## Prerequisites

- **TaskWeave** installed in its own virtual environment (see [Install](reference.md#install)). This guide uses `C:\TaskWeave\venv`.
- **Claude Code** installed and signed in (the native installer or npm; for npm, TaskWeave runs the `claude.exe` behind `claude.cmd`). `claude -p "Reply with exactly BRIDGE_JOB_OK. Do not use tools."` prints `BRIDGE_JOB_OK`.
- **The OpenAI tunnel client** (`tunnel-client.exe`). Step 3a shows which file to download and where to put it.
- **On the OpenAI Platform (organization settings):** a tunnel, whose **tunnel ID** you can see in Tunnels management, and a **runtime API key** for the tunnel client. Running the tunnel client and selecting the tunnel in ChatGPT need the *Tunnels Read + Use* permissions; creating or editing a tunnel needs *Tunnels Read + Manage*. An organization owner or RBAC administrator grants these. A new role can take up to 30 minutes to take effect.
- **In ChatGPT:** a plan and workspace in which you are allowed to add custom MCP servers. Which plans offer this, and whether a workspace admin has to allow it, is decided by OpenAI and your admin; check the official guide. The tunnel must be associated with that ChatGPT workspace, not only with a Platform organization.
- **A dedicated folder for the first test.** This guide uses:

  ```text
  C:\TaskWeave\
    venv\                    TaskWeave's virtual environment (from Install)
    taskweave-stdio.cmd      the launcher (Step 2)
    workspace\               a DISPOSABLE folder Claude may read; create it now
    jobs\                    job folders; created on first start
    taskweave-stderr.log     TaskWeave's warnings and errors; created on first start
    tunnel-client\           tunnel-client.exe, cloudflared.exe and their files (Step 3a)
  ```

  The tunnel client keeps its profile (configuration) file in `%APPDATA%\tunnel-client\`, not in this folder (Step 3c).

  Put one harmless file in it, for example `C:\TaskWeave\workspace\readme.txt` containing `hello from the disposable workspace`. Do not point the first test at a real project.

## Step 1: Verify TaskWeave locally

```powershell
C:\TaskWeave\venv\Scripts\taskweave.exe --version
C:\TaskWeave\venv\Scripts\taskweave.exe --help
```

**Success:** `taskweave <version>`, and the help lists `--transport {stdio,http}`.

Now run one MCP handshake over stdio by hand. Paste this into the same PowerShell window; it sends `initialize` and `tools/list` and prints what TaskWeave writes to stdout:

```powershell
$env:TASKWEAVE_WORKSPACE = "C:\TaskWeave\workspace"
$env:TASKWEAVE_JOBS_DIR = "C:\TaskWeave\jobs"
$mcpCheck = @(
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"check","version":"0"}}}',
  '{"jsonrpc":"2.0","method":"notifications/initialized"}',
  '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
)
$out = $mcpCheck | C:\TaskWeave\venv\Scripts\taskweave.exe --transport stdio
$out.Count
($out[0] | ConvertFrom-Json).result.serverInfo
($out[1] | ConvertFrom-Json).result.tools.name
Remove-Item Env:TASKWEAVE_WORKSPACE, Env:TASKWEAVE_JOBS_DIR
```

**Success:** `2` (exactly two lines on stdout), `name TaskWeave` with `version 0.1.0`, and the tool names (fifteen with the default `TASKWEAVE_TOOL_NAMES=both`, eight with `task` or `research`). TaskWeave writes **only MCP messages to stdout**; its warnings and errors go to stderr, so they appear in the console here but are not part of `$out`. If `$out` holds anything other than JSON lines, the MCP client will fail to parse it.

**If it fails:** a configuration problem prints `Invalid TaskWeave configuration:` with every problem listed (for example a missing workspace or `claude.exe` not found) and exits with code 2. Fix those first. Nothing in the later steps can work until this one does.

## Step 2: Create the launcher

The tunnel client starts exactly one command and TaskWeave reads its settings from environment variables, so a small `.cmd` file sets them in one place. Copy `examples/windows/taskweave-stdio.cmd` to `C:\TaskWeave\taskweave-stdio.cmd`. It contains:

```bat
@echo off
rem TaskWeave stdio launcher for MCP clients that start a command (such as the OpenAI tunnel client).
rem Print NOTHING to stdout in this file: stdout carries the MCP protocol.
rem Copy this file into the folder that holds your TaskWeave venv, then adapt the values below.
setlocal
set "ROOT=%~dp0"

rem The only folder Claude works in. It must already exist. Use a disposable folder for the first test.
set "TASKWEAVE_WORKSPACE=%ROOT%workspace"
rem Job folders (prompts, reports, metadata). Created if missing. Must not be inside the workspace.
set "TASKWEAVE_JOBS_DIR=%ROOT%jobs"
rem Uncomment and set when claude.exe is not on the PATH this launcher sees. Use the native claude.exe, not a .cmd shim.
rem set "TASKWEAVE_CLAUDE_BIN=C:\path\to\claude.exe"
rem Highest mode any caller may request: read_only, read_write or full_repository. Start with read_only.
set "TASKWEAVE_MAX_PERMISSION_MODE=read_only"
set "TASKWEAVE_MAX_CONCURRENT=1"
set "TASKWEAVE_JOB_TIMEOUT_SECONDS=1800"

rem Startup errors and warnings (stderr) are appended to a log file next to this launcher.
"%ROOT%venv\Scripts\taskweave.exe" --transport stdio %* 2>>"%ROOT%taskweave-stderr.log"
exit /b %ERRORLEVEL%
```

- `%~dp0` is the launcher's own folder, so the paths stay correct wherever you put `C:\TaskWeave`. Adapt the folder names and values to your setup; none of them are required defaults.
- **Keep `TASKWEAVE_MAX_PERMISSION_MODE=read_only` for the first run.** It is the most any ChatGPT user of this connector can request (see [Permission modes](reference.md#permission-modes)).
- `@echo off` and the absence of any `echo` matter: anything the launcher prints to stdout corrupts the MCP stream.
- stderr goes to `taskweave-stderr.log`, because the tunnel client may not show it anywhere you will look.
- `%*` passes extra arguments through. The tunnel client adds none; the control center's **Connect** page adds `--setup-check` to prove the launcher without touching your real jobs folder, and you can do the same by hand (Step 2's check below).
- TaskWeave reads these settings only when it starts. After you edit the launcher, restart the tunnel client.

**Alternative: a settings profile instead of variables.** If you would rather manage these values in the [control center](reference.md#control-center-and-settings-profiles), keep a settings file **next to the launcher** and start TaskWeave with `--profile` and `--config`. Remove the `set "TASKWEAVE_..."` lines when you do, because environment variables override the profile:

```bat
@echo off
setlocal
set "ROOT=%~dp0"
"%ROOT%venv\Scripts\taskweave.exe" --profile chatgpt --config "%ROOT%taskweave.toml" --transport stdio %* 2>>"%ROOT%taskweave-stderr.log"
exit /b %ERRORLEVEL%
```

Create the profile with `C:\TaskWeave\venv\Scripts\taskweave-control.exe --config C:\TaskWeave\taskweave.toml` (Settings, *New profile*, name `chatgpt`, workspace `C:\TaskWeave\workspace`, jobs folder `C:\TaskWeave\jobs`). Do not rely on the default settings file in `%LOCALAPPDATA%` here: if you start the tunnel client from the Microsoft Store version of PowerShell, TaskWeave cannot see that folder (see [Windows: a profile or settings file is not found](reference.md#windows-a-profile-or-settings-file-is-not-found)). Model, effort and cost-cap changes made in the control center then apply to ChatGPT's next job without restarting the tunnel client, and so do working folders, their aliases and descriptions, the default folder and mode, and folder routing. The workspace, jobs folder, permission cap and limits still need a restart.

Run the Step 1 check again through the launcher, exactly the way the tunnel client will start it (note the forward slashes):

```powershell
$out = $mcpCheck | cmd.exe /d /c C:/TaskWeave/taskweave-stdio.cmd
$out.Count
```

**Success:** `2`, as in Step 1. **If it prints `0`:** read `C:\TaskWeave\taskweave-stderr.log`. **If it prints more than `2`:** something in the launcher is echoing to stdout.

## Step 3: Install, configure and start the tunnel client

This step has seven parts. 3a to 3e are done once per machine; 3f and 3g are how you start and check the tunnel every time (see "Every time you start the tunnel" at the end of this step).

**3a. Get the tunnel client.** On the [openai/tunnel-client releases page](https://github.com/openai/tunnel-client/releases), open the latest release and download **`tunnel-client-v<version>-windows-amd64.zip`** (`-windows-arm64.zip` on an ARM PC). Take this full download, not the smaller `tunnel-client-runtime-...` files: it contains `tunnel-client.exe`, the `cloudflared.exe` it runs beside it, and the local admin page this guide uses. Extract the whole zip into `C:\TaskWeave\tunnel-client\` and keep the files together.

In PowerShell, store the path once per window (a quoted path must be run with the call operator `&`) and check the program starts:

```powershell
$tc = "C:\TaskWeave\tunnel-client\tunnel-client.exe"
& $tc --version
```

**Success:** a version line such as `0.0.18+...`. These executables are not code-signed, so Windows SmartScreen may warn the first time; only continue if you downloaded the zip from the official `openai/tunnel-client` releases page. To check the download, compare `(Get-FileHash <zip>).Hash` with the SHA-256 GitHub shows next to the file on the release page.

**3b. Provide the runtime API key without writing it into a file.** The profile refers to the key by environment variable name only. Set it in the **same PowerShell window** you will run the tunnel client from:

```powershell
$env:CONTROL_PLANE_API_KEY = Read-Host -MaskInput "Runtime API key"   # PowerShell 7; in Windows PowerShell 5.1 drop -MaskInput
```

This lasts only for that window: a new window needs it again. Use the runtime key, never an admin key. The tunnel client also accepts `file:C:/path/to/key.txt` instead of `env:...` if you prefer a file that only your account can read. Never put a literal key in a profile, a launcher, a script you share or a repository.

**3c. Create the profile.** A profile is a YAML file that tells the tunnel client which tunnel to serve and which command starts TaskWeave. Create one from the stdio sample with your tunnel ID:

```powershell
& $tc init --sample sample_mcp_stdio_local --profile taskweave --tunnel-id <your-tunnel-id> --mcp-command "cmd.exe /d /c C:/TaskWeave/taskweave-stdio.cmd"
```

- The name after `--profile` is yours to choose (this guide uses `taskweave`; any name works, for example `claude-bridge`). It becomes the file `%APPDATA%\tunnel-client\<name>.yaml`, and every later command refers to the profile by the same name: `--profile taskweave` reads `%APPDATA%\tunnel-client\taskweave.yaml`.
- `& $tc profiles list` shows every profile and its file path.
- `init` will not overwrite an existing profile (`profile "taskweave" already exists; pass --force to replace it`). To change a profile, edit its file, for example `notepad "$env:APPDATA\tunnel-client\taskweave.yaml"`, then rerun 3e.
- You can also write the file by hand: save the YAML below as `%APPDATA%\tunnel-client\<name>.yaml` with your own values.

The profile looks like this, with your own tunnel ID:

```yaml
config_version: 1
control_plane:
  base_url: "https://api.openai.com"
  tunnel_id: "tunnel_<your-tunnel-id>"
  api_key: "env:CONTROL_PLANE_API_KEY"
health:
  listen_addr: "127.0.0.1:8080"
admin_ui:
  open_browser: false
log:
  level: info
  format: json
mcp:
  commands:
    - channel: main
      command: "cmd.exe /d /c C:/TaskWeave/taskweave-stdio.cmd"
```

**Windows paths in `command`: use forward slashes.** The tunnel client splits `command` the way a Unix shell does, where `\` is an escape character, so `C:\TaskWeave\taskweave-stdio.cmd` reaches Windows as `C:TaskWeavetaskweave-stdio.cmd` and fails with *"is not recognized as an internal or external command"*. `C:/TaskWeave/taskweave-stdio.cmd` works. For a path with spaces, put double quotes around the path inside the command, for example `--mcp-command 'cmd.exe /d /c "C:/My Tools/TaskWeave/taskweave-stdio.cmd"'` (in PowerShell). `cmd.exe /d` skips any AutoRun commands configured for `cmd.exe`, which could otherwise print to stdout. One launcher is enough; a second wrapper `.cmd` that `call`s the first one is not needed.

The health listener `127.0.0.1:8080` belongs to the tunnel client (it has nothing to do with TaskWeave's own HTTP port, which this setup does not use). Keep it on `127.0.0.1`: do not change it to `:8080` or `0.0.0.0`, and do not enable remote UI access. If port 8080 is already in use, pick another loopback port.

**3d. Recommended: dry-run the command through the tunnel client, offline.** `tunnel-client dev proxy` runs an in-memory control plane on your PC, so it needs no key, no tunnel ID and no network. It starts TaskWeave with the same command-splitting rules as the real tunnel, and it is the only check here that proves the launcher path in your profile works:

```powershell
& $tc dev proxy --mcp-command "cmd.exe /d /c C:/TaskWeave/taskweave-stdio.cmd" --print-json --duration 60s
```

It prints JSON with an `mcp_url`. In a second PowerShell window:

```powershell
'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"check","version":"0"}}}' | Set-Content -NoNewline init.json
curl.exe -s -X POST "<mcp_url from the JSON>" -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" --data-binary "@init.json"
```

**Success:** a JSON reply containing `"serverInfo":{"name":"TaskWeave","version":"0.1.0"}`. A `502 Bad Gateway`, or a log line saying `stdio MCP command failed`, means the command could not start: check the slashes, the launcher path and `taskweave-stderr.log`.

**3e. Validate the profile** (before starting the tunnel, in the window where you set the key):

```powershell
& $tc doctor --profile taskweave --explain
```

**Success:** a list of `CHECK ... PASS` lines (some `SKIP` lines are normal for a stdio command), then `RESULT ok` and `NEXT   tunnel-client run --profile taskweave`. A failure ends in `RESULT fail` and a `FAILED_CHECKS` line naming the checks to fix. `--explain` says why a failed check matters and what to change. What `doctor` does **not** prove: it checks that the key variable is set, not that the key is valid (a wrong key fails only when `run` connects), and it checks that `cmd.exe` exists, not that your launcher file exists (3d checks that).

| `doctor` says | Meaning and fix |
|---|---|
| `control_plane_api_key FAIL ... environment variable "CONTROL_PLANE_API_KEY" is not set` | The key is not set in **this** window. Do 3b here. |
| `health_listener FAIL ... bind: Only one usage of each socket address` | Port 8080 is taken. Usually the tunnel client is already running (check with 3g); otherwise another program uses the port: change `listen_addr` in the profile to another `127.0.0.1` port and use that port in 3g. |
| `profile_load FAIL ... cannot find the file specified` | No file `%APPDATA%\tunnel-client\<name>.yaml` for the name you passed. Check the name with `& $tc profiles list`. |
| `invalid tunnel ID` (from `init` or `doctor`) | A tunnel ID is `tunnel_` followed by 32 lowercase letters or digits, sometimes with a 4-character namespace in between (`tunnel_<namespace>_<32 characters>`). Copy it again from Tunnels management. |

**3f. Start the tunnel client.** In the window where you set the key (3b):

```powershell
& $tc run --profile taskweave
```

`run` stays in the foreground and prints log lines; leave this window open for as long as ChatGPT should reach TaskWeave. Closing it, or pressing Ctrl+C, stops the tunnel and the TaskWeave process it started. The tunnel client starts TaskWeave itself: do not start `taskweave` separately.

**3g. Check that it is running and ready.** From a **second** PowerShell window:

```powershell
curl.exe -i http://127.0.0.1:8080/readyz
```

**Success** looks like this (the date differs):

```text
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Date: ...
Content-Length: 5

ready
```

Type `curl.exe`, not `curl`: in Windows PowerShell 5.1, `curl` is a different command (`Invoke-WebRequest`) with different options.

| `/readyz` result | Meaning and fix |
|---|---|
| `HTTP/1.1 200 OK` and `ready` | The tunnel client is running and its startup checks have passed. Go on to Step 4. |
| `curl: (7) Failed to connect to 127.0.0.1:8080 ... Could not connect to server` | Nothing is listening: the tunnel client is not running, or the profile uses another port. Look at the `run` window. If it stopped by itself, read `C:\TaskWeave\taskweave-stderr.log` (see below). |
| Any other status, with a short reason in the body | It is running but not ready yet. The reason says which startup check is pending or failed; wait a few seconds and retry, then look at the `run` window. |

More detail, from the same second window: `curl.exe -i http://127.0.0.1:8080/healthz` (`200` and `live` only means the process is up), `curl.exe -s "http://127.0.0.1:8080/health?details=true"` (MCP, polling and queue status), and `& $tc health --port 8080`. The local admin page `http://127.0.0.1:8080/ui` shows status, channel routing and logs. With a stdio command the tunnel can report ready before ChatGPT has made its first call; that is normal.

**If it is not ready:** look at the `run` window and the admin page, export the recent (redacted) logs with `curl.exe -fsSJO "http://127.0.0.1:8080/api/logs/export?minutes=30"`, and read `C:\TaskWeave\taskweave-stderr.log`. **If TaskWeave exits, the tunnel client shuts itself down** (0.0.17 and 0.0.18; from 0.0.19 it restarts the process instead, so a broken launcher can loop quietly: set `MCP_STDIO_FAILURE_POLICY=exit` in the window that runs the client to get the shutdown back. On the older versions its log says `stdio MCP command failed; requesting tunnel-client shutdown`); the reason is in `taskweave-stderr.log`, usually a configuration error from Step 1.

**Every time you start the tunnel** (after a reboot, or after closing the window), repeat only 3b, 3f and 3g:

```powershell
# Window 1: start the tunnel client and leave this window open
$tc = "C:\TaskWeave\tunnel-client\tunnel-client.exe"
$env:CONTROL_PLANE_API_KEY = Read-Host -MaskInput "Runtime API key"
& $tc run --profile taskweave

# Window 2: check it is ready (expect HTTP/1.1 200 OK and "ready")
curl.exe -i http://127.0.0.1:8080/readyz
```

The ChatGPT connector (Step 4) is created once and reconnects to the tunnel by itself whenever the tunnel client is running.

## Step 4: Create the ChatGPT connector

Do this while `tunnel-client run` is running and `/readyz` returns 200. Follow the current [official steps](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels); at the time of writing they are:

1. Open ChatGPT's connector settings (`https://chatgpt.com/#settings/Connectors`; the section may be named Connectors, Apps or Plugins).
2. Add a custom MCP server (connector).
3. Under **Connection**, choose **Tunnel**, then select your tunnel from the list or paste its tunnel ID.
4. TaskWeave over stdio does not use OAuth, so no MCP authentication needs to be configured for it.
5. Read the risk warning, confirm it, and create the connector.

**Success:** the connector shows the TaskWeave tools (`start_task`, `check_task`, `get_task_result`, `cancel_task`, `list_tasks`, `list_task_artifacts`, `get_task_artifact`, `list_providers`, plus the older names such as `start_research_job`, which keep earlier setups working).

The tunnel ID identifies your tunnel. It is not a credential: using the tunnel needs the runtime key and organization permissions. Still, treat it as private configuration and do not publish it. The runtime API key **is** a secret.

| Problem | What to check |
|---|---|
| No option to add a custom MCP server | Your plan or workspace does not allow custom MCP connectors, or an admin has not enabled them. Ask your workspace admin; see the official guide for which plans support it. |
| Your tunnel is not in the list | The tunnel is associated with the wrong ChatGPT workspace (a tunnel tied only to a personal Platform organization does not appear in a business workspace; add the workspace in Platform tunnel settings). You lack *Tunnels Use*. You selected a different Platform organization. Role changes can take up to 30 minutes. |
| "Tunnels access required" | Confirm the Platform organization, then ask an owner or RBAC administrator for the role. |
| Discovery fails, or no tools appear | `tunnel-client run` has stopped, or `/readyz` is not 200. Rerun `doctor --profile taskweave --explain`. Check `taskweave-stderr.log`. |
| Old or missing tools after an upgrade | Refresh or reconnect the connector in ChatGPT so it reloads the tool list. |

## Step 5: Run a safe end-to-end test

Ask ChatGPT, in plain words, to use the TaskWeave tools. For example:

1. **Tools are there.** "List the TaskWeave tools you can use." All of them appear.
2. **Start a read-only job.** "Use TaskWeave `start_task` with the prompt: *Read readme.txt in the workspace and summarize it in one sentence. Do not modify anything.*" It returns a `job_id` within seconds, with `permission_mode: read_only`.
3. **Poll.** "Check that job until it is no longer queued or running." It ends `completed`. Each call returns quickly; the job runs in the background.
4. **Get the result.** "Get the result of that job." Expect `status: completed`, `return_code: 0`, a report that summarizes the file, `report_persisted: true` and `artifacts_finalized: true`. On disk, `C:\TaskWeave\jobs\<job_id>\` holds `metadata.json`, `prompt.txt`, `stdout.txt`, `stderr.txt`, `report.txt` and `artifacts.json`.
5. **Cancel.** Start another read-only job with a longer task (for example "*Read every file in the workspace and write a detailed report*"), then immediately ask ChatGPT to cancel it. Expect `status: cancelled`, `reason: cancelled_by_user`, `termination_pending: false`, and a persisted report. If the job finished before the cancel arrived, it stays `completed`; that is not an error.
6. **Write access (optional, test instance only).** Only on a disposable instance, and only if you deliberately want to test writes: set `TASKWEAVE_MAX_PERMISSION_MODE=read_write` in the launcher and restart the tunnel client. Ask for a job with `permission_mode: "read_write"` that creates `taskweave-write-smoke-test.txt` containing one known line, then a `read_only` job that reads it back. Afterwards set the ceiling back to `read_only` and restart the tunnel client again.
7. **What the modes mean.** A job asks for a mode (`permission_mode`, default `read_only`); the operator's launcher sets the **ceiling** (`TASKWEAVE_MAX_PERMISSION_MODE`). A request above the ceiling is rejected before any job is created, with `permission_mode 'read_write' is not permitted on this server (maximum: 'read_only')`. That is the control working, not a fault. Raising the ceiling lets **every** user of the connector request that mode. `read_write` means Claude may create and edit files inside the workspace. It does not mean shell access, deleting, renaming or moving files, or access outside the workspace, and it is not an OS sandbox (see the Security section at the top of this README).
8. **A follow-up in the same thread.** In the same chat, ask a follow-up and ask ChatGPT to pass the `thread_id` from the first job. The new job returns the same `thread_id` with `turn: 2`, and `taskweave-viewer` (or the control center's Jobs section) shows both as one conversation. Claude does not remember the first turn, so the follow-up prompt must repeat any context it needs.
9. **A file back.** "Use TaskWeave to read readme.txt and return its lines as a CSV file with columns `line_number,text`." The result's report contains `[Artifact saved: ....csv ...]`, `artifacts` lists the file, and ChatGPT can fetch it with `get_task_artifact`. Claude returns the file inside its report, so this works in `read_only`.
10. **Model and effort (optional).** Ask for a job with `model: "claude-sonnet-5-5"` and `effort: "low"`. The result and `metadata.json` record both. If the server restricts models or effort (`TASKWEAVE_MODELS`, `TASKWEAVE_EFFORTS`, or the profile's lists), a value outside them is rejected before a job is created; `list_providers` shows what is allowed under `job_choices`.
11. **A file you upload (optional).** Drop a small PDF into the ChatGPT chat and say "*Use TaskWeave to summarize this PDF; I will upload it in TaskWeave.*" Expect `start_task` with `wait_for_files: true`, status `waiting_for_files`, and the control center opening at the job (or a banner in a control center page that is already open). Upload the same PDF there with **Upload files**; the job starts at once, and the job folder holds `prompt-artifacts/01.pdf`. Not yet verified through a real tunnel.

**Keep one server process for a job's whole life.** The tunnel client starts TaskWeave once and keeps it running, so polls from later ChatGPT messages reach the same server. If you script your own tests, keep one MCP session open for start, polling and result. A test that starts a fresh server for every call can kill the job it just started: the MCP Python SDK's stdio client, for example, ends the server's whole process tree on Windows when its session closes. The next server then correctly reports that job as `failed` with reason `server_restarted`. Restarting the tunnel client itself does not do this: in testing, a job that was running when the tunnel client stopped kept running and was re-attached by the next TaskWeave process.

## Troubleshooting by layer (Windows and ChatGPT)

Work from the top down and do not move on until a layer works.

| Layer | How to check | Success looks like | If it fails |
|---|---|---|---|
| 1. Claude CLI and sign-in | `claude -p "Reply with exactly BRIDGE_JOB_OK. Do not use tools."` and `where claude` | `BRIDGE_JOB_OK`; `where` lists a `claude.exe` | Sign in again, or install the native CLI. Set `TASKWEAVE_CLAUDE_BIN` to `claude.exe` if only a `.cmd` shim is on `PATH`. |
| 2. TaskWeave MCP startup | Step 1 handshake | 2 JSON lines, `TaskWeave` `0.1.0`, the TaskWeave tools | Fix every item under `Invalid TaskWeave configuration:` (exit code 2). |
| 3. Launcher and stdout/stderr | Step 2 handshake through `cmd.exe /d /c C:/TaskWeave/taskweave-stdio.cmd` | The same 2 lines | `0` lines: read `taskweave-stderr.log`. More lines: something echoes to stdout. |
| 4. Tunnel: profile, key, health, logs | `doctor --profile taskweave --explain`, then `run`, `/healthz`, `/readyz`, `/ui` | All checks pass; both endpoints return 200 | Key variable unset in this window, wrong tunnel ID, missing *Tunnels Read + Use*, backslashes in `command`, port 8080 in use. If the tunnel client stops by itself (`stdio MCP command exited ... exit status 2`), TaskWeave could not load its settings: read `taskweave-stderr.log`, or run the launcher by hand in the same window (`$null \| cmd /d /c C:/TaskWeave/taskweave-stdio.cmd`) to see the message. `Profile '...' was not found ... (profiles: none)`: see [Windows: a profile or settings file is not found](reference.md#windows-a-profile-or-settings-file-is-not-found). |
| 5. ChatGPT connector | Step 4 | Connector created; tools listed | Plan or workspace permissions, tunnel's workspace association, role propagation (up to 30 minutes), tunnel client not running. |
| 6. Job lifecycle | Step 5; `C:\TaskWeave\jobs\<job_id>\metadata.json` | `completed`, `return_code: 0`, `report_persisted: true` | `get_task_result` works after any failure: read `reason`, `error` and `stderr`. `permission_mode ... is not permitted`: ceiling (see Step 5, item 7). `failed_to_start` / `cli_missing_required_flags`: layer 1. `timed_out`: raise `TASKWEAVE_JOB_TIMEOUT_SECONDS` or narrow the task. `server_restarted`: the TaskWeave process and the job's process both stopped while the job was running (see "Keep one server process"). |

The general [Troubleshooting](reference.md#troubleshooting) and [Job lifecycle and restarts](reference.md#job-lifecycle-and-restarts) sections apply as well.

**What has and has not been verified.** On Windows 11 with tunnel-client 0.0.17: the forward-slash and quoting rules, the launcher, and the offline `dev proxy` round trip, including a job's completion, cancellation, permission-ceiling rejection and re-attachment after the tunnel client restarted; and, separately, live ChatGPT sessions through a real tunnel (Steps 4 and 5, including the optional write test), the last one with the released 0.1.0 wheel. With tunnel-client 0.0.18, offline only: `init`, `profiles list`, `doctor` (the messages quoted above), `/readyz`, the forward-slash and backslash behavior, and a job's completion, cancellation and ceiling rejection through `dev proxy`. With tunnel-client 0.0.19, offline only: `init --profile-dir`, `profiles list --json`, `doctor`, the profile layout and the exit policy through `dev proxy` (see the note at the start of this guide). Not verified by these tests: `/readyz` and `run` on 0.0.19, the start script's key prompt with a real key, a clean-machine install by a new user, the tunnel client on macOS or Linux, timeouts and server-restart recovery through a real tunnel, and any ChatGPT plan or workspace configuration other than the one used.
