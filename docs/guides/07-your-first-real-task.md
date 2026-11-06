# Your first real task

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Send one read-only job from Claude Desktop, watch it finish, and find its report and files in the control center. About 10 minutes.

## You need

- Claude Desktop connected to TaskWeave, as in [Use TaskWeave in Claude Desktop](04-use-taskweave-in-claude-desktop.md).
- A workspace with a small file in it, for example a `README.txt`.
- The control center open, or the command `taskweave-control` ready.
- If you also connected ChatGPT, the same task works there; use the ChatGPT section at the end.

## Steps

1. Open the control center's **Jobs** view and leave it open in a second window. Run `taskweave-control` if it is not open.

2. In Claude Desktop, start a new chat and ask, in plain words: "Use TaskWeave to read README.txt in the workspace and summarize it in one sentence. Do not modify anything."

3. Claude Desktop should call `start_task` and return a job ID within a few seconds. The job runs in the background; the chat does not need to stay open.

4. Ask Claude Desktop to check the job until it is no longer queued or running. The status should become `completed`.

5. Ask for the result. The report should summarize the file. Look for `status: completed`, `return_code: 0`, and a report message.

6. In the control center's Jobs view, find the job. Open it and read the prompt, status, report and any files. The same information is on disk under `<jobs folder>/<job id>/`.

7. Open the folder and look at the files: `metadata.json`, `prompt.txt`, `stdout.txt`, `stderr.txt`, `report.txt` and `artifacts.json`. Files the job returned are in a `reply-artifacts` folder (`01.csv`, `02.md`, ...), and files sent with the prompt are in `prompt-artifacts`. A read-only job can write its report and artifacts; it cannot change the workspace.

8. Optional: open the read-only session viewer with the jobs folder:

   ```powershell
   taskweave-viewer --jobs-dir <jobs folder>
   ```

9. If you connected ChatGPT, repeat the same task there: ask ChatGPT to use `start_task` with the same prompt, then check the job and get the result. The job appears in the same Jobs view.

## You should see

- A job ID within a few seconds.
- The job moves from `queued` or `running` to `completed`.
- A short report that mentions the file you asked about.
- The job in the Jobs view, with the prompt and report.
- A job folder on disk with the files listed above.
- In ChatGPT, the same tools and the same result shape.

## If it does not match

| What you see | What to do |
|---|---|
| No job ID | Claude Desktop may not have the TaskWeave tools. Restart Claude Desktop completely and check the tools again. |
| The job stays `queued` | Another job may be running, or the server is not running. Check the control center's Health page. |
| The job fails with a sign-in message | Run `claude` once, finish the sign-in, and start a new job. |
| The job fails with a permission message | The requested mode is above the operator's ceiling. A new job in `read_only` is the safe choice. |
| No report | Read `stderr.txt` and the job metadata; a timed-out or cancelled job may have partial output only. |
| The viewer shows nothing | Pass the same jobs folder the profile uses; the default may be different. |
| ChatGPT has no tools | The tunnel or connector is down; return to [Connect ChatGPT](06-connect-chatgpt.md). |

## Done when

One read-only job finishes `completed`, its report is readable, and you can find its folder and files.

## Next

[Get real work done](08-get-real-work-done.md)
