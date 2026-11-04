# Before you install

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Make sure the three things TaskWeave depends on are ready before you run the installer: Python, Claude Code with a working sign-in, and the public releases page. About 10 minutes, plus download time.

## You need

- A Windows, macOS or Linux computer.
- Permission to install software for your user.
- The [TaskWeave releases page](https://github.com/subhash7476/taskweave-releases/releases/latest) open in a browser. It is public; no GitHub account is needed.
- A download folder, for example `C:\TaskWeave-download` or `/home/you/taskweave-download`. Plain letters and digits are safest. If your Windows user name has other characters (such as é, ü or letters of another alphabet), the ChatGPT launcher uses Windows' short folder names instead; if your drive has those turned off, the Connect page says so, and installing with `-Root C:\TaskWeave` fixes it.

## Steps

1. Python is optional. Open PowerShell (Windows) or Terminal (macOS and Linux) and run:

   ```text
   python --version
   ```

   If you see `Python 3.10` or newer (on macOS or Linux, `python3 --version` is also fine), the installer uses it. If the command is not found, opens the Microsoft Store, or shows an older version, you do not need to fix it: the installer downloads a private Python 3.12 for TaskWeave into its own folder.

2. A private Python needs internet access to `files.pythonhosted.org` (for the uv tool) and `github.com` (for Python itself) during the install. If your network blocks them, install Python from <https://www.python.org/downloads/> instead and tick **Add python.exe to PATH** on Windows.

3. Check Claude Code:

   ```text
   claude --version
   ```

   You should see a version line. If the command is not found, install Claude Code from <https://www.anthropic.com/claude-code>, then open a new terminal so the command is found.

   **Windows: either install works.** Claude Code can be installed two ways:

   | Installed with | What `claude` is | Works with TaskWeave |
   |---|---|---|
   | The native installer: `irm https://claude.ai/install.ps1 \| iex` in PowerShell | `claude.exe`, usually in `%USERPROFILE%\.local\bin` | Yes |
   | npm: `npm install -g @anthropic-ai/claude-code` | `claude.cmd` in `%APPDATA%\npm`, which starts the `claude.exe` npm put in its `node_modules` folder | Yes: TaskWeave runs that `claude.exe` directly, never the `.cmd` |

   TaskWeave never runs a `.cmd` file, because `cmd.exe` cuts multi-line prompts short and lets prompt text run as commands. For an npm install it reads which `claude.exe` the `.cmd` starts and runs that instead. If npm was told to skip install scripts (`--ignore-scripts`), that `claude.exe` is missing; the checks then say so, and reinstalling with npm or using the native installer fixes it. On macOS and Linux either install works too.

4. Prove that Claude Code is signed in. Run this one-line task:

   ```text
   claude -p "Reply with exactly BRIDGE_JOB_OK. Do not use tools."
   ```

   You should see exactly `BRIDGE_JOB_OK`. If you see a sign-in message instead, run `claude` once in a terminal and finish the sign-in, then run the one-line task again.

5. Check that you can open the releases page: <https://github.com/subhash7476/taskweave-releases/releases/latest>. You should see the newest release and its files. You do not need to download anything yet.

6. Choose one folder for the downloaded installer files. Use a plain path with no other characters, for example `C:\TaskWeave-download` or `/home/you/taskweave-download`. Create it now.

7. On Windows, know what the download warning looks like. A downloaded `.cmd` or `.ps1` file may make Windows show **Windows protected your PC** or **This file came from another computer**. The safe action for a file from this repository is **More info**, then **Run anyway**. Do not do this for a file from a source you do not trust.

## You should see

- `Python 3.10` or newer, or a note to yourself that the installer will set up a private Python.
- A Claude Code version line, and on Windows a `where.exe claude` line ending in `claude.exe`.
- `BRIDGE_JOB_OK` from the one-line task.
- The TaskWeave releases page.
- A created download folder with a plain path.

## If it does not match

| What you see | What to do |
|---|---|
| `python` opens the Microsoft Store | Install Python from python.org and make sure **Add python.exe to PATH** was ticked. Open a new terminal and check again. |
| `python` is not found after installing | Close and reopen the terminal. If it still fails, the PATH was not updated; reinstall Python with the PATH option. |
| `claude` is not found | Install Claude Code from the official page, then open a new terminal. If it is installed, the command may be in a folder that is not on PATH; set the Claude CLI path in the TaskWeave profile later. |
| The one-line task asks you to sign in | Run `claude` once and finish the sign-in. Then run the one-line task again. |
| The one-line task prints something else | Read the message. If it says the native binary is not installed, Claude Code was installed with npm while install scripts were skipped; run `npm install -g @anthropic-ai/claude-code` again, or use the native installer. |
| The checks say the `claude.exe` behind `claude.cmd` is missing | npm skipped Claude Code's install script. Run `npm install -g @anthropic-ai/claude-code` again without `--ignore-scripts`, or run `irm https://claude.ai/install.ps1 \| iex` in PowerShell. |
| The releases page does not open | Check the internet connection, or ask whether a company proxy blocks GitHub. |
| Windows warns about the download | For a file from the TaskWeave releases page, choose **More info**, then **Run anyway**. |

## Done when

You know whether you have Python 3.10+, Claude Code prints `BRIDGE_JOB_OK`, you can open the releases page, and you have a plain download folder.

## Next

[Install TaskWeave](02-install-taskweave.md)
