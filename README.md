# Teamwork for DeepSeek Harness

A portable teamwork orchestration runtime, with DeepSeek Harness (DSH) as its first integration.

An independent orchestration core for multiple Harnesses, with DSH as the first integration. The public release is **0.2.0-dev.1 prerelease**; the current worktree is the unreleased **0.3.0-dev.1**, which adds bounded repair rounds, pause/resume, and candidate verification queue recovery. It is not a complete product.

This project is an independent implementation. It is not an official product of DeepSeek or Google, and it does not contain their private runtime samples or internal prompts.

## Current capabilities

- Host tools: `teamwork_start`, `teamwork_status`, `teamwork_control` (cancel; 0.3 also supports pause, resume, and explicit requirement revision).
- 0.3's `teamwork_inspect` provides digest-verified paginated artifact reads, change manifests, and a read-only three-way integration conflict preview against the current project.
- Independent Runtime: SQLite state, idempotent commands, dispatch outbox, and replayable SSE events.
- 0.3 provides a `teamwork-runtime start` persistent background mode: it continues authorized work after the launching terminal and DSH entry disconnect. The default `run` is attached mode and pauses when the owner exits; `status` / `stop` use independent operator credentials and an instance ID, with no PID guessing or force-killing of the service.
- Each Attempt uses an independent working copy, DSH SDK process, and scoped credentials; timeout and cancellation are supported.
- Optional verification: candidate content digest → fresh read-only review → independent acceptance command → deterministic Gate.
- 0.3's start supports structured requirements and file/directory change scope; the review answers each requirement individually, and the Runtime checks scope and rejects out-of-bounds candidates.
- `revise` preserves prior-version evidence, freezes the current project as the new baseline, revokes the old Gate, and expires stopped derived work; the new version is re-implemented, re-reviewed, and re-accepted.
- The optional `budget.maxModelAttempts` is a total allowance across implementation, review, repair, resume, requirement revision, and derived tasks, not per-step approval; dispatch is automatic within the reserved allowance and only pauses once exhausted. It is not a token or monetary limit.
- Without authorized automatic integration, the original project is not modified. `verified` means the candidate passed the current acceptance policy, not that it was integrated.
- 0.3 can explicitly enable `teamwork_integrate`: preview and authorize first, then serial write-back, retain backups, and accept the final merged tree; cancellation and conditional "keep current files" handling are supported.
- Provide `autonomy: {"integration":"on-gate-pass"}` and an explicit spec at creation to authorize once and automatically complete Gate → preflight → serial write-back → final acceptance, without a second integration approval. Further provide `conflicts:"resolve"` and a shared total budget to automatically create conflict-resolution subtasks and re-review, re-accept, and write back.
- The `scope: "workflow"` of `teamwork_status` / `teamwork_control` summarizes and controls the entire derivation chain on the root task. Use the aggregate revision to pause, resume, or cancel all related tasks and integration dispatch in one operation; no need to operate on subtasks one by one. This is optional manual control, not an approval step during autonomous execution.
- Conflicts can create a new work item via `teamwork_integrate`'s `resolve`: using the current project as the baseline, read-only viewing of the three-way inputs, re-implementation, independent review, and acceptance; when an accepted automatic write-back policy exists it integrates automatically after passing, otherwise an explicit integration command is still used.

The 0.3 worktree also supports operator-configured 1–5 bounded repair rounds with per-round evidence retention; only one round is executed by default. Pause offers drain / interrupt; resume uses a new session rather than reconnecting to the old process. Integration is off by default and requires `integration.enabled: true` plus authorization via a creation-time policy or an explicit command; see the example in `examples/runtime.integration.example.json`. Not yet implemented: full RunSpec, strong-evidence reconciliation of unknown processes, bulk artifact export, slash commands, and OpenCode/Pi adapters. For the full development direction see [ROADMAP](ROADMAP.md). The subsequent adapter order is DSH → OpenCode → Pi.

## Using from source

Requires Node.js >=22.13; the current test baseline is Windows / Node.js 22.22.1.

```sh
git clone https://github.com/LING71671/teamwork-dsh.git
cd teamwork-dsh
git checkout v0.2.0-dev.1
npm ci --registry=https://registry.npmjs.org
npm test
```

`v0.2.0-dev.1` has 35 tests; the current 0.3 worktree has 243. `npm test` builds the source and runs the tests, including an offline model adapter for the real DSH SDK/Cordis, per-requirement review and change scope, requirement revision and stale-evidence invalidation, independent verification after three-way conflict resolution, explicit integration into a temporary project, preservation of user changes, final acceptance, and process-exit recovery; no API key is required, and it does not validate real-model task effectiveness.

Edit the absolute paths and DSH model routing in `examples/runtime.example.json`, and adjust `examples/host.cordis.patch.yml` to your actual locations. The example paths are placeholders only and will not replace your credentials.

```sh
npm start -- --config examples/runtime.example.json --doctor
npm start -- --config examples/runtime.example.json
```

In another terminal, set `TEAMWORK_CONNECTION_FILE` to the `connection.json` generated by the Runtime, then start the DSH host using the example patch. For full configuration, acceptance examples, protocol, and limitations see the [development guide](DEVELOPMENT.md). When you actually launch a task, DSH will use the model service you configured, which may incur charges.

## Release packages

[Releases](https://github.com/LING71671/teamwork-dsh/releases) provide a compiled npm-format `.tgz` and a SHA-256 checksum file. You can run `npm install /absolute/path/teamwork-dsh-plugin-0.2.0-dev.1.tgz` in a standalone installation directory, then use `npx teamwork-runtime --config /absolute/path/runtime.json`. The host must be able to resolve the installed `@teamwork/dsh-plugin/host`, or use its absolute `file://` module URL.

It is not currently published to the npm registry; `private: true` prevents accidental publication and does not affect local tarball installation. Both the source and the compiled package use the [MIT license](LICENSE).

## Security and boundaries

Working copies are not an operating-system sandbox. Use only for cooperative tasks; there is currently no guarantee of recovering escaped child processes, and no defense against malicious processes under the same OS user. Review tool permissions are limited, but the implementer shell and acceptance programs still require operator trust.

Do not commit `connection.json`, environment credentials, or runtime state. A dispatched process that cannot prove exit after an interruption enters `blocked` and is not automatically redispatched. Please read [Recovery and boundaries](DEVELOPMENT.md#recovery-and-boundaries) first.

## Development and contributing

The core is in `contracts.ts` / `kernel.ts` / `store.ts` / `runtime.ts`; DSH is concentrated in `driver-dsh.ts` and `plugin-dsh/`. Run `npm test` before committing changes; protocol changes should update the client, tests, and documentation in sync.

Issues and Pull Requests are welcome. Please include the version, reproduction steps, and redacted logs; do not upload tokens, real session contents, or private project files. For the version history see [CHANGELOG](CHANGELOG.md).
