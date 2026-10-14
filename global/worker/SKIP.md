# First delivery: skip this directory

Three Global control-plane hosts are enough for ACP. Do not register Global
workers on minialauda.

Why: the bootstrap ISO dies at handoff. Any worker registered here must also
be listed in `global/control-plane/bootstrap/04-dcs-import-extra-resources.yaml`
or the handoff target set is wrong. After KIND is gone you cannot fix that ISO.

After `global/management/` is Ready, add Global workers the same way as
Workload workers: new Registration + SeedImage on the **finished Global**,
`baseImage` on the platform registry, never the bootstrap ISO.

If a later change-request forces bootstrap-phase Global workers, come back to
these manifests, use a **separate** ISO from the control-plane ISO, and extend
the import ConfigMap before POST installer.
