# Release plan and CI/CD architecture

## 1. Guiding rule

> A workflow is added when the build system it drives exists.

This repository currently contains documentation only. Shipping a "build the Windows MSI"
workflow before there is a Windows build system produces a red badge, teaches contributors
that CI is ignorable, and — worse — invites the reader to believe a platform is supported when
it is not. Workflows below are therefore split into **implemented now**, **added with the
milestone that needs them**, and **explicitly not planned**.

## 2. Workflow inventory

| Workflow | File | State | Trigger | Purpose |
|---|---|---|---|---|
| Docs | `docs.yml` | **Implemented now** | PR, push to `main` | Markdown lint (`markdownlint-cli2`) + link check (`lychee`) |
| PR title | `pr-title.yml` | **Implemented now** | PR | Conventional Commits enforcement |
| Go CI | `ci-go.yml` | M1 | PR, push to `main` | build, `go vet`, `golangci-lint`, `govulncheck`, unit tests, amd64+arm64 |
| Integration | `integration-linux.yml` | M3 | PR (label), push to `main`, nightly | network-namespace WireGuard integration suite |
| Debian | `release-debian.yml` | M6 | tag `v*` | build `.deb`, checksum, upload to release |
| Windows | `build-windows.yml` | M4 | PR, push to `main` | build + unit tests on `windows-2025`, amd64 (+ arm64 cross) |
| Windows MSI | `release-msi.yml` | M6 | tag `v*` | build MSI, checksum, upload |
| Android | `build-android.yml` | M5 | PR, push to `main` | Gradle build, unit tests, lint |
| Android release | `release-apk.yml` | M6 | tag `v*` | signed APK/AAB from GitHub Secrets |
| Release orchestration | `release.yml` | M6 | tag `v*` | release notes, checksums, SBOM, provenance |

Only the first two exist today. The rest are specified so that their implementation is a
mechanical step at the right milestone, not a design decision made under time pressure.

## 3. Implemented now

### `docs.yml`

Runs `markdownlint-cli2` over all Markdown, and `lycheeverse/lychee-action` over all links.

The link check is not cosmetic. The research documents are the evidence base for every
architectural claim, and an upstream source link that rots silently converts a sourced claim
into an assertion. A broken evidence link is a documentation defect and blocks the pull
request.

### `pr-title.yml`

Enforces Conventional Commits on the pull request title, which becomes the squashed commit
subject. Scope and type are constrained to the set in `CONTRIBUTING.md`.

Both run on `ubuntu-24.04` with read-only `contents` permission and `concurrency` cancellation
of superseded runs.

## 4. Added at milestone 1 — `ci-go.yml`

```yaml
jobs:
  test:
    strategy:
      matrix:
        goos: [linux]
        goarch: [amd64, arm64]
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: go.mod }
      - run: go build ./...
      - run: go vet ./...
      - run: golangci-lint run
      - run: govulncheck ./...
      - run: go test -race -count=1 ./...
```

Notes:

- `arm64` is exercised by cross-compilation plus `qemu-user-static` for the test run. Native
  arm64 runners are used if available at implementation time.
- `golangci-lint` configuration is committed so local and CI runs agree.
- `govulncheck` is a gate, not a report: a known vulnerability fails the build.

## 5. Added at milestone 3 — `integration-linux.yml`

The integration suite needs the `wireguard` kernel module, network namespaces and `sudo`.

- Gated behind a label on pull requests (`integration`) so that documentation-only changes do
  not pay the cost, and unconditional on `main` and nightly.
- The runner is a standard GitHub-hosted `ubuntu-24.04`; `sudo` is used only for namespace and
  interface setup, never for the daemon under test, which runs unprivileged with
  `CAP_NET_ADMIN`.
- Artifacts uploaded on failure: daemon logs, `wg show dump` before/after, namespace topology.
- Nightly soak job (randomised connectivity toggles, 60 minutes) as a separate job so its
  failures are attributable.

## 6. Added at milestone 4 — `build-windows.yml`

```yaml
runs-on: windows-2025
strategy:
  matrix:
    goarch: [amd64, arm64]
```

- `go build`, `go vet`, unit tests, and a smoke test that starts the service and queries
  status over the named pipe.
- The WireGuardNT spike result from [implementation-plan M4 task 1](implementation-plan.md)
  determines whether an integration test can run in CI at all: it requires the WireGuard for
  Windows driver, which a bare runner does not have. If it cannot, the job builds and unit-tests
  only, and the integration run is recorded as a manual, documented step.

**Signing.** Authenticode signing is configured as a separate job that runs only when the
signing secret is present:

```yaml
- name: Sign
  if: env.WINDOWS_CERT_THUMBPRINT != ''
  run: ./build/sign.ps1
- name: Warn when unsigned
  if: env.WINDOWS_CERT_THUMBPRINT == ''
  run: Write-Warning "Artifact is unsigned: signing credential not configured"
```

The warning is mandatory. A release that silently ships unsigned binaries is worse than one
that says so.

