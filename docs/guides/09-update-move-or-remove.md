# Update, move or remove

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Keep TaskWeave healthy when a new release arrives, move it to another computer, or remove it without leaving secrets or stray jobs behind. About 10 minutes per task.

## You need

- The current TaskWeave install and its settings file.
- The release page for the version you want.
- The backup name the Claude Desktop button showed, if you used the button.
- A decision about whether you want to keep your jobs and settings.

## Steps

1. **Update.** When the control center shows that a new version is available, run this in a terminal (on Windows it continues in a new window; type `yes` if the licence agreement changed):

   ```text
   taskweave update
   ```

   It downloads the new release's installer, checks it, and installs the new version over the old one. Your settings and jobs stay. `taskweave update --check` only says whether a newer release exists. **Or by hand on Windows:** download the new release's zip for your computer, extract it, then double-click `install.cmd` in the extracted folder. The installer keeps your settings file and jobs; a newer version upgrades the package in place.

2. **Or by hand on macOS or Linux.** Download the new release's zip for your computer, extract it, open Terminal in the extracted folder, then run:

   ```bash
   sh install.sh
   ```

3. Check the version after the update:

   ```powershell
   taskweave --version
   ```

4. Run the readiness check:

   ```powershell
   taskweave doctor --json
   ```

5. **Roll back.** Download the previous release's zip, extract it, and run its installer. Your settings and jobs stay; the package goes back to the older version. Do not reuse a version number for a different file.

6. **Move to another computer.** Install fresh on the new computer. Copy the settings file and the jobs folder only if you need the history. Do not copy the virtual environment: it contains paths and binaries for the old computer. Reconnect Claude Desktop with the button, or add the snippet again.

7. **Remove the Claude Desktop entry.** Quit Claude Desktop, open the config file from **Settings, Developer, Edit Config**, and remove only the `taskweave` entry under `mcpServers`. If you used the button, the safest undo is to put back the backup it named. Start Claude Desktop again.

8. **Remove the tunnel profile (ChatGPT path).** In the control center's Connect page, remove the generated profile and start script, or delete the profile folder the page named. The runtime key is never stored there; if you saved it elsewhere, delete it too.

9. **Remove TaskWeave.** Close any tunnel client window first. On Windows, open **Settings, Apps, Installed apps**, find **TaskWeave** and choose **Uninstall**; or, on any system, run this in a terminal and type `yes` when it asks:

   ```text
   taskweave uninstall
   ```

   It stops the control center, removes the shortcuts, the PATH entry, the program and a private Python if the installer set one up. Your settings file and jobs stay; add `--remove-data` to remove them too. Your workspace folder is never removed.

10. **If the uninstaller is missing** (an install made by hand), stop every TaskWeave process, then delete the virtual environment folder. On Windows the default is under `%LOCALAPPDATA%\TaskWeave`; on macOS and Linux it is `~/taskweave`. This removes the program, not your settings or jobs.

11. **Check nothing is left running.** Stop the control center with the **Stop TaskWeave Control Center** shortcut (Start menu on Windows, Applications on a Mac, the app menu on Linux) or `taskweave-control --stop`, and close the tunnel client window. If a job was running, it is stopped when its server stops; a restart may mark it failed or re-attach it, depending on whether its process is still alive.

12. **Keep the release files you trust.** If you may need to roll back later, keep the zip of the version you trust; it holds its own `SHA256SUMS.txt`. Do not keep a runtime key in that folder.

## You should see

- The new version from `taskweave --version`.
- `doctor` reporting the same folders and settings as before.
- The Claude Desktop tools still listed after a restart.
- No `taskweave` entry left in the config after removal.
- No TaskWeave process, tunnel client or control center left running.
- The jobs folder and settings file either kept deliberately or deleted deliberately.

## If it does not match

| What you see | What to do |
|---|---|
| The installer says the existing settings file has no matching profile | Pass a profile name the file has, or add one in the control center. The installer never edits the settings file. |
| The new version does not start | Roll back to the previous wheel with the installer; then report the failure with `taskweave doctor --json`. |
| Claude Desktop still shows the old tools | Quit it completely, including from the system tray, and start it again. |
| You cannot find the config file | Open **Settings, Developer, Edit Config**; the file it opens is the one the button uses. |
| The tunnel still connects after removal | The start script or tunnel client is still running. Stop its window and remove the generated script. |
| A job folder contains a secret | Do not publish it. Remove the file, rotate the secret if it was real, and report it. |
| You deleted the wrong folder | Restore from your backup. The installer does not keep a copy of your jobs or settings. |

## Done when

The version you intended is installed, the client still works after a restart, and anything you removed is gone without leaving a runtime key or a running process behind.

## Next

[When something is wrong](10-when-something-is-wrong.md)
