# Codex CLI provider

**Status: included, switched off.** The adapter (`src/taskweave_mcp/providers/codex.py`) is complete and tested with a stand-in CLI, but it offers **no permission mode** until the real-CLI check below passes. Until then, putting `codex` in `TASKWEAVE_PROVIDERS` stops start-up with that instruction. This follows the project's admission rule: a mode name means the same thing for every provider ([permission-modes.md](../permission-modes.md), "What each mode guarantees").

It was written from the Codex CLI's source at release `rust-v0.166.0` (`codex-rs/exec/src/cli.rs`, `utils/cli/src/shared_options.rs`, `exec/src/exec_events.rs`), not from a signed-in run.

## How a job runs

```text
codex exec --json --skip-git-repo-check --ignore-user-config --ignore-rules --ephemeral \
  -C <working folder> -c approval_policy="never" --sandbox <sandbox> [-m <model>] \
  [-c model_reasoning_effort="<effort>"] -- <wrapped prompt>
```

- `--json` prints one event per line. The report is the text of the last `agent_message`. A `turn.failed` or `error` event fails the job with `codex_reported_error`. Command output, file changes and reasoning in the other events never leave the job folder.
- `--ignore-user-config` and `--ignore-rules` keep the user's own Codex settings and rules out of the job, like Claude's `--restricted`. `--ephemeral` keeps no session between jobs.
- The prompt is one argument after `--`, so it can never become a flag.
- Effort levels `low`, `medium`, `high`, `xhigh` and `max` map to `model_reasoning_effort`. There is no fallback model.
- **No cost cap:** Codex cannot stop a job at a dollar amount, so a cap needs `TASKWEAVE_CODEX_ALLOW_UNCAPPED=true`. Codex reports tokens, not dollars, so its jobs do not count towards the daily and monthly budgets.
- On Windows, npm's `codex.cmd` starts `codex.js`, which TaskWeave runs as `node codex.js`, never through `cmd.exe`.

## Candidate modes and known gaps

| TaskWeave mode | Codex sandbox | Expected gaps against the guarantee table |
|---|---|---|
| `read_only` | `read-only`: no writes and no network, enforced by the operating system's sandbox | Codex can still **run commands** (read-only), and its read-only policy may let them **read outside the workspace**. Either one keeps the mode unoffered. |
| `read_write` | `workspace-write` with `network_access=false` | The same command gap; **deleting** files in the workspace is not prevented. |
| `full_repository` | not mapped | Codex has no allow-list of commands that TaskWeave could map; not offered. |

Data folders and attachments are not supported for Codex jobs: Codex's `--add-dir` makes a folder *writable*.

## Verify it (owner, Windows, real CLI)

1. Install the Codex CLI and sign in (`codex login`; `codex login status` must say you are logged in).
2. Run the harness with the candidate modes. It costs a little model usage and runs in a throwaway folder:

   ```bash
   python scripts/verify_permissions.py --provider codex --candidate
   ```

3. For each mode that prints `PASS`, and only those, add it to `VERIFIED_MODES` in `providers/codex.py`, update the table above, and record the run (CLI version, date, output) in `docs/permission-modes.md` under "Evidence".
4. Then `TASKWEAVE_PROVIDERS=claude,codex` starts, and a client chooses Codex per job with `provider="codex"`.

## Settings

The usual provider settings with the `TASKWEAVE_CODEX_` prefix ([providers.md](../providers.md)): `TASKWEAVE_CODEX_BIN` (default: `codex` on PATH), `TASKWEAVE_CODEX_MODELS`, `TASKWEAVE_CODEX_DEFAULT_MODEL`, `TASKWEAVE_CODEX_EFFORTS`, `TASKWEAVE_CODEX_DEFAULT_EFFORT`, `TASKWEAVE_CODEX_PASS_ENV` and `TASKWEAVE_CODEX_ALLOW_UNCAPPED`. Only `CODEX_HOME` (where Codex keeps its sign-in) is passed through by default. An API key (`OPENAI_API_KEY` or `CODEX_API_KEY`) must be listed in `TASKWEAVE_CODEX_PASS_ENV`.
