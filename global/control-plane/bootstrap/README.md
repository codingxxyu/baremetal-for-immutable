# APPLY-ON: bootstrap (minialauda) only

Kubeconfig: bootstrap node `~/.kube/config` (KIND `minialauda`).

Do **not** apply this directory onto the finished Global.

| File | How |
|---|---|
| `01-dex-tls-bootstrap.yaml` | `kubectl apply` after minialauda is up |
| `02-kubeadm-provider-apprelease.yaml` | after packtool pushed the kubeadm chart to `127.0.0.1:11443` |
| `03-baremetal-provider-apprelease.yaml` | after OS images + dex.tls Ready |
| `04-dcs-import-extra-resources.yaml` | **after** KCP Ready; fill inventory names from BaremetalMachine; **before** POST installer |
| `05-installer-config-baremetal.json` | **not** kubectl. Copy to gitignored `installer-config.json` on the bootstrap node. POST to `http://${INSTALLER_IP}:8080/cpaas-installer/api/config/dcs`. `INSTALLER_IP` is the installer **Pod** IP. JSON cannot have comments. |

Finished-Global providers live in `../../management/`, not here.

After installer Success, **do not** re-apply `../manifests/03` or `../manifests/05`.
Those files still say `156.3.1.26:11443`. Official rewrite is Cluster only;
patch BaremetalCluster if it still points at the bootstrap node.
