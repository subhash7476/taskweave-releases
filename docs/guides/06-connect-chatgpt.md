# Connect ChatGPT

Verified with TaskWeave 0.2.0rc1, tunnel-client 0.0.19, on 2026-11-03

## Goal and time

Connect ChatGPT to TaskWeave through the OpenAI tunnel client. About 30 minutes after [Prepare ChatGPT on OpenAI](05-prepare-chatgpt-on-openai.md), plus the time the connector takes to appear.

## You need

- A saved TaskWeave profile and a working `taskweave doctor`, as in [First-run setup](03-first-run-setup.md).
- The tunnel ID and runtime key from [Prepare ChatGPT on OpenAI](05-prepare-chatgpt-on-openai.md).
- The tunnel client executable from the official OpenAI release.
- The control center open with the **Connect** page. Run `taskweave-control` if needed.
- A disposable workspace for the first job.

## Steps

### Sub-step 1: Make and prove the launcher

1. Open the control center's **Connect** page.
2. Choose the saved profile the launcher should use.
3. Press **Write the launcher**. The page writes `taskweave-stdio.cmd` in a folder whose path you can see; a path with a space is fine.
4. Press **Prove the launcher**. It must print two JSON replies and list all the TaskWeave tools.
5. If the proof fails, read the message before changing anything else; a launcher that prints extra text breaks the tunnel.

### Sub-step 2: Download and check the tunnel client

1. Download the official tunnel client for Windows from the OpenAI tunnel-client repository. Do not use a mirror.
2. Put it in a folder of its own, for example next to the launcher.
3. Press **Offline check** on the Connect page and point it at the executable. It reads the version and checks the command it will use.
4. If Windows warns about the download, choose **More info**, then **Run anyway** for the file from the official OpenAI release.
5. The page shows the version. If it is newer than the versions this guide was checked against, the page says so; the manual steps still work.

```powershell
tunnel-client --version
```

### Sub-step 3: Create the tunnel profile

1. In Connect, create the profile with the tunnel ID. The page passes a profile folder, so the client never writes into a shared default folder.
2. Use a profile name you will recognize, such as `taskweave`.
3. If the page shows the generated command, it is the same one it used. A typical form is:

   ```powershell
   tunnel-client init --profile-dir <profile folder> --sample sample_mcp_stdio_local --profile taskweave --tunnel-id <tunnel id> --mcp-command "<launcher path>" --health-listen-addr 127.0.0.1:8080
   ```

4. Confirm the profile exists and lists a health address on loopback:

   ```powershell
   tunnel-client profiles list --profile-dir <profile folder> --json
   ```

5. Run the client's own readiness check:

   ```powershell
   tunnel-client doctor --profile taskweave --profile-dir <profile folder> --explain
   ```

### Sub-step 4: Write and start the start script

1. Press **Write the start script** in Connect. It creates `taskweave-tunnel.cmd` and a PowerShell helper next to the launcher.
2. Double-click `taskweave-tunnel.cmd`. It asks for the runtime key in its own window; the prompt is masked.
3. Type the runtime key from your password manager. The script never writes it to a file, and TaskWeave never stores it.
4. Leave the window open. The script starts the tunnel client and TaskWeave together over stdio.
5. If the key is wrong, the client fails when it starts, not before; that is expected.

### Sub-step 5: Check readiness and the first job

1. In a second window, check the local readiness address:

   ```powershell
   curl.exe -i http://127.0.0.1:8080/readyz
   ```

2. You should see `HTTP/1.1 200 OK` and the body `ready`.
3. In Connect, press **Watch for a job**.
4. Send a first prompt from ChatGPT only after the readiness light is green.

### Sub-step 6: Create the ChatGPT connector and run the first job

1. In ChatGPT, open the connector settings and add a custom MCP server (connector).
2. Under **Connection**, choose **Tunnel** and select your tunnel.
3. TaskWeave over stdio does not use OAuth, so no MCP authentication is needed for it.
4. Read the risk warning, confirm it, and create the connector.
5. The connector should list the TaskWeave tools: `start_task`, `check_task`, `get_task_result`, `cancel_task`, `list_tasks`, `list_task_artifacts`, `get_task_artifact` and `list_providers` (plus the older names such as `start_research_job`, which keep earlier setups working).
6. Ask ChatGPT: "Use TaskWeave `start_task` to read README.txt in the workspace and summarize it in one sentence. Do not modify anything."
7. Ask it to check the job until it is no longer queued or running. It should end `completed` with a short report.
8. If the tools do not appear, refresh the connector or reconnect it so ChatGPT reloads the tool list.

## You should see

- The launcher proof prints two JSON replies and all the tools.
- `tunnel-client --version` prints a version line.
- The profile lists a `127.0.0.1` health address.
- `doctor` prints a successful result.
- `/readyz` returns 200 with `ready`.
- The connector lists the TaskWeave tools.
- The first read-only job ends `completed` with a report.

## If it does not match

| What you see | What to do |
|---|---|
| The launcher proof fails | Run it again and read the message. Extra output on stdout is the usual cause; the launcher must print only the protocol. |
| The tunnel client is newer than the checked versions | Continue; the page says which version was checked. If a command fails, use the manual steps in [the ChatGPT walkthrough](../chatgpt-tunnel.md). |
| No option to add a custom MCP server | Your plan or workspace does not allow it, or an administrator has not enabled it. See [Prepare ChatGPT on OpenAI](05-prepare-chatgpt-on-openai.md). |
| Your tunnel is not in the list | The tunnel is associated with another workspace or organization, or the role has not taken effect. Wait up to 30 minutes, then check again. |
| `Tunnels access required` | Ask for **Tunnels Read + Use**; confirm the Platform organization. |
| Discovery fails or no tools appear | The tunnel client or TaskWeave stopped, or `/readyz` is not 200. Run `tunnel-client doctor --profile taskweave --profile-dir <profile folder> --explain`, check `taskweave-stderr.log`, then start the script again. |
| The client stops when TaskWeave starts | Read `taskweave-stderr.log`; the TaskWeave profile or settings file could not load. The generated script sets the failure policy to exit so the client does not hide the problem. |
| The key prompt rejects the key | Use the runtime key, not an admin key. The client is meant to fail when it starts, not before. |

## Done when

`/readyz` returns 200, ChatGPT lists the TaskWeave tools, and one read-only job finishes with a report.

## Next

[Your first real task](07-your-first-real-task.md)
