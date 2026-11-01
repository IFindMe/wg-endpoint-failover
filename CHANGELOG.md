# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

While the major version is `0`, the public interface may change in a minor release.
Breaking changes are always called out explicitly under `Changed` or `Removed`.

## [Unreleased]

### Added

- Repository foundation: license, contribution and security policy, issue and pull request
  templates, editor and ignore configuration.
- Research: upstream WireGuard endpoint semantics, per-platform endpoint-change
  capabilities, and prior art (`contrib/reresolve-dns`, wgrtc, PrivyCS).
- Architecture: component overview, endpoint manager, health monitoring state machine,
  companion configuration schema, security model, failure-mode analysis, Android
  integration strategy, and architecture decision records.
- Planning: implementation plan, release and CI/CD plan, roadmap.
- Mermaid diagram sources under `design/diagrams/`.

Nothing is implemented. See [README.md](README.md#platform-status).

[Unreleased]: https://github.com/IFindMe/wg-endpoint-failover/commits/main
