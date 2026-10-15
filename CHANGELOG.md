# Changelog

## 1.4.0 — 2026-10-16

Delivery gap-fill on registry addresses and post-handoff traps. ACP still **v4.4.1**.

- `cpaas.io/registry-address` is **not** the Global API VIP. It is `<host>:11443` only (no `/v2/`).
- Global **Cluster** (bootstrap apply): `156.3.1.26:11443`. Official installer rewrites **Cluster** (and DCSCluster) to `<platform-access-domain-or-vip>:11443` after the platform registry is up. Do **not** re-apply `global/control-plane/manifests/05-cluster.yaml` after that rewrite — it would put the bootstrap IP back.
- Global **BaremetalCluster** starts with the same bootstrap IP. Official rewrite set is Cluster / DCSCluster, **not** BaremetalCluster. After installer Success, if BaremetalCluster still shows `156.3.1.26:11443`, patch it to the platform registry before tearing KIND down. Do **not** re-apply `03-baremetal-cluster.yaml` after installer.
- Workload Cluster / BaremetalCluster are created on the finished Global and already use the platform registry. Never put the bootstrap IP there.
- `console.host` / BaremetalCluster `spec.controlPlaneLoadBalancer.host` stay the API VIP (`<global-control-plane-vip>`). That VIP is TCP 6443, not the registry.
- New gate: after installer, print Cluster and BaremetalCluster `cpaas.io/registry-address` and confirm they are the platform host, not `156.3.1.26`.
- First-delivery Global workers after handoff: `global/worker/post-global/` (platform `baseImage`). Bootstrap-phase YAML in `global/worker/manifests/` stays `DO-NOT-APPLY` and still points at `172.18.0.1:11443`.

## 1.3.0 — 2026-10-16

Retarget the pack from ACP **v4.4.0** to ACP **v4.4.1**. Same ISO-first / skip-bootstrap-workers / site CIDR layout.

Official OS matrix has **no v4.4.1 row**. 4.4.1 (2026-10-01) is a patch (CLI session refresh, MicroOS alive/iptables). Baked-in versions stay the **v4.4.0** matrix row:

| | |
|---|---|
| Kubernetes | `v1.35.6-acp.1` |
| containerd | `2.2.6-3` |
| coredns | `1.14.3-v4.4.6` |
| etcd | `v3.6.13-acp.0` |
| pause | `3.10` |
| kube-ovn chart | `v4.4.6` |

- Core Package filename is `installer-core-v4.4.1-hybrid.tar`. Chart tgz versions come from the **4.4.1** payload, not a leftover 4.4.0 tgz.
- Catalog key / KCP / MachineDeployment stay `v1.35.6-acp.1`. Do not mix a 4.3.x OS archive.
- Site CIDRs stay as already written (Global `10.3/10.4` join `100.64`; Workload `10.5/10.6` join `100.65`).
- Site bootstrap host IP filled as `156.3.1.26` (dex.tls SAN, elemental `:12443`, Cluster `cpaas.io/registry-address`, installer `hostIP`). Platform entry and control-plane VIPs stay placeholders.

## 1.2.0 — 2026-10-15

Gap-fill on 1.1.0. Same ACP 4.4.0 / ISO-first / skip-bootstrap-workers baseline.

- SeedImage gate is `status.downloadURL` non-empty (condition type varies: `Ready` vs `SeedImageReady`). Wait loops on the URL, not on a guessed condition name.
- SeedImage `size: 20Gi` needs a default StorageClass on the apply cluster (KIND, then finished Global). Added as a preflight / gate.
- `config/site.values.example.yaml` no longer lists Global workers as first-delivery inventory. Matches `SKIP.md`.
- Installer JSON cannot carry `APPLY-ON`. `global/control-plane/bootstrap/README.md` states: copy to `installer-config.json`, POST from the bootstrap node to the installer **Pod IP:8080** only.
- `global/management/README.md` states finished-Global kubeconfig and `caCertSecretKey: tls.crt`.
- packtool registry auth: run `packtool pack push -h` first. Do not assume `-u`.
- `jq` is optional; dex.tls key listing has a python3 fallback.
- Global worker YAML stamped `DO-NOT-APPLY` on first delivery (not only SKIP.md).
- File table lists Cluster / template / JSON apply-target. elemental URL: bootstrap `:12443`, finished Global ingress `:443`.
- README cover and VERSION bumped to 1.2.0.

## 1.1.0 — 2026-10-15

Delivery-architect revision of the 1.0 field-engineer pack.

- Split finished-Global provider AppReleases out of `global/control-plane/bootstrap/` into `global/management/`. Bootstrap is only minialauda (01–05). Applying 06/07 against KIND is a class of outage this layout removes.
- Stamp every manifest with `APPLY-ON` (bootstrap / finished Global).
- Add pack `VERSION.yaml`, preflight, and a gate table. Do not leave a phase until its gate is green.
- First delivery: `global/worker/` is skipped by layout (`SKIP.md`), not only by a README sentence.
- README cover states kubeconfig per phase. After KIND is gone, there is no default context.

## 1.0.0

Initial ACP 4.4.0 ISO first-install pack: bootstrap Global + workload YAML, no renderer, no scripts.
