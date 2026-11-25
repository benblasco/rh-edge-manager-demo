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
| `playbooks/rhem-install-rpm.yml` | Main playbook |

## Variables

Edit `inventory/group_vars/rhem_servers/rhem.yml` for FQDN, RHSM repo id, firewall ports, and demo TLS behaviour.

Secrets in `vault.yml` (copy from `vault.yml.example`):

- `rhem_pam_password` — PAM issuer UI user password
- `rhem_registry_username` / `rhem_registry_password` — Red Hat registry login

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