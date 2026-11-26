# Ansible: RHEM connected RPM install

Automates the connected install described in [../README.rhem-install-rpm.md](../README.rhem-install-rpm.md) on a single RHEL 10 host (gateway + local `flightctl` CLI).

## Prerequisites (manual)

- RHEL 10 with sudo over SSH, Podman, active Edge Manager subscription, and firewall managed by firewalld.
- DNS `A`/`AAAA` for `rhem_fqdn` pointing at this host.
- `registry.redhat.io` credentials for `podman login`.
- Ansible control node with `ansible-playbook`, collections from `requirements.yml`, and **Python 3.12+** when using `flightctl.core` playbooks (catalog, and future fleet/image API automation).

## Layout

| Path | Purpose |
|------|---------|
| `requirements.yml` | Galaxy collections |
| `inventory/hosts.yml` | Target host(s) in group `rhem_servers` |
| `inventory/group_vars/rhem_servers/rhem.yml` | Non-secret variables |
| `inventory/group_vars/rhem_servers/vault.yml` | Secrets (not in git; see `vault.yml.example`) |
| `playbooks/rhem-install-rpm.yml` | Connected RPM install |
| `playbooks/rhem-image-builds.yml` | Worker on host; OCI repos, ImageBuild, ImagePromotion via `flightctl.core` on localhost |
| `playbooks/rhem-image-builds-advanced.yml` | Advanced bootc image on host; `CatalogItem` via API on localhost |
| `playbooks/rhem-catalog-grafana-demo.yml` | Grafana quadlet `CatalogItem` → `default` catalog ([README.catalog-quadlet.md](../README.catalog-quadlet.md)) |
| `playbooks/rhem-fleet-grafana-demo.yml` | Grafana quadlet `Fleet` referencing catalog item (API on localhost) |
| `tasks/rhem_flightctl_render_and_apply.yml` | Shared: Jinja → `flightctl_resource` / `flightctl_image_builder` (no `flightctl apply`) |
| `tasks/rhem_flightctl_wait_imagebuild.yml` | Poll `flightctl_image_builder_info` until build completes |
| `tasks/rhem_flightctl_wait_imagepromotion.yml` | Poll `flightctl_resource_info` for `ImagePromotion` |
| `templates/imagebuilds/*.j2` | API manifests for Repository, ImageBuild, ImagePromotion (rendered on controller) |
| `templates/advancedbuilds/*.j2` | Containerfile, registry drop-in, advanced `CatalogItem` |
| `templates/fleets/*.j2` | Fleet definitions (e.g. Grafana quadlet + `catalogItemRef`) |
| `files/imagebuilder-worker-subscription.conf` | Quadlet drop-in for RHSM mounts on the worker |
| `inventory/group_vars/rhem_servers/rhem_imagebuilds.yml` | Simplified ImageBuild variables (non-secret) |
| `inventory/group_vars/rhem_servers/rhem_advanced_imagebuilds.yml` | Advanced build variables (non-secret; uses registry vars from `rhem_imagebuilds.yml`) |
| `inventory/group_vars/rhem_servers/rhem_flightctl_connection.yml` | Shared `flightctl.core` API host, TLS, username (password in vault) |
| `inventory/group_vars/rhem_servers/rhem_catalog_grafana.yml` | Grafana catalog apply toggles and item name |
| `inventory/group_vars/rhem_servers/rhem_fleet_grafana.yml` | Grafana fleet name and catalog references |

## Variables

Edit `inventory/group_vars/rhem_servers/rhem.yml` for FQDN, RHSM repo id, firewall ports, and demo TLS behaviour.

Secrets in `vault.yml` (copy from `vault.yml.example`):

- `rhem_pam_password` — PAM issuer UI user password
- `rhem_registry_username` / `rhem_registry_password` — Red Hat registry login (host `podman login` and OCI `Repository` ociAuth)
- Optional: `rhem_built_registry_username` / `rhem_built_registry_password` — private push registry auth

Image-build toggles and registry hostnames: `inventory/group_vars/rhem_servers/rhem_imagebuilds.yml`. Advanced build names, packages, and binding: `rhem_advanced_imagebuilds.yml`.

## Collections and controller Python

Galaxy collections (includes `flightctl.core` for RHEM API playbooks):

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

`flightctl.core` modules call the RHEM HTTPS API from the **Ansible controller** (not over SSH to the gateway). The controller must run **Python 3.12 or newer** and the pip packages in `requirements-controller.txt`:

```bash
python3 --version   # expect 3.12+
python3 -m pip install -r requirements-controller.txt
```

The RHEM gateway must be **API version 1.3+** (1.4 RPM in this lab). The controller needs DNS/network reachability to `https://{{ rhem_fqdn }}` for catalog and other API playbooks.

RHEL verification on the controller:

```bash
python3 --version
python3 -m pip show flightctl-client PyYAML jsonpatch jsonschema
ansible-galaxy collection list flightctl.core
```

## Secrets

