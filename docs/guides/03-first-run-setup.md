# First-run setup

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

Use the control center's Setup page to choose folders, check the setup, run one tiny test task, and connect Claude Desktop. About 10 minutes after the install.

## You need

- TaskWeave installed, as in [Install TaskWeave](02-install-taskweave.md).
- Claude Code installed and signed in, as in [Before you install](01-before-you-install.md).
- The control center open. If it is not, use the **TaskWeave Control Center** shortcut (Start menu on Windows, Applications on a Mac, the app menu on Linux) and open **Setup**, or run `taskweave-control --setup` in a new terminal window.

## Steps

1. Open the **Setup** page. The control center opens it automatically after a fresh install; the address is `http://127.0.0.1:8765/#setup`.

2. **Folders.** Choose a workspace and a jobs folder. The workspace is the only folder a read-only job may read; the jobs folder holds one folder per job. You can type a path, browse for one, or use **Create it for me**. A custom Claude CLI path may be `claude.exe` or npm's `claude.cmd`; TaskWeave runs the `claude.exe` either way.

3. Press **Next: check the setup**. The **Checks** step shows a card for each check. A card marked **ok** is ready; **warn** is usable but worth reading; **fail** means the setup will not work yet.

4. Fix every **fail**, then press **Check again**. The usual fixes are: sign in to Claude Code, set the Claude CLI path to the real `claude.exe`, create a missing workspace or jobs folder, or change the settings file path.

5. Press **Next: test task**, then **Send the test task**. This sends one tiny read-only request with no tools to your signed-in Claude Code. It should end at **Ready: the agent answered the test task** in under a minute. It spends some model usage, capped at the amount shown on the page ($0.50). Even this one-line request can cost about $0.20 to $0.25, because Claude Code sends a large system prompt the first time.

6. If the test fails, read the message. A sign-in problem shows Claude Code's own message; a missing CLI, or an npm install without its `claude.exe`, shows the path problem. Nothing is installed or changed by the test itself. Run it again after the fix.

7. Press **Next: connect a client**, choose **Claude Desktop**, and follow [Use TaskWeave in Claude Desktop](04-use-taskweave-in-claude-desktop.md). If you cannot pass the test but the checks are otherwise fine, you can continue and fix the test later.

8. Check the spending limits in **Settings**. The installer's starter profile caps one job at $2 and all jobs together at $10 a day; a used-up daily budget refuses new jobs until midnight. Raise or empty them to suit you; the Health section shows what was spent today.

9. Save the profile if the page asks you to. The profile name is what the Claude Desktop entry and the command line use; the default `taskweave` is fine.

## You should see

- Four steps: **Folders**, **Checks**, **Test task**, **Connect**.
- Every check card green or explained.
- **Ready: the agent answered the test task** with a Claude Code version.
- No `onboarding-test` folder left in the jobs folder after the test.
- The profile saved and the Connect step offering Claude Desktop or ChatGPT.

## If it does not match

| What you see | What to do |
|---|---|
| Claude Code CLI check fails | Sign in by running `claude` once, or set the profile's Claude CLI path. An npm `claude.cmd` is fine; TaskWeave runs the `claude.exe` it starts. |
| The test fails with a sign-in message | Run `claude` in a terminal, finish the sign-in, and send the test again. |
| The test fails with a model or cost message | Read the message. The test is capped; a model that is not allowed in the profile fails before it starts. |
| The test says a test is already running | A second browser tab started one. Wait for the first to end. |
| A folder check fails | Create the folder with **Create it for me**, or choose an existing folder. |
| The checks say the `claude.exe` behind a `.cmd` is missing | Claude Code was installed with npm and its install script was skipped. Run `npm install -g @anthropic-ai/claude-code` again, or install the native Claude Code (`irm https://claude.ai/install.ps1 \| iex` in PowerShell) and choose its `claude.exe`, usually `%USERPROFILE%\.local\bin\claude.exe`. |
| The test fails with **The reply did not contain the expected text** on TaskWeave 0.2.0rc1 | Claude Code 2.1.317 and newer print their result in a shape 0.2.0rc1 does not read. Install a TaskWeave build that includes the fix, then send the test again. |
| The test says the cost cap was reached | The test request cost more than its cap. Send it again once; if it fails again, continue to Connect and set a cheaper default model in Settings. |
| You cannot pass the test today | Continue to the Connect step; the checks still tell you what to fix. Do not raise the permission mode to make the test pass. |

## Done when

The test task ends **Ready**, the profile is saved, and the Connect step shows your client choices.

## Next

[Use TaskWeave in Claude Desktop](04-use-taskweave-in-claude-desktop.md), or [Prepare ChatGPT on OpenAI](05-prepare-chatgpt-on-openai.md) if you chose ChatGPT.
