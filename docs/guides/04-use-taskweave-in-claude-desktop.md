# Use TaskWeave in Claude Desktop

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Add TaskWeave to Claude Desktop with the button, restart Claude Desktop, and see the TaskWeave tools. About five minutes after the setup.

## You need

- A saved TaskWeave profile from [First-run setup](03-first-run-setup.md).
- Claude Desktop installed.
- The control center open on the Setup page or the Connect step; run `taskweave-control --setup` if needed.
- On Linux, Claude Desktop is not an official application, so use the snippet instead of the button.

## Steps

1. **Shortcut, if you only need Claude Desktop:** download `TaskWeave-<version>.mcpb` from the [releases page](https://github.com/subhash7476/taskweave-releases/releases/latest), double-click it, choose **Install** in Claude Desktop, and pick your workspace folder in the form it shows. Then skip to step 6. Otherwise, in the Setup wizard, open **Connect** and choose **Claude Desktop**. The page lists the Claude Desktop config file it found. If it found more than one, choose the one with today's date that contains your other servers.

2. Press **Preview the change**. The preview shows the command and arguments that will be added, the names of the other servers in the file, and never their values. It also says whether the TaskWeave entry is new, already correct, or different.

3. If the entry is **different**, tick **Replace the existing TaskWeave entry**. The preview shows the old and new command and arguments; it never shows environment values.

4. Quit Claude Desktop completely, including from the system tray. The preview repeats this because a running Claude Desktop can rewrite the file.

5. Press **Add to Claude Desktop** (or **Replace the entry**). The page writes the entry, keeps a byte-for-byte backup next to the file, and shows the backup's file name.

6. Start Claude Desktop again. Open a chat and look for the TaskWeave tools. You should see `start_task`, `check_task`, `get_task_result`, `cancel_task`, `list_tasks`, `list_task_artifacts`, `get_task_artifact` and `list_providers`, plus the older names such as `start_research_job`, which keep earlier setups working.

7. Ask for a small read-only task in plain words, for example: "Use TaskWeave to read README.txt in the workspace and summarize it in one sentence. Do not modify anything." The job runs in the background; ask Claude Desktop to check it until it finishes.

8. To undo the change, quit Claude Desktop, find the backup file the page named next to `claude_desktop_config.json`, put it back in place of the config file, and start Claude Desktop again.

9. If the button is hidden on Linux, or the file is refused, copy the snippet the page shows and add it under `mcpServers` in the file Claude Desktop opens from **Settings, Developer, Edit Config**. The snippet is the same entry the button writes.

## You should see

- The config candidate path, or a box to paste the path that **Edit Config** opened.
- A preview naming the other servers, without their values.
- A backup file named `claude_desktop_config.json.taskweave-backup-<timestamp>` after applying.
- The TaskWeave tools (`start_task` and the rest) in a new Claude Desktop chat.
- A completed read-only job with a short report.

## If it does not match

| What you see | What to do |
|---|---|
| `No claude_desktop_config.json was found` | Open Claude Desktop, choose **Settings, Developer, Edit Config**, then paste the path it opens into the box and preview again. The button never creates a missing file. |
| `That file is not valid JSON` | The message names the line and column. Open the file from **Edit Config**, fix the JSON, and preview again. The file was not changed. |
| `That is not Claude Desktop's config file` | The path you pasted is not where Claude Desktop keeps its config. The message lists the right places; open **Settings, Developer, Edit Config** in Claude Desktop and paste the path of the file it opens. |
| `The file changed after the preview` | Something rewrote the file. Press **Preview again**, then apply. |
| `Claude Desktop may be running` | Quit Claude Desktop completely, including from the system tray, then press **Preview again** and apply. |
| The tools do not appear | Quit Claude Desktop completely and start it again. Check the config file has a `taskweave` entry under `mcpServers`. |
| `Command not found` for the taskweave command | The entry uses the command from the TaskWeave install. Reinstall or check the profile's command; the preview shows the exact command. |
| You cannot find the backup | The page shows its name after applying; it is next to `claude_desktop_config.json`. |
| Linux has no button | Use the snippet under `mcpServers`; Claude Desktop is not an official Linux application. |

## Done when

Claude Desktop lists the TaskWeave tools after a complete restart, and one read-only job finishes with a report.

## Next

[Your first real task](07-your-first-real-task.md)
