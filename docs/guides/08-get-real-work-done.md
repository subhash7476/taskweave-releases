# Get real work done

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Learn the habits that make a TaskWeave job useful: choosing a permission mode, writing a prompt that ends in a real result, following a job, and getting files back. About 15 minutes to read and try the examples.

## You need

- A working Claude Desktop or ChatGPT connection, from [Your first real task](07-your-first-real-task.md).
- A disposable workspace, ideally a small git repository you can throw away.
- The control center open, or `taskweave-control` ready.

## Steps

1. Choose the starting mode. `read_only` is the default and the right first choice. It can read files, list them and search them, and write a report. It cannot create or edit files or run commands.

2. Read the full mode table before raising the limit: [Permission modes](../reference.md#permission-modes). The short version:

   | Mode | Use it when | It still cannot |
   |---|---|---|
   | `read_only` | You want analysis, a summary, a review or a report. | Create or edit files, run commands, use git or the network. |
   | `read_write` | You want a small edit in a disposable workspace. | Run commands, use git, delete or move files, or write outside the workspace. |
   | `full_repository` | You want tests or local git in a disposable repository. | Push, fetch, install packages, use the network, or run commands outside a fixed allow-list. |

3. Keep the operator's ceiling low. The launcher or profile sets the maximum mode (`TASKWEAVE_MAX_PERMISSION_MODE`); a request above it is rejected before a job starts. Raising the ceiling lets every user of the connector request that mode. Do not raise it just to make one task pass.

4. Write a prompt that states the mode, the workspace, the exact result and the stopping point. Bad: "Look at the code." Good: "In the workspace, read `src/app.py` and `tests/test_app.py`. In read-only mode, list three concrete risks and quote the lines that show each one. Do not change anything."

5. Start the job and note the job ID and thread ID. Ask the client to check the job until it is no longer `queued` or `running`. A job has a timeout; when it times out, the process tree is stopped and the partial output is kept.

6. Use threads for follow-ups. Pass the same `thread_id` from the first job; every reply's `follow_up` names the `thread_id` to pass (TaskWeave 0.3.0rc3 or newer). Each job still starts a fresh Claude session, so repeat any context the follow-up needs. A follow-up stays in its thread's folder; for another folder, start a new thread. See [Threads](../reference.md#threads).

   To work in one of the operator's working folders, put `#` and the folder's name in your request, for example: "Start a read-only TaskWeave task: #nifty list the files in this folder." The tag picks the folder even when the chat client passes only the prompt; writing the folder's full path works too. The folder's name on its own, without `#`, does not. To have ChatGPT do this by itself, open Settings in the control center, copy the text under **Copy ChatGPT instructions**, and paste it into ChatGPT's custom instructions or your project's instructions. A folder you add in Settings is usable from the next task, without restarting the tunnel. If you keep many repositories in one folder, add that folder once under **Folder groups**: each repository in it becomes a working folder named after it, including ones you clone later. Every response says which `folder` the job runs in and why (`folder_source`), and its `warnings` say when the prompt names a path the job cannot reach. Tags, paths in the prompt, folder groups and changes without a restart need TaskWeave 0.3.0rc2 or newer; after updating, refresh the TaskWeave connector in ChatGPT so it sees the new tool descriptions. See [Working folders](../reference.md#working-folders).

7. Ask for files back when a report is not enough. A job can return a CSV, a patch or another small file as an artifact. TaskWeave saves them in the job's `reply-artifacts` folder, numbered in order (`01.csv`, `02.md`, ...); the listing keeps the name Claude gave each file. The control center and `list_task_artifacts` show them; `get_task_artifact` fetches the content. See [Artifacts and attachments](../reference.md#artifacts-and-attachments).

8. Send files with a read-only job. A chat client cannot pass on a file you dropped into the chat, so ask it to start the task and wait for your files (for example: "Start a read-only TaskWeave task that waits for me to upload the files, with this prompt: ..."). The job gets its ID at once with status `waiting_for_files`, and TaskWeave opens the control center at that job on the computer that runs it. Click **Upload files**, choose or drop the files, and press **Upload and start**. The job starts as soon as the files are checked; they are saved in its `prompt-artifacts` folder as `01.pdf`, `02.png`, and so on. You have 30 minutes by default (`TASKWEAVE_UPLOAD_WAIT_SECONDS`), and a job takes one upload. Allowed types are text, Markdown, CSV, PDF, PNG, JPEG, WebP and GIF. This needs TaskWeave 0.3.0rc1 or newer, and it has not yet been checked with a real ChatGPT connection. See [Files you upload in the control center](../reference.md#files-you-upload-in-the-control-center).

9. Watch cost, model and effort. The profile can limit models and effort, and the test task is capped. A job's metadata records the model and effort it used. Start with the default; change one thing at a time.

10. Use the control center's Jobs view or the read-only viewer for a long job:

   ```powershell
   taskweave-viewer --jobs-dir <jobs folder>
   ```

   The viewer does not run jobs; it only reads the jobs folder.

11. For an HTTP client other than Claude Desktop or ChatGPT, start TaskWeave over HTTP and put your own tunnel or reverse proxy in front of it:

    ```powershell
    taskweave --transport http --port 8080
    ```

    Keep it on loopback and read [Use remotely (HTTP behind a tunnel)](../reference.md#use-remotely-http-behind-a-tunnel) before exposing it.

12. Practise the five examples below in a throwaway workspace. Start with the read-only ones; only use a write mode when you can delete the workspace afterwards.

#### Example 1: Summarize a repository

- **Mode:** `read_only`
- **Prompt:** "Read the top-level README, the package metadata and the files under `src/`. Write a one-page summary: what the project does, how to run it, and the three most important files. Quote file names. Do not change anything."
- **Expect:** a report with a short architecture summary and no workspace changes.

#### Example 2: A read-only code review

- **Mode:** `read_only`
- **Prompt:** "Review `src/app.py` for correctness and safety. List each finding with the line, why it matters, and a concrete fix. Do not edit files and do not run commands."
- **Expect:** a list of findings; the agent cannot change the code, so you decide what to apply.

#### Example 3: A small edit on a throwaway branch

- **Mode:** `read_write`
- **Prompt:** "In this disposable workspace, add a `--verbose` flag to `src/app.py`, update the usage text, and explain exactly what you changed. Do not delete or rename files and do not run commands."
- **Expect:** an edited file and a report. Review the diff yourself before using it anywhere real.

#### Example 4: Run the tests in a disposable repository

- **Mode:** `full_repository`
- **Prompt:** "Run the project's test command, report the exact command and the result, and fix only failures caused by the code in this workspace. Do not push, fetch or install anything."
- **Expect:** the allow-listed test command runs and the report names the result. `full_repository` is best effort, not a sandbox; use a disposable repository.

#### Example 5: A research report with a CSV

- **Mode:** `read_only`
- **Prompt:** "Read the notes in `research/`, then return a CSV with columns `source,claim,confidence` and a short report explaining the rows. Save the CSV as an artifact."
- **Expect:** a report and a CSV artifact. The job can read the notes and write its report without changing them.

## You should see

- Jobs that end `completed` with a report you can act on.
- No workspace changes from `read_only` jobs.
- A clear rejection when a requested mode is above the ceiling.
- Follow-ups grouped under the same thread.
- Artifacts listed in the control center and fetchable by the client.
- A job that waits for files starts as soon as you upload them in the control center.
- A job that times out stops and keeps its partial output.

## If it does not match

| What you see | What to do |
|---|---|
| `permission_mode ... is not permitted` | The request is above the operator's ceiling. Ask for a lower mode; do not raise the ceiling for one task. |
| The job changes files in a read-only task | Stop and report it. `read_only` should not write to the workspace; the report is the only output. |
| The job runs too long | Give a smaller prompt or a narrower folder; the timeout is a limit, not a target. |
| A follow-up has no context | Repeat the context; each job starts a fresh Claude session. Pass the `thread_id` to group the turns. |
| The report says a path is "outside" the working directory | The job ran in another folder (the response's `folder` says which). Put `#name` for the working folder in the request, or ask for it by name. A follow-up stays in its thread's folder, so start a new thread for another folder. If nothing works, refresh the connector so it sees the current tools, or make that folder the operator's *Default working folder*. |
| `thread_folder_mismatch` | A thread stays in one folder. Start a new thread (leave out `thread_id`) for the other folder. |
| `thread_conversation_mismatch` | With *Group each ChatGPT chat's threads* on, the thread belongs to another ChatGPT chat. Continue it from the chat that started it, or start a new thread (leave out `thread_id`). |
| `folder_ambiguous` or `folder_conflict` | The request named two folders (two `#` tags, or paths in two folders), or a tag that disagrees with the folder the client chose. A job runs in one folder: name only one. |
| An artifact is missing | Ask for a small file and name the format; very large files are refused or returned as a reference. |
| The upload page does not open | Start `taskweave-control` yourself; every control center page shows a banner for each job waiting for files. TaskWeave does not open a page when one is already open, inside a container, in a Linux session without a display, or with `TASKWEAVE_OPEN_UPLOAD_PAGE=false`. |
| The upload is refused | Check the file type and the limits shown in the dialog (10 files, 10 MiB each, 25 MiB in all by default). A job takes one upload; for more files, start a new task. |
| A job ended with `upload_timeout` | No files arrived in time. Start the task again and upload sooner, or raise `TASKWEAVE_UPLOAD_WAIT_SECONDS`. |
| Waiting for files is rejected | It works only for `read_only` jobs and not together with `attachments`. Ask for a read-only task. |
| HTTP clients cannot connect | Check the token, the loopback address and the tunnel; see [Troubleshooting](../reference.md#troubleshooting). |
| You are unsure a mode is safe | Use `read_only` and ask for a plan first. A plan is cheaper than a bad edit. |

## Done when

You can choose a mode deliberately, write a prompt with a clear result, follow a job to completion, and find its report and artifacts.

## Next

Eleven ready-to-use prompts to start from: [Task templates](../task-templates.md). Then [Update, move or remove](09-update-move-or-remove.md), or [When something is wrong](10-when-something-is-wrong.md).
