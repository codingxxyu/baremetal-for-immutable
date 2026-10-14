# Changelog

## 1.1.0 — 2026-10-15

Delivery-architect revision of the 1.0 field-engineer pack.

- Split finished-Global provider AppReleases out of `global/control-plane/bootstrap/` into `global/management/`. Bootstrap is only minialauda (01–05). Applying 06/07 against KIND is a class of outage this layout removes.
- Stamp every manifest with `APPLY-ON` (bootstrap / finished Global).
- Add pack `VERSION.yaml`, preflight, and a gate table. Do not leave a phase until its gate is green.
- First delivery: `global/worker/` is skipped by layout (`SKIP.md`), not only by a README sentence.
- README cover states kubeconfig per phase. After KIND is gone, there is no default context.

## 1.0.0

Initial ACP 4.4.0 ISO first-install pack: bootstrap Global + workload YAML, no renderer, no scripts.
