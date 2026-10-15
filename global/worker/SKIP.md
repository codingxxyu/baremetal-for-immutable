# First delivery: do not apply this directory

Three Global control-plane hosts are enough for ACP. Do not register Global
workers on minialauda.

Why: the bootstrap ISO dies at handoff. Any worker registered here must also
be listed in `global/control-plane/bootstrap/04-dcs-import-extra-resources.yaml`
or the handoff target set is wrong. After KIND is gone you cannot fix that ISO.

After `global/management/` is Ready, add Global workers from `post-global/`
(same flow as Workload workers):

1. Apply `post-global/01-registration-seedimage.yaml` on the **finished Global**
   (`global-kubeconfig`). `baseImage` is the platform registry.
2. Never reuse the control-plane ISO or `manifests/01` (`172.18.0.1:11443`).
3. Confirm Cluster / BaremetalCluster `cpaas.io/registry-address` is already
   `156.3.1.26:11443` before building the worker ISO.

`manifests/` is the bootstrap-phase copy. Keep it `DO-NOT-APPLY` unless a later
change-request forces workers **before** POST installer; then extend the import
ConfigMap and use a **separate** ISO from the control-plane ISO.
