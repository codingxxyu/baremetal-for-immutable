# Changelog

## 1.5.0 — 2026-10-15

KubeOS `alaudaos.k.25` variant for ACP **v4.4.1**, x86_64 delivery.

- Uses `kubeos-rootfs` and `kubeos-iso`; imported source tag is `baremetal`, final delivery tag is `alaudaos-feat-kubeos-provider-integration.ku.202609281026-0.x86_64`.
- Uses catalog key `v1.35.6-acp.1_amd64_alaudaos.k.25` and inventory labels `os.baremetal.alauda.io/id` / `os.baremetal.alauda.io/arch`.
- Adds KubeOS `install.system-uri`, removes the generic btrfs snapshotter and COS_STATE cloud-config, and keeps the generic `/dev/elemental-install-target` workflow.
- Uses Bare Metal provider `v1.0.0-fix.51.7.g41764fd0-feat-kubeos-support`; kubeadm remains `v1.0.14`.
- Preserves site registry handoff, Global/Workload VIPs, `hcs-workloadcluster`, and the first-delivery Global-worker skip policy.

## 1.4.0 — 2026-10-16

Delivery gap-fill on registry addresses, official Step 1 variables, and post-handoff traps. ACP still **v4.4.1**.

- Site: `HOST_IP` = `156.3.1.26` (bootstrap). `PLATFORM_HOST` / `CONTROL_PLANE_VIP` / finished-Global registry = `156.3.1.50` (6443 + 80/443 + 11443). What dies at KIND teardown is `127.0.0.1:11443` / `172.18.0.1:11443`.
- `cpaas.io/registry-address` is `<host>:11443` only (no `/v2/`). First apply Global Cluster = `156.3.1.26:11443`. After installer it must be `156.3.1.50:11443`.
- Official installer rewrites Cluster / DCSCluster to `REGISTRY_DOMAIN` (`156.3.1.50:11443`). BaremetalCluster is **not** in the rewrite set — patch it after Success. Do **not** re-apply Global `03` / `05`.
- Workload Cluster / BaremetalCluster / management AppRelease / post-handoff SeedImage use `156.3.1.50:11443`. Never `172.18.0.1` or leftover bootstrap `156.3.1.26:11443`.
- Global VIP `156.3.1.50` is API + platform entry + platform registry. Workload API VIP is `156.3.1.95`.
- This site Core Package tree is `/root/installer` (KIND / packtool / nerdctl at `/root/installer/res/amd64`). Official example `/root/cpaas-install` is not this site.
- README now includes official Step 1 `export` block, filled with this site's values. There is still **no renderer**: the exports are for the operator's shell / `setup.sh`; YAML is already baked.
- Site provider packs (not `.amd64.`): `cluster-api-provider-kubeadm.ALL.v1.0.14.tgz` (`targetRevision: v1.0.14`) and `cluster-api-provider-baremetal.ALL.v1.0.0-beta.6.ge76cf8f8.tgz` (`targetRevision: v1.0.0-beta.6.ge76cf8f8`). Written into AppRelease YAML.
- Workload cluster name is `hcs-workloadcluster`. Pool hostnames: `hcs-workloadcluster-cp-1..3` / `hcs-workloadcluster-worker-1`. MachineInventory `name` is still filled after register.
- First-delivery Global workers after handoff: `global/worker/post-global/` (platform `baseImage` `156.3.1.50:11443`). Bootstrap-phase YAML in `global/worker/manifests/` stays `DO-NOT-APPLY` (`172.18.0.1:11443`).

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
