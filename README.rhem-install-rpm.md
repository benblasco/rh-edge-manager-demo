# Red Hat Edge Manager RPM installation instructions

Connected RPM control plane on RHEL 10 for host `rhem.hex.blasco.id.au`. Official guide: [Install Red Hat Edge Manager on RHEL 1.4](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_install_rhem_on_rhel).

**Automated install:** same steps via Ansible — see [ansible/README.md](ansible/README.md).

## Prerequisites

- RHEL 10 minimal, sudo, SSH, Podman, active Edge Manager subscription
- FQDN hostname (required for `flightctl-pam-issuer`)
- DNS `A`/`AAAA` for `rhem.hex.blasco.id.au` → this host
- Firewall: TCP **443** (UI/API), **7443** (enrolled devices)
- Baseline (~100 devices): 4 vCPU, 4 GB RAM, 70 GB disk, internet
- `registry.redhat.io` credentials for `podman login`

See [Install the RHEM RPM package](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel) and [Network ports](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_rhem_standalone_gateway).

## Host preparation (playbook tag: `prepare`)

```bash
sudo hostnamectl set-hostname rhem.hex.blasco.id.au
podman --version
```

Open gateway ports in firewalld:

```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=7443/tcp
sudo firewall-cmd --reload
```

Plan DNS and TLS before clients connect. Details: [Plan DNS and TLS](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_rhem_gateway_dns_and_certs).

## Install packages (playbook tag: `install`)

[Install the RHEM RPM package](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel).

```bash
sudo podman login registry.redhat.io
sudo subscription-manager repos --enable edge-manager-1.4-for-rhel-10-x86_64-rpms
sudo dnf install -y flightctl-services
```

## DNS and TLS — service configuration (playbook tag: `install`, before first start)

Configure **after** `flightctl-services` is installed (the RPM ships a full default at `/etc/flightctl/service-config.yaml.rpmnew`).

**Do not** replace `/etc/flightctl/service-config.yaml` with a short snippet only. Services render configs from the **full** file; a minimal file breaks API, PAM issuer, and gateway startup.

Seed from the package default and set your FQDN:

```bash
sudo cp /etc/flightctl/service-config.yaml.rpmnew /etc/flightctl/service-config.yaml
sudo sed -i 's/^  baseDomain:.*/  baseDomain: rhem.hex.blasco.id.au/' /etc/flightctl/service-config.yaml
```

Confirm OIDC/PAM auth is enabled (present in `.rpmnew`):

```yaml
global:
  auth:
    type: oidc
```

Other keys you typically leave at RPM defaults: `generateCertificates: builtin`, `global.auth.oidc`, `global.auth.pamOidcIssuer`, and per-service sections.

Distribute `/etc/flightctl/pki/ca.crt` to admin clients that must trust the builtin CA.

## Start and verify services (playbook tag: `install`)

Apply service config **before** the first start:

```bash
sudo systemctl enable --now flightctl.target
sudo systemctl list-units 'flightctl-*.service'
sudo systemctl status flightctl-gateway.service
```

Troubleshoot: `sudo journalctl -u <unit> -b --no-pager`

Web UI (after auth): `https://rhem.hex.blasco.id.au/`

### If the gateway or API keeps restarting

Symptoms: `flightctl-gateway.service` stuck in `activating`, nothing listening on 443/7443, journal mentions `global.auth.type: Required value` or template render errors.

```bash
sudo cp /etc/flightctl/service-config.yaml.rpmnew /etc/flightctl/service-config.yaml
sudo sed -i 's/^  baseDomain:.*/  baseDomain: rhem.hex.blasco.id.au/' /etc/flightctl/service-config.yaml
sudo systemctl restart flightctl.target
```

Do **not** run `flightctl-standalone render quadlets` on an RPM-only host; it expects a source tree under `deploy/podman/`.

## Authentication credentials (playbook tag: `pam`)

RHEM sign-in uses the **PAM issuer** container, not RHEL host accounts in `/etc/passwd`. Create users **inside** `flightctl-pam-issuer` with `podman exec` after `flightctl-pam-issuer` is running. Group `flightctl-admin` grants global administrator. See [Configure authentication credentials](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel).

Lab admin: username `user1`, password `redhat123`.

```bash
sudo podman exec flightctl-pam-issuer groupadd flightctl-admin
HASH=$(openssl passwd -6 redhat123)
sudo podman exec flightctl-pam-issuer useradd -m --groups flightctl-admin -p "$HASH" user1
```

Prefer interactive hashing in non-lab environments: `mkpasswd -m sha-512` (package `whois`) or `openssl passwd -6`.

## Flight Control CLI (playbook tag: `cli`)

```bash
sudo dnf install -y flightctl-cli
flightctl version
flightctl login https://rhem.hex.blasco.id.au --username user1 --password redhat123
```

If the client does not trust the builtin CA, install `/etc/flightctl/pki/ca.crt` or use `--insecure-skip-tls-verify` for lab only.

## Managed devices (image mode only)

This deployment manages **bootc / image-mode** devices only. Do not deploy package-mode agents.

Next steps:

- [Operating system images](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-con_edge_mgr_images_intro)
- [Enrollment methods](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_enrollment_methods)
- [Enroll devices](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/get_started-con_enrolling_devices)

```bash
flightctl get enrollmentrequests --field-selector="status.approval.approved != true"
flightctl approve enrollmentrequest/<name>
```

Enrollment approval is manual; it is not part of the Ansible install playbook.

## Verification (playbook tag: `verify`)

```bash
dig +short rhem.hex.blasco.id.au
sudo ss -tlnp | grep -E ':443|:7443'
sudo systemctl is-active flightctl-gateway.service
flightctl version
```

## References

- [Install RHEM on RHEL](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_install_rhem_on_rhel)
- [Install the RHEM RPM package](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel)
- [Plan DNS and TLS](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_rhem_gateway_dns_and_certs)
- [HTTPS gateway](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_rhem_standalone_gateway)
- [Ansible install (this repo)](ansible/README.md)

Disconnected/air-gapped install is out of scope for this document.