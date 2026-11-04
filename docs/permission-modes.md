# Permission modes: design, matrix and limits

Status: implemented and tested with the stub CLI (automated suite). Enforcement by the real Claude CLI
(2.1.286) was checked with throwaway-directory probes and `scripts/verify_permissions.py`; see
"Evidence". Re-run that script after upgrading the CLI.

## What each mode guarantees, for every provider

A mode means the same thing whichever coding CLI runs the job. These are the outcomes a provider
must deliver; how it does so is up to its adapter (Claude's mechanism is described in the rest of
this document). A provider lists a mode in `supported_permission_modes` only after
`scripts/verify_permissions.py --provider <id>` passes for it with the real CLI. A mode it cannot
enforce is not offered, and a request for it is rejected before a job is created.

| Must never happen | `read_only` | `read_write` | `full_repository` |
|---|---|---|---|
| A file in the workspace created, changed or deleted | never | creating and editing allowed; deleting, renaming and moving never | allowed |
| A file written outside the workspace | never | never | never |
| A file outside the workspace read | never | never | never |
| An arbitrary command run | never | never | only the allow-list |
| A push, network fetch or dependency install | never | never | never |
| `.git/` or the agent's own settings folder written | n/a | never | never (git itself excepted) |
| User or project agent settings, hooks or MCP servers loaded | never | never | never |
| TaskWeave's settings or another provider's credentials visible to the agent | never | never | never |

Where a provider is stronger than this table (for example an OS sandbox that also blocks reads by
the commands it runs), its page in `docs/providers/` says so. Where it would be weaker on any row,
that mode is not offered for it.

## Principles (Claude Code)

- The server turns the validated mode into CLI flags itself (`claude_policy.policy_for`; mode names,
  validation and the ceiling are provider-neutral, in `permissions`). The prompt is passed as a
  single argument and cannot add or change flags; the prompt text describing a mode is advisory only.
- Default is `read_only`. The operator sets a ceiling with `TASKWEAVE_MAX_PERMISSION_MODE` (default `read_only`). The deprecated settings `MODEL_CLI_MAX_PERMISSION_MODE`, `CLAUDE_BRIDGE_MAX_PERMISSION_MODE`, `MODEL_CLI_ALLOW_EDITS`, and `CLAUDE_BRIDGE_ALLOW_EDITS` remain accepted for migration. Every explicitly supplied setting is a constraint: all must agree or startup fails closed. Explicit empty or whitespace-only values are rejected.
- Anything not pre-approved is denied (`--permission-mode dontAsk`, `--permission-prompts none`).
- A working directory is **not** a security boundary. File-tool confinement comes from `--restricted`, and
  shell commands are not confined at all (see Limits).

## Flags every job gets

| Flag | Effect (verified) |
|---|---|
| `--restricted` | File tools refuse paths outside the working directory, including through symlinks/junctions; user, project and local settings files are ignored (a repository cannot plant hooks or permission rules); writes to `.git/` and `.claude/` are refused. With CLI 2.1.317 it also makes `--output-format json` print one JSON array of every event instead of the single result document; TaskWeave takes only the final `result` element. |
| `--strict-mcp-config` | No MCP servers load. Without it, `--tools Read,Glob,Grep` still left **117 MCP tools** (Gmail, Docs, ...) loaded on this machine. The earlier read-only configuration had this gap. |
| `--permission-mode dontAsk` + `--permission-prompts none` | Unlisted tools and commands are denied; nothing can prompt. |
| `--tools <list>` | Only these built-in tools exist for the session (and its sub-agents). |
| `--allowedTools` / `--disallowedTools` | Pre-approval and explicit denial (deny wins). |

The bridge fails closed: if `claude --help` lacks any required flag, the job fails with
`cli_missing_required_flags` and no process is started with restrictions missing.

## Tool and capability matrix

`Y` = allowed, `-` = not available/denied. "Enforced by" is the mechanism, not a promise beyond it.

