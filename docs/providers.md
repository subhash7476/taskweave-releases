# Providers

A provider is a local coding CLI that TaskWeave runs jobs through. This release ships one,
**Claude Code**. The provider layer is provider-neutral, so further CLIs (Codex, Grok Build,
Gemini and others) are added as adapters without changing how jobs, permission modes, settings,
Health or the viewer work. Nothing here changes Claude's flags, environment, job fields or
stored jobs.

## Contract

A provider adapter owns:

- its spec (`ProviderSpec`): id, display name, how to find its CLI, the environment variables it
  needs, and which job choices its CLI has (effort levels, cost cap, fallback model);
- capability discovery (`descriptor()`), including the permission modes it enforces;
- CLI probes: version, required flags, sign-in state (used by job preflight and by Health);
- validation of model and provider-specific options;
- safe argv and child-environment construction;
- parsing provider stdout into a normalized report/usage envelope.

`JobManager` continues to own queuing, concurrency, persistence, restart
recovery, timeouts, cancellation, process-tree termination, output limits,
attachments, artifacts, and result retrieval. Provider-normalized session IDs
are stored as provider-neutral `session_id`; Claude keeps the legacy
`claude_session_id` alias as well. Each job records the CLI's version as
`provider_version` (and, as before, `cli_version`).

```text
MCP request (provider omitted: the server's default provider)
  -> JobManager validation (mode, prompt, limits, attachment bytes)
  -> ProviderRegistry (enabled providers only) -> adapter.validate()
  -> provider's model policy: model, effort, cost cap
  -> adapter.build_launch() -> generic runner.spawn()
  -> adapter.parse_output() -> JobManager finalization
```

`providers/catalog.py` lists the adapters this release knows. Configuration validates against it,
and the registry creates adapters only for the providers a server enables.

## Enabling and configuring providers

| Setting | Meaning |
|---|---|
| `TASKWEAVE_PROVIDERS` | Providers this server runs, comma-separated (default: `claude`). A provider's CLI is required only when it is listed here. |
| `TASKWEAVE_PROVIDER` | Provider for jobs whose client does not name one (default: the first listed). |
| `TASKWEAVE_<ID>_BIN` | The provider's CLI (default: found on `PATH`). Claude: `TASKWEAVE_CLAUDE_BIN`, as before. |
| `TASKWEAVE_<ID>_PASS_ENV` | Variables passed to that provider's jobs despite scrubbing, typically its API key. Claude: `TASKWEAVE_PASS_ENV`, as before. |
| `TASKWEAVE_<ID>_MODELS`, `_DEFAULT_MODEL`, `_EFFORTS`, `_DEFAULT_EFFORT`, `_MAX_COST_USD`, `_FALLBACK_MODEL` | That provider's model policy, with the same meaning as Claude's `TASKWEAVE_MODELS` and so on (which stay Claude's). |
| `TASKWEAVE_<ID>_ALLOW_UNCAPPED` | `true` lets jobs run although a cost cap is set and the CLI cannot enforce it (see below). |

`<ID>` is the provider id in capitals, for example `TASKWEAVE_CODEX_BIN`. These names are new and
have no `CLAUDE_BRIDGE_` alias. Settings profiles hold the same values (`providers`, `provider`,
`<id>_bin`, `<id>_models`, ...), and the control center's Settings page shows a group per provider.

A policy is checked against what the CLI can do, at startup and before each job:

- **Effort** levels outside the provider's own levels are rejected (a CLI without an effort
  setting accepts none).
- **Cost cap:** if a cap is set and the CLI cannot stop a job at it, startup fails and jobs are
  refused, so a cap is never silently ignored. Setting `TASKWEAVE_<ID>_ALLOW_UNCAPPED=true` accepts
  uncapped jobs; they record `max_cost_usd: null`.
- **Fallback model** is rejected for a CLI that has none.

`list_providers` reports each enabled provider with its own `job_choices` and marks the default.

## Environment isolation between providers

Each provider declares the non-secret variables its CLI needs (Claude: `ANTHROPIC_*`,
`CLAUDE_CODE_*`). A job receives its own provider's variables and never another provider's; for
example a Claude job does not see a Codex configuration variable, and a Codex job does not see
`ANTHROPIC_*`. Credentials are still scrubbed for every provider and reach a job only when listed in
that provider's `PASS_ENV` setting. TaskWeave's own settings never reach any job.

## Starting a CLI on Windows

TaskWeave never runs a CLI through `cmd.exe`. A provider whose CLI is installed with npm (its
`PATH` entry is a `.cmd` shim) may set `node_entry` in its spec: the shim is then read, and the
script it would start is run as `node <script>`. The script must be a file inside the shim's own
folder; otherwise the CLI is refused with a message to point the setting at the executable or
`.js` entry point. Claude does not allow this, because the native `claude.exe` exists.

## Adding a provider

1. **Spike.** Install the CLI, save its `--help`, and check headless mode, output format, and
   which controls can enforce each permission mode (see the outcome table in
   [permission-modes.md](permission-modes.md)).
2. **Adapter** in `src/taskweave_mcp/providers/<id>.py` with its `ProviderSpec`, registered in
   `providers/catalog.py`. Offer only the modes it enforces.
3. **Stub CLI** in `tests/` that emits the CLI's real output format, and tests for the command line,
   output parsing and failure mapping. Add the adapter to the contract tests in
   `tests/test_provider_contract.py`.
4. **Evidence:** `python scripts/verify_permissions.py --provider <id>` with the real CLI, on
   Windows at least. A mode is offered only after it passes.
