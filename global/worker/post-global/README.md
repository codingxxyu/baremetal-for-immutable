# APPLY-ON: finished Global only

Kubeconfig: `./global-kubeconfig`.

This is the ISO for **optional Global workers after handoff**. First delivery
still skips workers.

`../manifests/` is the bootstrap-phase copy (`baseImage` `172.18.0.1:11443`).
Do not apply it on the finished Global.

Pool / MachineDeployment YAML for Global workers is not duplicated here.
After this ISO registers inventories, copy the shape from
`workload/worker/manifests/02`–`05` and set `cpaas.io/cluster-name: global`.
Do not reuse the Workload ISO.
