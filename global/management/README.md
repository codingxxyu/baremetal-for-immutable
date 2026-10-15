# APPLY-ON: finished Global only

Kubeconfig: `./global-kubeconfig` (Secret `global-kubeconfig` on minialauda, then copied off the bootstrap node).

Do **not** apply this directory onto minialauda.

| File | How |
|---|---|
| `01-kubeadm-provider-apprelease.yaml` | after OS images + both charts are in the **platform** registry |
| `02-baremetal-provider-apprelease.yaml` | same; `caCertSecretKey` is `tls.crt` (installer self-signed `dex.tls` has no `ca.crt`). Change back to `ca.crt` only if the Secret actually has that key. |

`elemental.server.url` is `https://156.3.1.50` (ingress 443). It is **not** bootstrap `:12443`.

Catalog on Global starts empty. Confirm `elemental-image-catalog` has `v1.35.6-acp.1_amd64_alaudaos.k.25` resolving to `tkestack/kubeos-rootfs:alaudaos-feat-kubeos-provider-integration.ku.202609281026-0.x86_64` before any Workload apply.

For KubeOS K25, `SeedImage.baseImage` uses `kubeos-iso`, while `MachineRegistration.config.elemental.install.system-uri` uses `kubeos-rootfs`. Bootstrap SeedImages use `172.18.0.1:11443`; post-handoff registrations and SeedImages use `156.3.1.50:11443`.

Before tearing KIND down, both Cluster/global and BaremetalCluster/global must
show `cpaas.io/registry-address: 156.3.1.50:11443`, never `127.0.0.1`,
`172.18.0.1`, or leftover `156.3.1.26:11443`. Installer rewrites Cluster to
`REGISTRY_DOMAIN`. BaremetalCluster is not in the rewrite set — patch it after
Success. Do not re-apply bootstrap `manifests/03` or `05`.

Do not import bootstrap `dex.tls` (cert-manager, IP SAN) into the finished Global.
