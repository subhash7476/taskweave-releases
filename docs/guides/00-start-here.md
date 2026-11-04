# Start here

Verified with TaskWeave 0.2.0rc1, tunnel-client: not used, on 2026-11-03

## Goal and time

This page helps you choose the shortest path from nothing to a real task done by TaskWeave. Reading it takes about five minutes. The Claude Desktop path takes about 30 minutes after the prerequisites are ready.

TaskWeave runs long tasks in the background through your own signed-in Claude Code. You start a job from an AI client, close the chat if you like, and come back to a report and any files the job produced. It is not a chat window of its own, and it does not replace Claude Code: it uses the Claude Code you already installed and signed in.

## You need

- A Windows, macOS or Linux computer where you can install software.
- Nothing else to install first besides Claude Code: if you have no Python 3.10 or newer, the installer sets up a private copy for TaskWeave.
- Claude Code installed and signed in.
- A web browser to open the [TaskWeave releases page](https://github.com/subhash7476/taskweave-releases/releases/latest). It is public: no GitHub account or invite is needed.
- About 30 minutes for the Claude Desktop path, or about an hour for the ChatGPT path if an administrator has to grant permissions.
- A willingness to copy a command into a window and click through dialogs. This is not a guide for someone who has never opened a terminal and will not.

## Steps

1. Decide which path you need:

   | Path | Choose it when | What it needs | Time |
   |---|---|---|---|
   | Claude Desktop | You want the shortest path and no typed commands after the prerequisites. | Python, Claude Code, Claude Desktop. | About 30 minutes. |
   | ChatGPT | You specifically need ChatGPT, for example because it is the client your team uses. | Everything above, plus an OpenAI tunnel, a runtime key, workspace permissions and a plan that allows custom connectors. | About an hour, including a wait of up to 30 minutes for role changes. |
   | Another MCP client | You already use a client that can speak HTTP or stdio. | The same prerequisites, plus the client's own MCP setup. | Depends on the client. |

2. If you chose Claude Desktop, continue with [Before you install](01-before-you-install.md), then [Install TaskWeave](02-install-taskweave.md).
3. If you chose ChatGPT, read [Prepare ChatGPT on OpenAI](05-prepare-chatgpt-on-openai.md) first. It explains the administration steps and the wait before you install anything else.
4. If you chose another MCP client, read the README's [Use remotely (HTTP behind a tunnel)](../reference.md#use-remotely-http-behind-a-tunnel) section after installing TaskWeave. The guides below still explain the first-run setup and the job model.

**The honest advice:** start with Claude Desktop. The ChatGPT path ends in OpenAI's administration pages and a wait of up to 30 minutes. Do it only if you specifically need ChatGPT.

**Safety headline:** a job is read-only by default. It can read files in the workspace and write a report, but it cannot create, edit or delete files, run commands, use git or reach the network. Writes and shell access are separate permission modes that an operator must allow. Read [Permission modes](../reference.md#permission-modes) before you raise the limit.

## You should see

You know which path you are on, what it needs, and roughly how long it takes. You also know that the default permission mode is read-only.

## If it does not match

| What you see | What to do |
|---|---|
| The releases page does not open | Check the internet connection, or a company proxy that blocks GitHub. The address is https://github.com/subhash7476/taskweave-releases/releases/latest. |
| You do not have Python or Claude Code | Ask a technical friend once for help with the prerequisites. [Before you install](01-before-you-install.md) lists every check. |
| You need ChatGPT but cannot add custom connectors | Ask your ChatGPT workspace administrator. The plan and the workspace decide this; TaskWeave cannot change it. |
| You are not sure which client you have | If it is Claude Desktop, use the Claude Desktop path. If it is a browser chat other than ChatGPT, read its MCP documentation first. |

## Done when

You have chosen a path and opened its first guide.

## Next

- Claude Desktop: [Before you install](01-before-you-install.md)
- ChatGPT: [Prepare ChatGPT on OpenAI](05-prepare-chatgpt-on-openai.md)
- Something wrong already: [When something is wrong](10-when-something-is-wrong.md)

## Glossary

- **Claude Code**: Anthropic's coding agent, the program that does the work of a job. TaskWeave uses the copy you installed and signed in.
- **Agent**: the AI program that reads files and writes a report. In these guides it means Claude Code.
- **Workspace**: the one folder a job may read. With a write mode, it may also change files there.
- **Job**: one task sent to TaskWeave. It has a job ID, a prompt, a status, a report and sometimes files.
- **Thread**: a group of jobs from one conversation, so follow-up turns stay together.
- **MCP**: the protocol AI clients use to call TaskWeave's tools. You do not need to understand it to follow these guides.
- **Connector**: ChatGPT's name for an MCP server it can call.
- **Tunnel**: a private path from OpenAI to a program on your computer, so no inbound port is opened.
- **Runtime key**: the secret the tunnel client uses to authenticate. TaskWeave never stores it.
- **Release**: a tagged set of files on the public TaskWeave releases page.
- **EULA**: the End User License Agreement, the terms you accept once when the installer asks.
- **Wheel**: the Python package file that contains TaskWeave, ending in `.whl`.
- **Checksum**: a fingerprint of a file, used to notice a wrong or incomplete download. It does not prove who published the file.
- **Virtual environment**: a private folder for one program's Python packages, so it does not mix with other programs.