| Capability | read_only | read_write | full_repository | Enforced by |
|---|---|---|---|---|
| Read (`Read`), list (`Glob`), search (`Grep`) in workspace | Y | Y | Y | `--tools` |
| Read/list/search outside workspace, or through a link leaving it | - | - | - | CLI `--restricted` (file tools) |
| Create files (`Write`) | - | Y | Y | `--tools` |
| Edit files (`Edit`) | - | Y | Y | `--tools` |
| Write into `.git/`, `.claude/` | - | - | - | CLI `--restricted` |
| Notebook edit | - | - | - | tool not in `--tools` |
| Delete / rename / move files | - | - | Y, only tracked files via `git rm` / `git mv`, untracked via `git clean` | no such tool; git refuses paths outside the repository |
| Shell (`Bash`) | - | - | Y, allow-list only | `--tools` + `--allowedTools Bash(pattern)` |
| PowerShell tool | - | - | - | not in `--tools` |
| Local git: status diff log show branch add commit rm mv checkout switch restore stash clean | - | - | Y | pattern allow-list |
| `git push/fetch/pull/clone/remote/config/submodule/worktree`, `git -c`, `git -C`, `git --...` | - | - | denied | pattern deny rules |
| `git ... --output`, `commit -F/--file/--template` (write/read outside repo) | - | - | denied | pattern deny rules (found by probe) |
| Tests/build: python -m pytest/unittest, pytest, npm test/run test/build/lint, cargo test/build/check, go test/build/vet, dotnet test/build | - | - | Y | pattern allow-list |
| Dependency installation (pip, npm, yarn, pnpm, cargo, go, dotnet, apt, brew, ...) | - | - | denied | pattern deny rules; policy is "pre-install dependencies yourself" |
| `curl`, `wget`, `ssh`, `scp`, `sudo`, `rm`, `mv`, `cp`, shells, `env`, `printenv` | - | - | denied | pattern deny rules (and not on the allow-list) |
| Web search / fetch | off | off | off | opt in for all modes with `TASKWEAVE_ALLOW_WEB=true` |
| MCP connectors, skills, sub-agents, scheduling, notifications | - | - | - | `--strict-mcp-config`, not in `--tools` |
| Bypass-permissions / auto mode | - | - | - | never passed; `--restricted` refuses bypass |
| Credentials in Claude's environment | scrubbed | scrubbed | scrubbed | server (`runner.child_env`) |

## Enforcement classes

**Server-enforced (tested in the automated suite):** mode validation and exact values; default `read_only`;
operator ceiling at submission and again at launch; mode recorded and immutable (guard in `_save`; tampered or
unknown values fail the job instead of defaulting upward); the exact `--tools/--allowedTools/--disallowedTools`
flags per mode; no bypass flags; flags independent of prompt content; fail-closed capability check; environment
scrubbing; fixed cwd; process-tree termination on cancel/timeout; cancelling a finished job changes nothing.

**CLI-enforced (the CLI applies it; not provable with the stub; verified manually):** removal of unlisted tools,
workspace confinement of file tools including links, ignoring of settings files and MCP servers, denial of
`.git/`/`.claude/` writes, and the shell pattern allow/deny rules.

**OS-enforced:** nothing. The bridge does not create a sandbox, drop privileges or restrict the network. Claude and
every command it runs have your OS user's rights.

**Advisory only:** the boundary text added to the prompt, and anything the model says it will or will not do.

## Limits and known risks (read before enabling `full_repository`)

1. **Shell patterns are not containment.** Prefix allow-lists are bypassable by options the pattern did not
   anticipate. Probing found real escapes (`git log --output=<outside>` wrote a file outside the workspace; `git commit -F
   <outside file>` read one) and those are now denied, but unknown options may remain. Treat every allowed command as
   able to do anything that command can do.
2. **Test, build and git commands execute repository-defined code** (test files, `package.json` scripts, build
   scripts, git hooks in `.git/hooks`). A hostile repository can therefore run code as your user in `full_repository`
   regardless of the allow-list. Only enable it for repositories you trust, in a container or low-privilege account.
3. **`Write`/`Edit` cannot be restricted to non-destructive use.** In `read_write` a file can be overwritten or
   emptied. Use version control and review diffs.