5. **Docs:** `docs/providers/<id>.md` (flags, evidence, differences from the outcome table) and a
   row in the README.

## Current capability schema

`list_providers` returns the provider descriptor, implemented capabilities, and
status. Capabilities are intentionally conservative: they describe what this
release implements and tests, not everything the upstream CLI documents.

Each provider also reports `effort_levels`, `enforces_cost_cap`, `fallback_model` and
`read_only_data_dirs`. A new provider advertises `read_only_data_dirs` only after its CLI is shown
to let `read_only` jobs read the operator's data folders (`TASKWEAVE_DATA_DIRS`) and nothing else
outside the workspace; without it, its jobs get no data folders.

Current Claude capabilities:

- text input: yes
- attachments: validation/staging foundation exists, but **not enabled** for
  Claude in this release; attachment requests are rejected before job creation
- direct image input: no
- document/PDF input: no
- direct video input: no
- artifact output: yes, text files only. Claude returns each requested file inline in
  its report between `<taskweave-artifact filename="...">` and `</taskweave-artifact>`
  lines; after the run the server saves it under the job-local `artifacts/` directory
  (see "Files Claude returns" in the README). Claude itself never writes there.
- artifact retrieval: yes (report plus files already staged under the job-local `artifacts/` directory)
- permission modes: `read_only`, `read_write`, `full_repository`
- read-only data folders: yes, for `read_only` jobs only (`--add-dir`; see "Read-only data folders" in the README)

Provider/model selection and provider-specific options are additive. Existing
clients that omit `provider` get the server's default provider (Claude unless configured otherwise). Jobs without provider
metadata (or with an empty value) are treated as Claude jobs for backward
compatibility. A recovered job with an unknown, non-empty provider is failed
with `unknown_provider_after_restart`; its output is never parsed as Claude.

Model values are validated before job creation (non-empty, at most 200
characters, no leading/trailing whitespace, no control characters). Claude
rejects any non-empty `provider_options`; an empty object is accepted.

The operator's model policy for Claude (`TASKWEAVE_MODELS`, `TASKWEAVE_DEFAULT_MODEL`,
`TASKWEAVE_EFFORTS`, `TASKWEAVE_DEFAULT_EFFORT`, `TASKWEAVE_MAX_COST_USD`,
`TASKWEAVE_FALLBACK_MODEL`, or the same fields in a settings profile; other providers use
their `TASKWEAVE_<ID>_*` equivalents) is applied
by the job manager before the provider sees the request: a `model` or `effort`
outside the allowed lists is rejected, and defaults fill in what the client did
not choose. `list_providers` reports the result under `job_choices`. The Claude
adapter passes the values as `--model`, `--effort`, `--max-budget-usd` and
`--fallback-model`.

## Planned sequence

1. Artifact retrieval (this batch).
2. Safe attachment staging (this batch).
3. Verified Claude image/PDF support.
4. Provider-neutral groundwork: catalog, per-provider settings, environment isolation, Health
   and the contract tests (this batch).
5. Codex, Grok Build and Gemini CLI adapters, each with real-CLI permission evidence; then
   verified image/PDF input where the CLI supports it.
6. Diagram/chart artifact workflows.
7. Video frame extraction only if a real need justifies it.

## Safety boundaries

- Attachments are accepted as base64 bytes only; client filesystem paths are
  rejected. Canonical bytes live under the job directory.
- Only text/Markdown/CSV/PDF/PNG/JPEG/WebP/GIF are accepted, with signature
  validation and strict size/count limits.
- No provider-visible workspace copies are created in this batch.
- Only `jobs_dir/<job_id>/artifacts/` and the report are scanned for artifacts.
  Files a provider writes elsewhere in the workspace are not copied into the
  artifact directory in this batch.
- Symlinks and Windows junctions are rejected at the `artifacts/` directory and
  during the walk. The scanner never hashes or retrieves a path that resolves
  outside the job-local artifact directory.
- Artifact retrieval never exposes absolute host paths. The response budget
  (`TASKWEAVE_MAX_INLINE_ARTIFACT_RESPONSE_BYTES`) bounds the actual
  returned JSON, including Base64 expansion and metadata. Artifacts over the raw
  or response limit return the `too_large` shape with `content: null`; they are
  not silently truncated. There is no HTTP download endpoint in this batch.
- HTML, XHTML and SVG are returned as inert plain text or download-only Base64
  with `trusted_renderable: false` and an explicit message; the server does not
  render them. This does not make the content safe.
- CSV bytes are never modified. Risk detection conservatively checks comma-,
  semicolon- and tab-delimited forms; when a cell begins with a spreadsheet
  formula prefix, the response sets `csv_formula_risk: true` and warns the
  client to neutralize the file before opening it in a spreadsheet. This is a
  warning, not sanitization.
- Text artifact retrieval applies best-effort redaction to the returned text and
  sets `redacted` accordingly. Base64 retrieval returns the original artifact
  bytes encoded as Base64 and does not apply text redaction to the payload.
  CSV formula-risk metadata is advisory detection, not sanitization. Consumers
  must treat all retrieved artifacts as untrusted content.
- Report text returned through MCP is best-effort redacted. Redaction is not a
  guarantee that arbitrary secrets are removed.

## Codex CLI

The Codex CLI adapter ships in this release but offers no permission mode until `scripts/verify_permissions.py --provider codex --candidate` passes with the real CLI. Its flags, mode mapping and known gaps are in [providers/codex.md](providers/codex.md).
