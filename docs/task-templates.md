# Task templates

Ready-to-use prompts for common jobs. Copy one, change the parts in *italics*, and send it from Claude Desktop, ChatGPT or any other MCP client. Each template names the permission mode it needs; the server operator's ceiling (`TASKWEAVE_MAX_PERMISSION_MODE`) must allow it. Read-only, the default, is enough for most of them.

**How to send one.** In your client, write: *Use TaskWeave to start a task with permission_mode read_only and this prompt:* and paste the template. The client gets a `job_id` back in seconds; ask it later to check the task and show the result. Files the job returns (CSV, Markdown, JSON, diagrams) appear under the result's artifacts. To follow up, ask for another task in the same thread; add *carry the previous report's context* when the follow-up needs it.

**What a job can see.** Only the workspace folder, or a working folder the operator named, plus read-only data folders and the files you attach to a read-only job. A read-only job cannot run programs, so it reads code and data rather than executing it: give it small data files, and use `full_repository` when a task needs tests or git.

## Understand and review

### 1. Tour of a codebase

**Mode:** `read_only` · **Returns:** a report and `architecture.mmd`

> Read this repository and write an orientation for a developer who is new to it: what it does, how it is organised (the main folders and what each is for), how a typical request or command flows through the code, how it is built and tested (from the files; do not run anything), and the five files to read first and why. Return the architecture as a Mermaid diagram in a file named architecture.mmd.

### 2. Bug review of one area

**Mode:** `read_only` · **Returns:** a ranked list of findings

> Review the code under *src/payments* for correctness bugs: wrong logic, unhandled errors, unsafe handling of input, race conditions and resource leaks. For each finding give the file and line, a concrete input or sequence that makes it go wrong, what happens then, and a suggested fix. Rank them by severity. Leave out style and naming comments. Say which files you read and which you skipped.

### 3. Security pass

**Mode:** `read_only` · **Returns:** a report and `findings.csv`

> Look through this repository for security problems: secrets or keys committed to files, injection (SQL, shell, path), unsafe deserialization, missing authentication or authorization checks, and dependencies pinned to versions with known problems (from the lock files). For each, show the evidence (file and line) and rate it high, medium or low. Return the list as findings.csv with columns severity,file,line,issue,fix. Do not include the value of any secret you find, only where it is.

### 4. Review my uncommitted changes

**Mode:** `full_repository` (it runs `git diff`) · **Returns:** a review

> Run git status and git diff to see the changes that are not committed yet. Review them as a careful reviewer would: bugs, missing tests, edge cases, and anything that breaks existing callers. Quote the lines you comment on. Do not change any file and do not commit.

## Documents and data

### 5. Summarize attached documents

**Mode:** `read_only` with attachments (PDF, text, Markdown, CSV or images) · **Returns:** a one-page summary

From ChatGPT, or any client that cannot pass the files itself, ask it to start the task and wait for you to upload the documents; then upload them in the control center when it opens. See [Files you upload in the control center](reference.md#files-you-upload-in-the-control-center).

> Read the attached documents and write a one-page summary: the decisions made, the figures that matter, open questions, and any place where the documents contradict each other. Give the document and page (or section) for every point.

### 6. Compare two versions of a document

**Mode:** `read_only` · **Returns:** `changes.md`

> Compare *policy-2025.md* and *policy-2026.md* in the workspace. List every change in meaning, not in wording: what was added, removed or changed, and who is affected. Return the list as a Markdown table in a file named changes.md with columns section,before,after,impact.

### 7. A question about a small data file

**Mode:** `read_only` · **Returns:** an answer and a CSV

> Using *sales.csv* in the workspace, find which region grew fastest from the first to the last quarter in the file. Explain how you computed it, show the numbers for every region, and return them as growth.csv with columns region,first_quarter,last_quarter,growth_percent. Say if any rows look wrong or are missing.

### 8. Research memo from a folder of notes

**Mode:** `read_only` (with the notes in the workspace or a data folder) · **Returns:** `memo.md`

> Read the notes in the *research* data folder and write a memo on *what our customers say about onboarding*: the main themes with how often each comes up, two or three representative quotes per theme with the file they came from, disagreements between sources, and what is missing from the notes. Return it as memo.md.

## Changes

### 9. Write documentation

**Mode:** `read_write` · **Returns:** a new file in the workspace

> Write *docs/api.md* describing every public function in *src/api/*: what it does, its parameters and return value, the errors it raises, and one short example. Take everything from the code; mark anything you are unsure of with "TODO: check". Do not change any other file.

### 10. Add tests and make them pass

**Mode:** `full_repository` (it runs the test command) · **Returns:** new tests and the test run summary

> Add unit tests for *src/parser.py* that cover normal input, empty input, the largest input it accepts and malformed input. Put them next to the existing tests and follow their style. Run the tests with the project's test command and fix only the tests you added until they pass. If a test reveals a real bug in parser.py, do not fix the bug: describe it in your report.

### 11. Release notes from history

**Mode:** `full_repository` (it runs `git log`) · **Returns:** `release-notes.md`

> Run git log from tag *v1.4.0* to HEAD and write release notes for users, not developers: new features, fixes and anything that needs action when upgrading, in plain language, grouped under those three headings. Leave out internal refactors and test-only changes. Return them as release-notes.md.

## Writing your own

- **Say what you want back.** Name the files ("return it as summary.csv with columns ...") and the shape of the answer (a table, a ranked list, one page).
- **Name the folder or files.** "the code under src/payments" finishes faster and costs less than "the repository".
- **Ask for evidence.** File and line, document and page: you can check them, and the agent is less likely to guess.
- **Pick the lowest mode that works.** Read-only for anything that only reads; `read_write` only to create or edit files; `full_repository` only when tests, builds or git are needed, and only for repositories you trust.
- **Set the budget first.** The per-job cost cap and the daily budget (Settings) stop a task that grows too large.

More on prompts in the reference: [Writing good prompts](reference.md#writing-good-prompts).
