# TaskWeave

**Hand a long task to Claude Code, close the chat, and collect the finished report later.**

TaskWeave runs tasks in the background with the Claude Code you already have signed in on your computer: code reviews, research across a folder of documents, reports, documentation, tests. Ask from Claude Desktop, ChatGPT or any other MCP client; every request returns in seconds, so a long job never times out the chat. By default a job can only **read** the folder you choose.

**[Download the latest release](https://github.com/subhash7476/taskweave-releases/releases/latest)** · User guides · Task templates · Reference · Licence agreement · Changelog

## Pick your way in

| You use | Start here |
|---|---|
| **Claude Desktop** | Download `TaskWeave-<version>.mcpb` from the release and double-click it. Claude Desktop installs the rest and asks for the folder TaskWeave may read: no Python and no installer. Claude Code must be installed and signed in. |
| **Claude Desktop, with the control center** | The full install below: a local page for jobs, settings, health checks and spending, and an **Add to Claude Desktop** button. |
| **ChatGPT** (Windows) | The full install, then Connect ChatGPT through OpenAI's tunnel client. |
| **A server or a shared machine** | The Docker image, reachable over HTTP with a token: TaskWeave in Docker. |
| **Another MCP client or agent** | The full install; TaskWeave speaks MCP over stdio or HTTP (Reference). |

## Install

You need Claude Code installed and signed in (Before you install lists the checks). The installer does the rest. It asks you to accept the licence agreement first.

1. From the [latest release](https://github.com/subhash7476/taskweave-releases/releases/latest), download the one zip for your computer: `windows-x64` (most Windows PCs; there is no Windows on ARM zip yet), `macos-arm64` (Apple silicon), `macos-x64` (Intel Macs), `linux-x64` or `linux-arm64`.
2. Extract it (on Windows: right-click, **Extract All**).
3. In the extracted folder, double-click `install.cmd` (Windows), or run `sh install.sh` in Terminal (macOS, Linux).

No Python? The installer sets up a private one. The zip carries every dependency, so nothing else is downloaded when Python is already there.

The installer checks every download against the checksums GitHub publishes, installs TaskWeave in its own folder, puts the `taskweave` commands on your PATH, adds a **TaskWeave Control Center** shortcut, and opens the setup page in your browser. `taskweave update` and `taskweave uninstall` take care of the rest later. Step by step: Install TaskWeave.

## What you can do with it

- **Ask for long work and come back to a result.** A codebase tour, a ranked bug review, a summary of a folder of PDFs, release notes from git history. Task templates has eleven prompts to start from.
- **Get files back.** CSV, Markdown, JSON, HTML and diagrams (Mermaid, DOT, SVG) arrive as separate files next to the report.
- **Keep follow-ups together.** Jobs from one conversation form a thread, and a follow-up can carry the previous report's context.
- **Choose how much the agent may do, per job.** Read only (the default), edit files, or run a fixed list of test, build and git commands; never more than the highest mode you allow.
- **Stay in control of cost.** A cost cap per job, daily and monthly budgets for all jobs, and every job's cost in the control center.
- **See everything locally.** Jobs, prompts, reports and settings stay on your computer, in a local control center.

## Works with your Claude sign-in

Your Claude subscription, an Anthropic API key, Amazon Bedrock or Google Vertex AI: whatever your Claude Code signs in with. Sign in with an API key, Bedrock or Vertex shows the settings for each.

## Safety in short

- Read-only by default; the mode is enforced by the server, never by the prompt.
- One workspace folder, chosen by you; a job cannot pick another.
- Keys and tokens are kept out of the agent's environment and out of logs.
- The HTTP server refuses to start without a token.
- Claude Code's tool limits are not an operating-system sandbox: for untrusted callers or for jobs that run commands, use a container or a separate user (Reference).

## Support

Report a problem as an [issue in this repository](https://github.com/subhash7476/taskweave-releases/issues). When something is wrong says what to include and what to remove first; the control center's **Copy diagnostics** button gives you a report with keys, e-mail addresses, your user name and your home folder already removed. Never post a key, token or tunnel ID.

TaskWeave is proprietary software, free to use under the End User License Agreement. This repository holds its releases and documentation; the source code is not published here.
