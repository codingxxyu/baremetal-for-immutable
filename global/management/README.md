# APPLY-ON: finished Global only

Kubeconfig: `./global-kubeconfig` (Secret `global-kubeconfig` on minialauda, then copied off the bootstrap node).

Do **not** apply this directory onto minialauda.

| File | How |
|---|---|
| `01-kubeadm-provider-apprelease.yaml` | after OS images + both charts are in the **platform** registry |
| `02-baremetal-provider-apprelease.yaml` | same; `caCertSecretKey` is `tls.crt` (installer self-signed `dex.tls` has no `ca.crt`). Change back to `ca.crt` only if the Secret actually has that key. |

`elemental.server.url` is `https://156.3.1.26` (ingress 443). It is **not** bootstrap `:12443`.

Catalog on Global starts empty. Confirm `elemental-image-catalog` has `v1.35.6-acp.1` before any Workload apply.

Before tearing KIND down, both Cluster/global and BaremetalCluster/global must
show `cpaas.io/registry-address: 156.3.1.26:11443`, never `127.0.0.1` or
`172.18.0.1`. On this site PLATFORM_HOST = HOST_IP, so installer rewrite of
Cluster is a no-op. BaremetalCluster is not in the rewrite set; leave it.
Do not re-apply bootstrap `manifests/03` or `05`.

Do not import bootstrap `dex.tls` (cert-manager, IP SAN) into the finished Global.
