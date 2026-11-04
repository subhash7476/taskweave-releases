# TaskWeave in Docker

Run TaskWeave and Claude Code together in a container, reachable over TaskWeave's HTTP transport. Use this for a server that other people's MCP clients call, for untrusted callers, or for `full_repository` jobs you want contained: Claude Code's tool limits are not an operating-system sandbox, but in the container they run as an unprivileged user that sees only the folders you mount.

The image is built from `packaging/docker/Dockerfile`: Node 22 (Claude Code's runtime), Python 3.11, git, Claude Code at a pinned version, and TaskWeave from its wheel at the tested dependency versions (`constraints.txt`). No secret is baked in.

## Build

**From the release files** (no source code needed). From the [releases page](https://github.com/subhash7476/taskweave-releases/releases/latest), download `Dockerfile`, `constraints.txt`, the wheel (`taskweave_mcp-X.Y.Z-py3-none-any.whl`) and `SHA256SUMS.txt` into an empty folder, check them, put the wheel in a `dist` folder, and build there:

```bash
sha256sum -c SHA256SUMS.txt --ignore-missing    # macOS: shasum -a 256 -c SHA256SUMS.txt --ignore-missing
mkdir dist && mv taskweave_mcp-X.Y.Z-py3-none-any.whl dist/
docker build -t taskweave:X.Y.Z .
```

**From a checkout** of the release commit, after building the wheel:

```bash
python -m build --wheel
docker build -f packaging/docker/Dockerfile -t taskweave:X.Y.Z .
```

`--build-arg CLAUDE_CODE_VERSION=<version>` picks another Claude Code; check it against the CLI notes in the README first.

## Run

```bash
export TASKWEAVE_AUTH_TOKEN="$(python -c 'import secrets; print(secrets.token_urlsafe(32))')"
export ANTHROPIC_API_KEY=...        # an Anthropic API key; billed to that API account
docker run --rm -p 127.0.0.1:8080:8080 \
  -e TASKWEAVE_AUTH_TOKEN -e ANTHROPIC_API_KEY \
  -v "$PWD/workspace:/workspace:ro" \
  -v taskweave-jobs:/data/jobs \
  taskweave:X.Y.Z
```

- **`-p 127.0.0.1:8080:8080`** publishes the port on this machine only. Put a reverse proxy or tunnel with TLS in front for anything remote (README, "Use remotely").
- **`TASKWEAVE_AUTH_TOKEN`** is required: the HTTP transport refuses to start without a token of 16 characters or more. Clients send `Authorization: Bearer <token>` to `http://<host>:8080/mcp`.
- **`ANTHROPIC_API_KEY`** signs Claude Code in non-interactively. TaskWeave passes `ANTHROPIC_*` variables to Claude Code and removes other secrets from its environment. Bedrock and Vertex work the same way with their variables and `TASKWEAVE_PASS_ENV` ([Sign in with an API key, Bedrock or Vertex](reference.md#sign-in-with-an-api-key-bedrock-or-vertex)).
- **The workspace** is mounted read-only above, which is all a `read_only` job needs. For `read_write` or `full_repository` jobs, mount it read-write and raise `TASKWEAVE_MAX_PERMISSION_MODE` (`-e TASKWEAVE_MAX_PERMISSION_MODE=read_write`); the container runs as user 10001, so that user needs write access to the folder (or add `--user "$(id -u):$(id -g)"`).
- **Jobs** live on the `taskweave-jobs` volume, so reports survive a new container.
- **Spending limits** default to $2 per job and $10 a day for all jobs (`TASKWEAVE_MAX_COST_USD`, `TASKWEAVE_DAILY_BUDGET_USD`); override them with `-e`.

## Check it

```bash
curl -i -X POST http://127.0.0.1:8080/mcp                                  # 401: the token is enforced
docker run --rm --entrypoint taskweave taskweave:X.Y.Z doctor             # the same checks as Health
```

The control center (`taskweave-control`) is a local page for one person's machine and is not started in the container; never publish it.
