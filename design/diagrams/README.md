# Diagrams

Mermaid sources. Each diagram is rendered by embedding the same Mermaid block in the
architectural document it belongs to, so the diagram and its explanation stay together.

| Source | Used by | Depicts |
|---|---|---|
| [`architecture-components.mmd`](architecture-components.mmd) | [overview.md](../../docs/architecture/overview.md) | Platform-independent core behind a one-method adapter |
| [`failover-sequence.mmd`](failover-sequence.mmd) | [overview.md](../../docs/architecture/overview.md) | One health-check cycle, including the commit-probe-observe path |
| [`failover-state-machine.mmd`](failover-state-machine.mmd) | [health-monitoring.md](../../docs/architecture/health-monitoring.md) | The failover state machine with transitions and guards |

## Conventions

- **The `.mmd` file is the source of truth.** When a diagram changes, update the `.mmd` file
  and the embedded copy in the document in the same pull request.
- No diagram is generated from code. Generated diagrams drift; hand-maintained ones with a
  named owner do not.
- Diagrams describe the design, not the implementation. A diagram that disagrees with the
  code is either stale or a bug in the code — resolve which, deliberately.

## Adding a diagram

1. Create `<topic>.mmd` here.
2. Embed the identical Mermaid block in the document that discusses the topic.
3. Add a row to the table above.

## Validating

GitHub renders Mermaid in Markdown natively. To validate a `.mmd` file locally, install
`@mermaid-js/mermaid-cli` and render it:

```bash
mmdc -i design/diagrams/failover-state-machine.mmd -o /tmp/state.svg
```

Rendering is not currently part of CI; it is a manual step when a diagram changes. Adding it
to CI is a reasonable milestone-1 task, since a broken diagram is a broken architecture
document.