## 7. Added at milestone 5 — `build-android.yml`

- Gradle build, Android Lint, unit tests.
- Debug signing by default; release signing only in `release-apk.yml`.
- `assembleDebug` on pull requests; `assembleRelease` on tags.
- The APK's Go component (`libwg-go.so`) build is the expensive step; cache the Go module and
  Gradle caches keyed on the lockfiles.

## 8. Added at milestone 6 — release workflows

### Trigger

Tags matching `v*`, and only tags. No workflow publishes an artifact from a branch or a pull
request.

### Versioning

[Semantic Versioning 2.0.0](https://semver.org/). While the major version is `0`, breaking
changes may appear in a minor release and are always listed under `Changed`. From `1.0.0`,
the companion configuration schema (`schema_version`), the status JSON, and the CLI surface
are the compatibility surface and follow SemVer strictly.

### Per-platform artifact matrix

| Platform | Artifacts | Signing |
|---|---|---|
| Linux | `.deb` (amd64, arm64), tarball (amd64, arm64, static), `SHA256SUMS` | Checksums only; package manager handles trust |
| Windows | MSI (amd64, arm64), `SHA256SUMS` | Authenticode when configured; explicit warning when not |
| Android | APK, AAB, `SHA256SUMS` | Release keystore from GitHub Secrets |

### Release notes

Generated from `CHANGELOG.md` sections with a conventional-commit summary as a fallback, and
including:

- the commit the artifacts were built from,
- toolchain versions,
- `wgctrl-go` version and, for Windows, the validated WireGuard for Windows version,
- an explicit **platform support** table copied from the README.

The platform support table in release notes is the mechanism that prevents the most dangerous
failure mode of this project: a user reading a release note and believing a platform works
when it has never been tested.

### Provenance

`actions/attest-build-provenance` produces an SLSA provenance statement per artifact, and an
SBOM (`syft`) is attached. Neither is optional: this is a networking control-plane component,
and a consumer needs to be able to verify what is inside it.

## 9. Secrets inventory

| Secret | Used by | Required from | Notes |
|---|---|---|---|
| `ANDROID_KEYSTORE_BASE64` | `release-apk.yml` | M6 | Base64 of the keystore; decoded to a temp file, never written to the workspace |
| `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD` | `release-apk.yml` | M6 | |
| `WINDOWS_CERTIFICATE_BASE64`, `WINDOWS_CERTIFICATE_PASSWORD` | `release-msi.yml` | M6 | Optional; absence produces unsigned artifacts with a warning |
| `GITHUB_TOKEN` | all | — | Scoped to the minimum; never logged |

Rules:

- No secret is written to the workspace outside a temp directory that is deleted in an
  `always()` step.
- Secrets are never echoed. Any step that prints environment variables is forbidden by review.
- The release job does not run on pull requests, so a malicious PR cannot exfiltrate a secret
  by modifying a workflow and observing it on a fork.

## 10. Branch protection and repository settings

Enabled on `main` after the planning-phase push; extended at each milestone as new jobs land:

| Setting | Value |
|---|---|
| Require pull request before merging | Yes, no approval count — single-maintainer repo; add one required approval when a second maintainer exists |
| Require status checks | `markdownlint`, `links`, `lint`; plus the Go CI jobs from M1 |
| Require linear history | Yes (squash merge) |
| Dismiss stale approvals on new commits | Yes |
| Require signed commits | No — would block contributors for no security gain here |
| Restrict force pushes | Yes |
| Restrict deletion | Yes |
| Admins bound to these rules (`enforce_admins`) | Yes — no bypass, including for force push and deletion |
| Private vulnerability reporting | Enable in Settings → General — the API surface available here does not expose it |
| Dependabot alerts and security updates | At M1, with the Go module (`.github/dependabot.yml`) |
| Code scanning (CodeQL) | Enabled when Go code exists (M1) |

`PR title` runs via `pull_request_target` with `permissions: pull-requests: read` only, so it
is safe to require and cannot be used to execute untrusted code.

## 11. Not planned

| Workflow | Why not |
|---|---|
| Publishing to a distribution repository (PPA, winget, F-Droid) | A separate decision with its own trust model; revisit after 1.0.0 |
| Automatic release on every merge | Pre-1.0 releases should be deliberate |
| Windows cross-compilation from Linux for the MSI | MSI authoring needs Windows tooling; `windows-2025` runners are sufficient |
| A "supported platforms" badge | It would imply support before implementation |

## 12. Summary of what ships when

| Milestone | CI that exists afterwards |
|---|---|
| Now | Docs lint, link check, PR title |
| M1 | + Go build/vet/lint/vuln/test on amd64+arm64 |
| M3 | + Linux network-namespace integration suite, nightly soak |
| M4 | + Windows build, unit tests, service smoke test |
| M5 | + Android Gradle build, Android Lint, unit tests |
| M6 | + Tag-triggered Debian, MSI, APK/AAB releases with checksums, signing, SBOM, provenance |