4. **Symlink/junction confinement covers the CLI file tools only.** A command run in `full_repository` that creates
   a link, or code run by a test, can reach outside the workspace. The bridge does not scan or police links.
5. **The workspace itself is trusted input.** `CLAUDE.md` files inside it still guide the model (advisory), and
   file contents can contain prompt-injection text. The flags, not the model, are the control.
6. **Network:** web tools are off by default, but test and build commands may use the network on their own.
   Only the shell allow-list, not a firewall, limits it.
7. **Credentials the CLI needs stay in its environment** (`ANTHROPIC_*`, `CLAUDE_CODE_*`, and anything in
   `TASKWEAVE_PASS_ENV`). The allow-list has no command that prints the environment, and `echo $VAR` was denied
   in probes, but a test script can read it.
8. **Re-verify on CLI upgrades.** Behaviour comes from CLI 2.1.286. Run `scripts/verify_permissions.py` after any upgrade.
9. A job's own `metadata.json`/`prompt.txt` live in the jobs directory. Startup rejects a jobs directory inside the
   workspace so Claude cannot read other jobs.

## Evidence (real CLI 2.1.286, Windows, throwaway directories)

- `--tools Read,Glob,Grep` alone: 5 built-ins but 117 MCP tools still loaded; with `--strict-mcp-config`: 0.
- `--restricted`: read of an absolute path outside the workspace refused; read through a directory junction refused
  ("resolves through a symlink"); `Write` outside via absolute path and via `..` refused; `Write` to
  `.claude/settings.json` and `.git/hooks/pre-commit` refused.
- `Grep` with an absolute or `../` path outside the workspace refused; `Glob` with an absolute pattern outside refused,
  and with a `../outside/*.txt` pattern returned "No files found" (no leak).
- `read_only` flags: `Write`, `Edit`, `Bash` reported "No such tool available".
- `full_repository` flags: `git add/commit`, `git rm` ran; `git push`, `curl`, `rm -rf`, `git -C`, `git -c` were
  denied; `echo hello` ran (the CLI auto-allows some harmless commands); `echo $VAR` and redirects to outside paths were denied.
- Without deny rules, `git log --output=<outside>` and `git diff --output=<outside>` **created files outside the
  workspace**; with `Bash(git*--out*)` they were denied.
- `scripts/verify_permissions.py` (real jobs through `JobManager` with the shipped flags): 0 violations in all three
  modes; `read_write` created and edited files; `full_repository` created, committed and `git rm`'d.

- Read-only data folders (`--add-dir=<folder>` with the `read_only` flags, Claude Haiku): `Read` and `Grep` in the added
  folder worked; the same read without `--add-dir` was refused, as was a read of a folder that was not added and a
  read through a junction in the added folder that pointed outside it; `Write` into the added folder: "not available".
  With the `read_write` flags, `--add-dir` plus `--disallowedTools "Edit(//<folder>/**)" "Write(//<folder>/**)"` did
  **not** stop a `Write` into the added folder, so data folders are only ever added to `read_only` jobs.

- Read-only attachments (`scripts/verify_attachments.py`, real CLI 2.1.286, Windows, Haiku alias, one `read_only`
  job through `JobManager`; 2026-11-01): a text file, a one-page PDF and a solid-red PNG were staged under the job
  directory and added with `--add-dir`. The model read all three (the text token, the PDF token and the colour
  "red") although the prompt names only the stored file names, not the folder. Asked to create `WRITE_PROBE.txt`
  beside them it answered `WRITE=not_possible`; no such file existed afterwards, the three staged files and the
  workspace were unchanged, `permission_denials` was empty and no host path appeared in the response. Cost about
  $0.017, 13 s. A first attempt failed with a transient OAuth refresh error (the desktop app was refreshing the same
  sign-in) and cost nothing; the retry passed. One run, one model: a sample, and "not writable" rests on the model's
  answer, the filesystem result and the `read_only` tool list, not on an operating-system lock.

These are samples, not proof of completeness.
