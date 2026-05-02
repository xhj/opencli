# C4R Downstream Patch Line Management

Date: 2026-05-02
Status: Initial operating model
Scope: How C4R maintains OpenCLI fork changes that are unlikely to be accepted upstream

Companion spec: [2026-05-02-c4r-government-safe-browser-substrate-retrofit-design.md](2026-05-02-c4r-government-safe-browser-substrate-retrofit-design.md)

## 1. Purpose

C4R needs a long-lived OpenCLI downstream patch line for a narrow government-site browser substrate use case. Upstream acceptance is expected to be unlikely because the requirements are specialized. The maintenance goal is not to turn OpenCLI into a C4R-only codebase; it is to keep C4R-specific substrate changes auditable, replayable, and easy to drop if a better external tool replaces them.

The user-facing value that must survive this process is C4R's ability to legally acquire public government-site content through a browser path that produces stable evidence: dedicated profile, explicit ready signal, isolated/post-challenge extraction, tool-level HTTP capture, scoped recovery, fresh context/page, and bounded pacing.

## 2. Repository Model

Use two repositories:

- `Crawl4Research`: owns C4R runtime, site manifests, critical-site OpenCLI plugin, evidence fixtures, and the submodule pin.
- `opencli` fork: owns OpenCLI browser substrate changes required by C4R.

Do not copy OpenCLI source into Crawl4Research. Do not implement government-safe profile/context/CDP primitives in the C4R plugin layer; those are host substrate responsibilities.

Local paths used during this handoff:

- C4R repo: `/Users/xhj/Developer/Crawl4Research`
- OpenCLI fork clone: `/Users/xhj/Developer/opencli`

Expected public fork:

- `github.com/xhj/opencli.git`

The current local clone may use another writable remote or mirror. Before pushing/pinning for another machine, ensure the pinned commit is fetchable from the intended fork URL.

## 3. Branching Policy

Use one long-lived downstream branch for the C4R substrate patch line:

```bash
c4r/gov-browser-v1
```

Use short-lived preparation/document branches only for staging specs or reviewable planning work:

```bash
c4r/gov-browser-substrate-docs
```

The long-lived branch should contain only government-safe substrate changes and their tests. It must not contain:

- C4R Python runtime code.
- C4R site manifests.
- `opencli_plugins/critical-sites/` adapter code.
- Real government-site smoke artifacts that belong in C4R evidence fixtures.
- Broad OpenCLI product rewrites unrelated to the government-safe contract.

## 4. Patch Queue Shape

Maintain the downstream branch as a small, mostly linear patch queue over a known upstream base. Prefer rebase over merge for upstream refreshes because C4R needs to inspect the delta from upstream clearly.

Recommended commit slices:

1. `feat(c4r): expose government browser capability contract`
2. `feat(c4r): add dedicated browser profile registry`
3. `feat(c4r): fail closed for government network capture`
4. `feat(c4r): add government page ready signals`
5. `feat(c4r): add isolated and post-challenge extraction`
6. `feat(c4r): add scoped cookie and storage recovery`
7. `feat(c4r): add fresh context recovery evidence`
8. `feat(c4r): add bounded pacing evidence`
9. `test(c4r): cover government acquire envelope`

Use `c4r` in commit scope or subject so downstream-only intent remains obvious in OpenCLI history.

## 5. Crawl4Research Pin Contract

Crawl4Research must pin a concrete OpenCLI commit, not a floating branch.

The C4R submodule path remains:

```text
upstream/OpenCLI
```

The C4R lock file remains:

```text
upstream/opencli-host-lock.json
```

When switching C4R to the forked patch line, extend or update the lock file with enough information to reconstruct provenance:

```json
{
  "path": "upstream/OpenCLI",
  "repo_url": "https://github.com/xhj/opencli.git",
  "upstream_repo_url": "https://github.com/jackwener/OpenCLI.git",
  "upstream_base_sha": "<sha-or-tag>",
  "downstream_sha": "<pinned-c4r-sha>",
  "patch_line": "c4r/gov-browser-v1",
  "government_safe_browser_contract": "c4r.opencli-gov-browser.v1",
  "package_version": "<package-json-version>",
  "dirty": false,
  "recorded_at": "<UTC timestamp>"
}
```

If C4R keeps the existing `git_sha` key for compatibility, set it to the downstream pinned commit and add `upstream_base_sha` separately.

## 6. Upstream Refresh Procedure

Use this procedure when updating the downstream patch line onto a newer upstream OpenCLI release or commit.

1. In `/Users/xhj/Developer/opencli`, verify the worktree is clean.
2. Fetch the original upstream remote and the C4R fork remote.
3. Check out `c4r/gov-browser-v1`.
4. Rebase the patch line onto the selected upstream tag/SHA.
5. Resolve conflicts without weakening the government-safe contract.
6. Run OpenCLI focused tests for changed browser substrate code.
7. Run the C4R government-safe contract tests from the C4R repo against this OpenCLI checkout or submodule pin.
8. Push the rebased downstream branch to the C4R fork.
9. Update the C4R submodule SHA to the new downstream commit.
10. Update `upstream/opencli-host-lock.json`.
11. Commit the C4R pin bump as a separate C4R commit.

Do not update the C4R submodule pin unless the new downstream commit is pushed and fetchable from the recorded fork URL.

## 7. Verification Boundary

OpenCLI fork verification should prove substrate behavior:

- capability contract schema
- profile id registry behavior
- no `workspace` as profile substitute
- tool-level network capture fail-closed path
- no page-context wrapper in government acquire
- ready signal hit/timeout behavior
- isolated/post-challenge extraction behavior
- scoped cookie/storage facts and reset controls
- fresh context/page evidence
- evidence envelope and exit semantics

C4R verification should prove integration behavior:

- C4R invokes the repo-local pinned OpenCLI host.
- C4R rejects missing or mismatched government-safe capability contract.
- C4R reads evidence envelope and frozen manifests.
- C4R keeps challenge/manual-session interpretation outside OpenCLI.
- C4R site/plugin fixtures match the pinned OpenCLI contract.

Real government-site contrast evidence belongs in C4R acceptance fixtures. It is not a substitute for OpenCLI deterministic tests.

## 8. Remote Hygiene

Recommended remote layout in `/Users/xhj/Developer/opencli`:

```bash
origin   git@github.com:xhj/opencli.git
upstream https://github.com/jackwener/OpenCLI.git
```

If a private mirror is used for `origin`, add a second remote for the GitHub fork before recording a C4R submodule pin that points at GitHub:

```bash
git remote add github git@github.com:xhj/opencli.git
```

The invariant is simple: every C4R-pinned downstream SHA must be fetchable by a fresh clone using the URL recorded in C4R's submodule/lock metadata.

## 9. Stop Conditions

Stop and reassess the OpenCLI substrate route if any of these becomes true:

- A real or auditable equivalent fresh context/page primitive cannot be provided by the extension/daemon architecture.
- Tool-level network capture cannot be made fail-closed without breaking the Browser Bridge model.
- Isolation cannot be proven, and the only viable extraction path is main-world injection before ready.
- The downstream patch queue grows into broad OpenCLI product rewrites unrelated to the C4R government-safe contract.
- Maintaining the fork costs more than replacing OpenCLI with another external browser substrate that satisfies the same contract.

The fallback decision is not to weaken the C4R runtime target. It is to replace the external tool while keeping the government-safe evidence contract intact.