```bash
cp inventory/group_vars/rhem_servers/vault.yml.example \
   inventory/group_vars/rhem_servers/vault.yml
# edit vault.yml, then:
ansible-vault encrypt inventory/group_vars/rhem_servers/vault.yml
```

For a lab-only host you may leave `vault.yml` unencrypted (still gitignored); do not commit it.

## Run

From this directory (`ansible/`):

```bash
ansible-playbook playbooks/rhem-install-rpm.yml --ask-vault-pass
```

If vault is not encrypted, omit `--ask-vault-pass`.

### Play order and tags

Tasks run in this order (aligned with the manual README):

1. **`prepare`** — RHEL check, Podman, FQDN hostname, firewalld, `/etc/flightctl` directory
2. **`install`** — RHSM repo, registry login, `flightctl-services` RPM, **then** seed `service-config.yaml` from `.rpmnew` and set `baseDomain`, flush handlers, start `flightctl.target`, wait for gateway
3. **`pam`** — PAM admin group/user in `flightctl-pam-issuer`
4. **`cli`** — `flightctl-cli`, optional CA trust, `flightctl login`
5. **`verify`** — Ports, gateway, DNS from controller, summary

Service configuration tasks are tagged **`prepare` and `install`** but execute only **after** the RPM is installed (they need `service-config.yaml.rpmnew` on the host).

| Tag | Main steps |
|-----|------------|
| `prepare` | Hostname, firewalld, prereq checks (config steps run after RPM if you also use `install`) |
| `install` | Repo, registry, RPM, service config from `.rpmnew`, `flightctl.target` |
| `pam` | Container admin group/user |
| `cli` | CLI package and login |
| `verify` | Post-install checks |

For a full connected install, run without `--tags` or use:

```bash
ansible-playbook playbooks/rhem-install-rpm.yml --ask-vault-pass
```

Partial run (must include `install` if the host has no RHEM packages yet):

```bash
ansible-playbook playbooks/rhem-install-rpm.yml --tags prepare,install --ask-vault-pass
```

## Image builds

After `rhem-install-rpm.yml` succeeds on the same inventory:

```bash
ansible-playbook playbooks/rhem-image-builds.yml --ask-vault-pass
```

### Play order and tags (`rhem-image-builds.yml`)

1. **`worker`** — Quadlet subscription drop-in, `imagebuilderWorker` keys in `service-config.yaml`, worker running ([product doc](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_configuring_imagebuilder_worker_subscription_rhel))
2. **`repositories`** (localhost) — `flightctl_resource` for OCI `Repository` manifests from `templates/imagebuilds/` ([OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository))
3. **`imagebuild`** (localhost) — `flightctl_image_builder` for `ImageBuild`; optional wait via `flightctl_image_builder_info`
4. **`promotion`** (localhost) — `flightctl_resource` for `ImagePromotion` when `rhem_imagebuild_promotion_enabled: true` ([publish from CLI](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_image_build_catalog_cli))
5. **`verify`** — Worker and registry on host; API `flightctl_resource_info` / `flightctl_image_builder_info` on localhost

| Tag | Main steps |
|-----|------------|
| `worker` | RHSM mounts, `rpmRepoAdd` / `rpmRepoEnable`, worker service |
| `repositories` | `registry-redhat-io` and built-registry `Repository` resources |
| `imagebuild` | Apply `ImageBuild` (waits by default when promotion is enabled via `rhem_imagebuild_wait_complete`) |
| `promotion` | Software Catalog `ImagePromotion` (waits via `rhem_imagepromotion_wait_complete` when enabled) |
| `verify` | Post-config checks |

Configure worker and repos only (no build):

```bash
ansible-playbook playbooks/rhem-image-builds.yml --tags worker,repositories --ask-vault-pass
```

A full run with default lab vars can take **30+ minutes** while the worker completes the build and promotion (`rhem_imagebuild_wait_complete` / `rhem_imagepromotion_wait_complete`). Set those to `false` for a quick manifest-only apply, then re-run with `--tags imagebuild,promotion,verify` after the build finishes.

RHEL verification after image-build playbook:

```bash
# On RHEM host
sudo systemctl is-active flightctl-imagebuilder-worker.service

# On Ansible controller (re-run verify tag or CLI on gateway)
ansible-playbook playbooks/rhem-image-builds.yml --tags verify --ask-vault-pass
flightctl get repository
flightctl get imagepromotion/testbuild1-promotion
flightctl logs imagebuild/testbuild1 -f
```

New control-plane playbooks should follow the same pattern: Jinja under `templates/`, `include_tasks: ../tasks/rhem_flightctl_render_and_apply.yml`, `module_defaults` from `rhem_flightctl_connection.yml`, and **no** `command: flightctl apply`.

## Software Catalog: Grafana quadlet

