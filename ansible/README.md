# Ansible: RHEM connected RPM install

Automates the connected install described in [../README.rhem-install-rpm.md](../README.rhem-install-rpm.md) on a single RHEL 10 host (gateway + local `flightctl` CLI).

## Prerequisites (manual)

- RHEL 10 with sudo over SSH, Podman, active Edge Manager subscription, and firewall managed by firewalld.
- DNS `A`/`AAAA` for `rhem_fqdn` pointing at this host.
- `registry.redhat.io` credentials for `podman login`.
- Ansible control node with `ansible-playbook` and collections from `requirements.yml`.

## Layout

| Path | Purpose |
|------|---------|
| `requirements.yml` | Galaxy collections |
| `inventory/hosts.yml` | Target host(s) in group `rhem_servers` |
| `inventory/group_vars/rhem_servers/rhem.yml` | Non-secret variables |
| `inventory/group_vars/rhem_servers/vault.yml` | Secrets (not in git; see `vault.yml.example`) |
| `playbooks/rhem-install-rpm.yml` | Connected RPM install |
| `playbooks/rhem-image-builds.yml` | Image builder worker, OCI repos, ImageBuild ([README.image-builds-basic.md](../README.image-builds-basic.md)) |
| `playbooks/rhem-image-builds-advanced.yml` | Advanced bootc image: Containerfile, early binding, `podman build`/`push` ([README.image-builds-advanced.md](../README.image-builds-advanced.md)) |
| `templates/imagebuilds/*.j2` | `flightctl apply` manifests (Repository, ImageBuild, optional ImagePromotion) |
| `templates/advancedbuilds/*.j2` | Containerfile and Podman registry drop-in for advanced builds |
| `files/imagebuilder-worker-subscription.conf` | Quadlet drop-in for RHSM mounts on the worker |
| `inventory/group_vars/rhem_servers/rhem_imagebuilds.yml` | Simplified ImageBuild variables (non-secret) |
| `inventory/group_vars/rhem_servers/rhem_advanced_imagebuilds.yml` | Advanced build variables (non-secret; uses registry vars from `rhem_imagebuilds.yml`) |

## Variables

Edit `inventory/group_vars/rhem_servers/rhem.yml` for FQDN, RHSM repo id, firewall ports, and demo TLS behaviour.

Secrets in `vault.yml` (copy from `vault.yml.example`):

- `rhem_pam_password` — PAM issuer UI user password
- `rhem_registry_username` / `rhem_registry_password` — Red Hat registry login (host `podman login` and OCI `Repository` ociAuth)
- Optional: `rhem_built_registry_username` / `rhem_built_registry_password` — private push registry auth

Image-build toggles and registry hostnames: `inventory/group_vars/rhem_servers/rhem_imagebuilds.yml`. Advanced build names, packages, and binding: `rhem_advanced_imagebuilds.yml`.

## Collections

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
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
5. **`verify`** — SELinux, ports, gateway, DNS from controller, summary

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
2. **`repositories`** — Render and `flightctl apply` OCI `Repository` manifests ([OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository))
3. **`imagebuild`** — Render and apply `ImageBuild` ([simplified workflow](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_edge_mgr_build_images_simple))
4. **`promotion`** — `ImagePromotion` when `rhem_imagebuild_promotion_enabled: true` (default **on** in lab `rhem_imagebuilds.yml`) ([publish from CLI](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_image_build_catalog_cli))
5. **`verify`** — Worker active, `flightctl get`, catalog items, built-registry probe, summary

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

RHEL verification after image-build playbook (target host):

```bash
sudo systemctl is-active flightctl-imagebuilder-worker.service
flightctl get repository
flightctl get imagepromotion/testbuild1-promotion
flightctl get catalogitems --catalog default
flightctl logs imagebuild/testbuild1 -f
getenforce
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
6. **`verify`** — Local image list, registry tag probe, `getenforce`

| Tag | Main steps |
|-----|------------|
| `prereq` | Entitlement, registry logins, `registries.conf.d` drop-in |
| `earlybinding` | Enrollment cert embedded in `config.yaml` |
| `context` | `/etc/flightctl/advancedbuilds/advancedbuild1` layout |
| `build` | `podman build` |
| `push` | Push to lab registry |
| `verify` | Post-build checks |

Render context and renew enrollment cert only (no rebuild):

```bash
ansible-playbook playbooks/rhem-image-builds-advanced.yml --tags earlybinding,context --ask-vault-pass
```

Set `rhem_advanced_regenerate_enrollment_cert: true` in `rhem_advanced_imagebuilds.yml` to replace `config.yaml`, then re-run `build` and `push`.

RHEL verification after advanced build (target host):

```bash
sudo podman images micro.lan:5000/advancedbuild1:latest
curl -kfsS "https://micro.lan:5000/v2/advancedbuild1/tags/list"
getenforce
```

## RHEL verification (on the target after a successful run)

```bash
getenforce
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