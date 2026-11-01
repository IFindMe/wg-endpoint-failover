## Summary

<!-- One paragraph. What changes, and why. -->

## Milestone / task

<!-- e.g. Milestone 2 — Linux MVP / task 2.3 endpoint adapter -->

Refs #

## Type of change

- [ ] Research or documentation only
- [ ] Bug fix
- [ ] New feature
- [ ] Refactor (no behaviour change)
- [ ] Build, CI or packaging
- [ ] Breaking change

## Invariants

Confirm the change respects the project invariants
([CONTRIBUTING.md](CONTRIBUTING.md#non-negotiable-invariants)):

- [ ] No code path writes peer keys, preshared keys, `AllowedIPs`, `listen_port` or
      interface addresses.
- [ ] No private key or preshared key is read, stored or logged.
- [ ] The user's WireGuard configuration file is not modified.
- [ ] No WireGuard protocol or cryptography change.

If any box cannot be ticked, explain why in the summary.

## Evidence

<!--
Behavioural claims need evidence. Paste the commands you ran and their output.
For endpoint changes, include the before/after `wg show <if> dump` with keys redacted,
showing that only the endpoint column changed.
-->

```text
wg show wg0 dump | cut -f 1,4,5,7        # run before and after the switch
<paste the two outputs, keys redacted>
```

## Testing

- [ ] Unit tests added or updated
- [ ] Integration test added or updated (Linux network namespace)
- [ ] Manually verified on: <!-- linux / windows / android, with versions -->
- [ ] Not applicable (documentation only)

## Documentation

- [ ] Affected documents updated in this pull request
- [ ] New ADR added for a decision that changes architecture
- [ ] No documentation change needed

## Risk

<!-- What could this break? What is the rollback? -->