After `rhem-install-rpm.yml` succeeds, install controller pip deps and collections (see [Collections and controller Python](#collections-and-controller-python)), then:

```bash
ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --ask-vault-pass
```

The playbook reads [`catalog/applications/grafana-demo/catalogitem.yaml`](../catalog/applications/grafana-demo/catalogitem.yaml) on the controller and applies it with `flightctl.core.flightctl_resource` (`kind: CatalogItem`, `catalog_name: default`). It does not copy a manifest to the RHEM host or require `flightctl` CLI on the gateway for catalog publish. Fleet deployment is not part of this playbook.

Connection settings: `rhem_flightctl_connection.yml` and `rhem_pam_password` in vault (same PAM user as `flightctl login` on the host).

### Tags (`rhem-catalog-grafana-demo.yml`)

| Tag | Main steps |
|-----|------------|
| `catalog` | Assert manifest on controller, `flightctl_resource` apply, `flightctl_resource_info` |
| `verify` | API catalog item lookup and summary |

Re-apply catalog item only (localhost API play):

```bash
ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --tags catalog --ask-vault-pass
```

Set `rhem_catalog_grafana_enabled: false` in `rhem_catalog_grafana.yml` to skip the play.

RHEL verification after a successful run (controller — API object):

```bash
ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --tags verify --ask-vault-pass
```

On the RHEM host (optional CLI checks):

```bash
flightctl get catalogitem grafana-demo --catalog default
```

## Advanced image builds

After `rhem-install-rpm.yml`, and preferably `rhem-image-builds.yml --tags repositories` so RHEM has the `micro-lan-built` OCI `Repository`:

```bash
ansible-playbook playbooks/rhem-image-builds-advanced.yml --ask-vault-pass
```

### Play order and tags (`rhem-image-builds-advanced.yml`)

1. **`prereq`** — RHSM repo, `registry.redhat.io` login, lab registry drop-in for Podman (HTTPS skip-TLS)
2. **`earlybinding`** — `flightctl login` if needed; `flightctl certificate request` → `config.yaml` (early binding)
3. **`context`** — Build directory and rendered `Containerfile`
4. **`build`** — `podman build` (async; may take 30+ minutes)
5. **`push`** — `podman push` to `micro.lan:5000/advancedbuild1:latest`
6. **`promotion`** — Apply `CatalogItem` to the Software Catalog (not `ImagePromotion`; there is no `ImageBuild` on this path)
7. **`verify`** — Registry probe, API/CLI catalog item lookup

| Tag | Main steps |
|-----|------------|
| `prereq` | Entitlement, registry logins, `registries.conf.d` drop-in |
| `earlybinding` | Enrollment cert embedded in `config.yaml` |
| `context` | `/etc/flightctl/advancedbuilds/advancedbuild1` layout |
| `build` | `podman build` |
| `push` | Push to lab registry |
| `promotion` (localhost) | `flightctl_resource` `CatalogItem` (`rhem_advanced_catalog_enabled`) |
| `verify` | Podman image, registry tags, catalog item listing |

Render context and renew enrollment cert only (no rebuild):

```bash
ansible-playbook playbooks/rhem-image-builds-advanced.yml --tags earlybinding,context --ask-vault-pass
```

Set `rhem_advanced_regenerate_enrollment_cert: true` in `rhem_advanced_imagebuilds.yml` to replace `config.yaml`, then re-run `build` and `push`.

Catalog only after a manual or prior `podman push`:

```bash
ansible-playbook playbooks/rhem-image-builds-advanced.yml --tags promotion,verify --ask-vault-pass
```

RHEL verification after advanced build (target host):

```bash
sudo podman images micro.lan:5000/advancedbuild1:latest
curl -kfsS "https://micro.lan:5000/v2/advancedbuild1/tags/list"
flightctl get catalogitem advancedbuild1 --catalog default
flightctl get catalogitems --catalog default
```

## Fleet: Grafana quadlet

After `rhem-catalog-grafana-demo.yml` registers the catalog item:

```bash
ansible-playbook playbooks/rhem-fleet-grafana-demo.yml --ask-vault-pass
```

| Tag | Main steps |
|-----|------------|
| `fleet` | Render `templates/fleets/grafana-demo.yaml.j2`, `flightctl_resource` `Fleet` |
| `verify` | `flightctl_resource_info` and summary |

Set `rhem_fleet_grafana_enabled: false` to skip. Devices must carry label `fleet: <rhem_fleet_grafana_name>` (enrollment not in this playbook).

## RHEL verification (on the target after a successful run)

```bash
sudo systemctl is-active flightctl-gateway.service
sudo ss -tlnp | grep -E ':443|:7443'
flightctl version
dig +short "$(hostname -f)"
```

## Notes

- `service-config.yaml` is seeded from `/etc/flightctl/service-config.yaml.rpmnew` (full RPM defaults); only `global.baseDomain` is patched. Do not replace the file with a minimal YAML snippet.
- `rhem_render_quadlets_on_config` defaults to `false`. `flightctl-standalone render quadlets` expects a source tree and fails on an RPM-only host.
- PAM password hashing runs on the **RHEM host** via `openssl passwd -6` (same as the manual install README); no extra Python packages on the Ansible controller.
- Changing `service-config.yaml` notifies a handler that restarts `flightctl.target`.
- PAM users are created inside the `flightctl-pam-issuer` container, not in `/etc/passwd` on the host.
- Enrollment approval (`flightctl approve …`) is post-install and not part of this playbook.