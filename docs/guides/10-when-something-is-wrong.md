# When something is wrong

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Find the failing layer and collect the right evidence for a GitHub issue. About five minutes to narrow the problem; longer if it is a tunnel or permission problem.

## You need

- The guide step that failed.
- The control center open, or `taskweave-control` ready.
- A GitHub account, to open an issue on the [TaskWeave releases repository](https://github.com/subhash7476/taskweave-releases/issues).
- A way to remove secrets and personal paths from what you paste.

## Steps

1. Work one layer at a time, in this order: **Claude's coding agent**, **the TaskWeave server**, **the transport or tunnel**, **client discovery**, **the job lifecycle**. Do not move to the next layer until the current one works.

2. **Coding agent layer.** Run the one-line check:

   ```text
   claude -p "Reply with exactly BRIDGE_JOB_OK. Do not use tools."
   ```

   It must print `BRIDGE_JOB_OK`. If it asks you to sign in, run `claude` once and finish the sign-in; if it says Claude Code's `claude.exe` is missing behind `claude.cmd`, reinstall Claude Code (guide 01).

3. **Server layer.** Check the installed command and the settings:

   ```powershell
   taskweave --version
   taskweave doctor --json
   ```

   Read the `fail` items first. A profile or settings-file error is usually a wrong path or a missing profile.

4. **Transport or tunnel layer.** For ChatGPT, check `/readyz` and the tunnel client's own doctor command. For an HTTP client, check the token and the loopback address. The tunnel client's logs and `taskweave-stderr.log` are the first places to look.

5. **Client discovery layer.** Refresh or reconnect the connector in ChatGPT, or restart Claude Desktop completely. A client can cache the old tool list after an upgrade.

6. **Job lifecycle layer.** Open the job in the control center and read `metadata.json`, `stderr.txt` and the report. A timed-out or cancelled job may have partial output by design.

7. **Collect the evidence.** In the control center, open **Health** and press **Copy diagnostics**; or run:

   ```powershell
   taskweave doctor --support
   ```

   Both make a support report: the versions, every check and its hint, with secrets, key-shaped text, paths in your home folder, your user name, e-mail addresses and tunnel IDs removed. Also collect the last part of `taskweave-stderr.log`, the guide step that failed, and the job ID if there is one. Do not collect the whole jobs folder unless asked.

8. **Remove secrets before you post.** Remove runtime keys, tokens, tunnel IDs, personal paths, email addresses and any file contents you do not want public. Replace them with `<redacted>`. The support report from **Copy diagnostics** or `taskweave doctor --support` has those removed already, but read it before you post, and check anything you copied by hand.

9. **Open a GitHub issue** at <https://github.com/subhash7476/taskweave-releases/issues>. Say which guide and step failed, what you expected, what you saw, the four versions (TaskWeave, tunnel client, Claude Code, operating system), and the evidence with secrets removed. Do not attach a runtime key or a config file that contains other servers' values.

10. If the problem is a bad release, do not edit the published files. Report it, and use the previous release until a new one is published.

## You should see

- One layer identified as the first failure.
- `doctor` output with the failing item named.
- A short, redacted evidence set.
- A GitHub issue that another person can reproduce.

## If it does not match

| What you see | What to do |
|---|---|
| `claude` is not found | Install Claude Code and open a new terminal. |
| `doctor` says the profile is missing | Use the profile name in the settings file, or create it in the control center. |
| `doctor` passes but the client has no tools | Restart the client completely; refresh the connector in ChatGPT. |
| `/readyz` is not 200 | The tunnel client or TaskWeave is not running, or the key is wrong. Start the script and read `taskweave-stderr.log`. |
| The tunnel is in the wrong workspace | Associate it with the ChatGPT workspace in the Platform tunnel settings; roles can take up to 30 minutes. |
| A job failed with `server_restarted` | The server stopped while the job ran. Start a new job; return the same thread ID if you want it grouped. |
| A job stays `waiting_for_files` | It is waiting for you to upload its files in the control center. Open `taskweave-control` on the computer that runs TaskWeave and use the job's banner, or cancel the job. See [Get real work done](08-get-real-work-done.md), step 8. |
| You cannot open an issue | Sign in to GitHub (a free account is enough). Issues are open to everyone on the public releases repository, so remove secrets first. |

## Done when

You know which layer failed, you have redacted evidence, and the issue explains how to reproduce it.

## Next

[Start here](00-start-here.md), or the guide for the layer that failed.
