# TG Dental HyperFrames security hardening

Status as of **2026-10-30**. This report concerns the public fork
`thinhnguyenjr1508/hyperframes`, branch `tg-dental/security-hardening`.
The upstream repository and this fork's `main` branch are intentionally unchanged.

## Scope and evidence

This is a **targeted source and dependency review**, not a comprehensive
penetration test. No malicious code or active data theft was demonstrated.
Findings below are based on repository source, published package metadata,
and public upstream security advisories. Exploitability in this application
has **not** been reproduced.

| Area | Previous behavior / version | Change on this branch | Evidence |
| --- | --- | --- | --- |
| CLI and Engine HTTP dependency | `undici@7.25.0` | Pin `7.30.1` in both manifests and Bun lockfile | `GHSA-xjj6-f33q-929c`, `GHSA-qrxm-fqhp-jwvm` |
| Test / SDK DOM dependency | `happy-dom@20.9.0` | Require `^21.0.4` and lock `21.0.4` | `GHSA-c8p4-j97h-3vg8`, `GHSA-834v-8h66-3h79` |
| CLI usage telemetry | Enabled on a new install | Disabled until an explicit boolean `telemetryEnabled: true` | `packages/cli/src/telemetry/config.ts` |
| Media-use telemetry | Sent without a shared opt-out record | Fail closed without explicit enabled preference | `skills/media-use/scripts/lib/telemetry.mjs` |
| Queue delivery | Exit-time child could send previously queued events | Check current config / env at both async and exit-time delivery | `packages/cli/src/telemetry/transport.ts` |

**Important:** Analytics sent after explicit opt-in may still include system
metadata and a HeyGen account-linked identifier if signed in. This is
intentional upstream behavior and must be considered before opting in.

The upstream published npm package (`npx hyperframes`) is **not** changed by
this fork. Users must install and run the audited fork or set the privacy
environment variables in any upstream installation.

## Regression coverage added

- Invalid or string-valued `telemetryEnabled` is not accepted as consent.
- Media-use is disabled without a config or with corrupted config.
- Media-use accepts affirmative opt-out spellings (`1`, `true`, `yes`, `on`).
- Queued CLI events are dropped if telemetry is disabled before delivery,
  including the detached exit-time sender.

## Verification performed

- [x] Re-read and compared all 12 modified files on the target GitHub branch.
- [x] Checked new `undici` and `happy-dom` version references and Bun lock
      integrity entries, cross-checking package integrity with npm registry metadata.
- [x] Parsed the Bun lockfile structure (JSONC trailing-comma normalization)
      and verified workspace/package version consistency.
- [x] `node --input-type=module --check` passed for media-use telemetry script
      and its test source (syntax check only).
- [x] Compared `main` and the security branch: only scoped files changed.
- [ ] Full Bun dependency resolution with `--frozen-lockfile`: **not run**.
- [ ] TypeScript typecheck, package builds, test suites, render smoke test: **not run**.
- [ ] GitHub Actions CI: no status checks reported on the security commit.

The execution environment used for this review had Node.js, but no Bun,
and local filesystem writes were blocked. Therefore **do not merge this
branch or deploy it to production until the following checks pass**.

## Validation commands for a trusted writable checkout

Run in a disposable environment without production credentials or patient data:

```bash
bun --version
bun install --frozen-lockfile
bun run build
bun run lint
bun run test:skills
bun run --cwd packages/cli test src/telemetry/config.test.ts
bun run --cwd packages/cli test src/telemetry/client.test.ts
bun run test:scripts
```

Inspect any failing test before correcting the lockfile or dependency graph.
`happy-dom` is a major-version upgrade and may require compatibility fixes
in test or SDK code. Then exercise at least one local render with synthetic
media only. Run `hyperframes telemetry status` and confirm that a new
installation says disabled; when intentionally enabled, verify the opt-out
overrides prevent PostHog requests.

## Release decision

**HOLD — not production-ready.** Code changes and static validation are
recorded, but required Bun/build/runtime tests are still outstanding.
No merge into `main`, upstream pull request, deployment, publication, or
installation of the fork is authorized by this report.
