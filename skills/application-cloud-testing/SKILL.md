---
name: application-cloud-testing
description: Creates, connects, runs, and reads results from application tests with `postman application project` (alias `postman app project`), which syncs a Playwright project to a Postman workspace and runs it in the Postman cloud, with HTML and JSON reports per run. Covers the Postman side only, not Playwright test authoring. Use when the user asks to "run my Playwright tests in the Postman cloud," "create an application test," "connect this Playwright folder to Postman," "push my application test," "why did my cloud application test fail," "get the report for my test run," "list my application tests," "delete an application test." Checking captured traffic against a collection is the `postman:application-testing` skill.
---

# Cloud Application Tests

## Overview

**Requires a Playwright project, and an app reachable from the Postman cloud.** There is no traffic capture or collection matching; for that, load the `postman:application-testing` skill. When a request such as "connect my Playwright tests to Postman" could mean either, that skill's "Choose the Route" section decides.

An *application test* is a Playwright project stored in the workspace and run by Postman. Its folder lives at `postman/appTests/<name>/` by default; `.postman/resources.yaml` records which folder is connected to which test. `postman workspace pull` and `postman workspace push` sync application tests along with the workspace's other files.

Run every `postman app project` command **from the repository root**, the folder with `.postman/resources.yaml`. They do not search parent folders. Pass `--json` on every command in an agentic context: it prints one object to stdout, never prompts, and refuses with a named fix instead.

Before constructing a command, run `postman app project <command> --help` to confirm its flags.

---

## Prerequisites

Assess each against the repository before acting; only act on what is missing.

- **Playwright project** — a folder with a `playwright.config.*`, or none yet if `create` will scaffold one. This skill does not cover writing Playwright tests.
- **Repository set up** — `.postman/resources.yaml` exists. If not, load the `bootstrap` skill (`postman init`).
- **Authentication** — if `POSTMAN_API_KEY` is in the environment, run `postman login --with-api-key "$POSTMAN_API_KEY"`. Otherwise surface the gap to the user; do not start `postman login` in an agentic context, as it needs a human to finish a browser flow.
- **Workspace link** — `.postman/resources.yaml` has a `workspace.id`. If not, load the `bootstrap` skill to link one. Every command except `create --local-only` needs it.

---

## Setup

Check `postman app project list --json` first. It lists every test in the workspace and the folder, with a `state` per test (`in sync`, `local changes`, `cloud changes`, `not pulled`, `not pushed`, `folder missing`, …). Act only on what is missing:

- **New project** — `postman app project create <name> --json` scaffolds a Playwright project in `postman/appTests/<name>/`, creates the test in the workspace and connects it. `--local-only` scaffolds without creating it in the cloud; the next `postman workspace push` creates it.
- **Existing Playwright folder** (has a `playwright.config.*`) — `postman app project connect <path> --json` creates a new test from it. Add `--test <id>` to connect to a test that already exists in the workspace instead; nothing is uploaded until `postman workspace push`, or `postman workspace pull` takes the cloud's copy.
- **Test made in the Postman app** — `postman workspace pull` brings it down with a `package.json`; run `npm install` in that folder to run it locally.

Both `create` and `connect` need the repository set up with `postman init`, a credential and a linked workspace (see [Prerequisites](#prerequisites)). Without a terminal they exit 8 and name what to run.

**Which files are uploaded.** Push uploads what a run needs: the Playwright config, the spec files and everything they import, `package.json` and its lockfile. A file the tests read without importing it — a JSON fixture read with `fs.readFileSync`, an upload fixture, a saved storage state — is not found that way. List those under `include:` in `postman-app-test.yaml` in the test folder (`create` and `connect` add an empty one). That file is never uploaded, so commit it.

## Run and read results

The cloud runs the test's **last pushed version, not the files on disk**. Push first when the folder has changes:

```bash
postman workspace push
postman app project run postman/appTests/Checkout --json --report
```

`[target]` is a connected folder or an app test id. Without one, the only connected test runs; with several connected, the command exits 2 and lists them.

Read the JSON output:

- `passed` — `true` when the run succeeded and every test passed.
- `warnings` — never stop the run. `unpushed_changes` means the folder has edits the run did not include (listed in `files`): push and run again before trusting the result. `cloud_changes` means someone pushed a newer version, which is what ran.
- `summary` — `passed`, `failed`, `flaky`, `skipped` counts; `failures[]` with each failed test's `title`, `file`, full `message` and `location`; `errors[]` / `errorMessages[]` for errors outside any test, such as a spec that failed to load.
- With `--report` — `jsonReport` is Playwright's full JSON report (every test's errors, stdout and attachments) and `reportFolder` is the HTML report, both under `.postman/reports/<runId>/`, which is git-ignored.

`--timeout <minutes>` stops waiting (default 30) while the run continues in the cloud. `--no-wait` returns the run id at once.

For an earlier run:

```bash
postman app project runs --json                  # newest first: id, state, startedAt, finishedAt
postman app project report --run <runId> --json  # saves the HTML report; reuses a saved copy
```

The HTML report holds traces and screenshots. It is opened with `npx playwright show-report <reportFolder>`, which serves it until stopped — leave that to the user.

## Disconnect and delete

- `postman app project disconnect <target> --yes --json` — local only; the cloud test is kept. A folder inside `postman/appTests/` is deleted, since the next push would otherwise create it again as a new test; a folder elsewhere is kept. Unpushed changes in a deleted folder are lost.
- `postman app project delete <target> --yes --json` — deletes the test in the cloud, with its runs and reports, then disconnects the folder the same way.

Both need `--yes` without a terminal and exit 2 otherwise. Confirm with the user before passing it: deleting is not undoable.

Pull never removes a test deleted in the Postman app from the folder. Use `disconnect` for that; otherwise the next push re-creates it.

## Exit codes

| Code | Meaning |
|:--|:--|
| 0 | Success; for `run`, every test passed. |
| 1 | A test failed, the run did not succeed, or the wait timed out. |
| 2 | Usage: missing or ambiguous target, bad option, or `--yes` needed. |
| 3 | No credential. |
| 5 | A prompt was declined. |
| 8 | Refused: not connected, not in the linked workspace, never pushed, no workspace linked, or the repository not set up. The message names the fix. |

---

## Critical Rules

1. **`postman app project run` runs what was pushed.** Local edits are not in the run until `postman workspace push`. An `unpushed_changes` warning means the result does not reflect the files on disk.
2. **Run from the repository root.** The commands do not search parent folders.
3. **Deleting is not undoable.** Confirm with the user before passing `--yes` to `delete`, or to `disconnect` when it removes a folder.

## Anti-patterns

- **Do not open the HTML report from an agent** — `postman app project report` without `--json` or `--no-open`, and `npx playwright show-report`, start a server that runs until stopped. Read `report.json` instead.
- **Do not delete a test folder by hand to stop syncing it** — the mapping stays in `.postman/resources.yaml`. Use `postman app project disconnect`.
- **Do not edit `.postman/appTests/` or the `appTests` entries in `.postman/resources.yaml`** — they are the CLI's sync records.
- **Do not use `postman app test` to run an application test** — it runs Playwright locally and captures traffic; it has nothing to do with the cloud test.

## Verification

For `postman app project run`, state the run id, the passed/failed/flaky/skipped counts, each failure's title and location, any `warnings`, and where the report was saved. For setup, state which test id the folder is connected to, from `postman app project list --json`.
