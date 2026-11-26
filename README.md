# Red Hat Edge Manager demo

Documentation for installing and automating Red Hat Edge Manager (RHEM) on RHEL.

## Documentation

- [Red Hat Edge Manager RPM installation instructions](README.rhem-install-rpm.md) — connected RPM control plane on RHEL 10 (manual steps and references to product docs).
- [Image builds using the RHEM image build worker](README.image-builds-basic.md) — worker subscription mounts, OCI registries, and simplified `ImageBuild` from the CLI.
- [Advanced bootc image builds (Containerfile)](README.image-builds-advanced.md) — custom layers (e.g. `advancedbuild1` → `micro.lan:5000`) with Podman on an entitled RHEL host.
- [Ansible: RHEM connected RPM install](ansible/README.md) — playbooks for install, simplified `ImageBuild`, and advanced bootc builds.
- [Scratchpad for my home lab specifics](README.scratchpad.md) — local VM, DNS, and ad hoc notes for this repo's owner.