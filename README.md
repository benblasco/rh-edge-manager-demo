# Red Hat Edge Manager demo

Documentation for installing and operating Red Hat Edge Manager (RHEM) on RHEL in a home-lab style setup (gateway, image builds, Software Catalog, fleets).

**Typical order:** install the control plane → configure CLI/UI access → build or import OS images → publish catalog items → (optional) define fleets. Use either the manual guides or the Ansible playbooks under `ansible/`; the playbooks follow the same steps as the manuals.

## Manual install and configure

Step through these on the RHEM host (and related lab infra) using `flightctl` and product docs.

| Order | Guide | What it covers |
| ----- | ----- | -------------- |
| 1 | [Connected RPM install on RHEL 10](README.rhem-install-rpm.md) | Subscription, `flightctl-services`, gateway, PAM issuer, firewall, optional CLI login |
| 2 | [Web UI and CLI login](README.rhem-use.md) | `flightctl login`, UI sign-in with the PAM user |
| 3 | [Simplified image builds (worker)](README.image-builds-basic.md) | Image builder worker, OCI repositories, `ImageBuild`, `ImagePromotion` → Software Catalog |
| 4 | [Advanced bootc builds (Containerfile)](README.image-builds-advanced.md) | Custom layers with Podman, push to lab registry, `CatalogItem` for OS images |
| 5 | [Grafana quadlet catalog item](README.catalog-quadlet.md) | Application `CatalogItem` (`spec.type: quadlet`) |
| 6 | [Basic fleet: OS + Grafana](README.fleets-basic.md) | `testbuild1` + `grafana-demo` fleet; **no device required** to apply the fleet resource |

## Automated install and configure

Ansible playbooks mirror the manuals. Control-plane resources (catalog, repositories, builds, fleets) are applied from the **Ansible controller** via `flightctl.core`; host playbooks handle packages, Podman, and the image builder worker.

Start here: **[ansible/README.md](ansible/README.md)** (inventory, `vault.yml`, collections, tags).

| Order | Playbook | Manual equivalent |
| ----- | -------- | ----------------- |
| 1 | `ansible/playbooks/rhem-install-rpm.yml` | [README.rhem-install-rpm.md](README.rhem-install-rpm.md) |
| 2 | `ansible/playbooks/rhem-image-builds.yml` | [README.image-builds-basic.md](README.image-builds-basic.md) |
| 3 | `ansible/playbooks/rhem-image-builds-advanced.yml` | [README.image-builds-advanced.md](README.image-builds-advanced.md) |
| 4 | `ansible/playbooks/rhem-catalog-grafana-demo.yml` | [README.catalog-quadlet.md](README.catalog-quadlet.md) |
| 5 | `ansible/playbooks/rhem-fleet-grafana-demo.yml` | [README.fleets-basic.md](README.fleets-basic.md) |

## Lab notes

| Guide | What it covers |
| ----- | -------------- |
| [Home lab scratchpad](README.scratchpad.md) | VM creation, DNS, inventory hostnames, and owner-specific commands (not generic product procedure) |