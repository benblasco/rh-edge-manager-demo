# Red Hat Edge Manager RPM installation instructions

Connected RPM control plane on RHEL 10 for host `rhem.hex.blasco.id.au`. Official guide: [Install Red Hat Edge Manager on RHEL 1.4](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_install_rhem_on_rhel).

## Prerequisites

- RHEL 10 minimal, sudo, SSH, Podman, active Edge Manager subscription
- FQDN hostname (required for `flightctl-pam-issuer`)
- DNS `A`/`AAAA` for `rhem.hex.blasco.id.au` → this host
- Firewall: TCP **443** (UI/API), **7443** (enrolled devices)
- Baseline (~100 devices): 4 vCPU, 4 GB RAM, 70 GB disk, internet

See [Install the RHEM RPM package](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel) and [Network ports](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_rhem_standalone_gateway).

## Host preparation

```bash
sudo hostnamectl set-hostname rhem.hex.blasco.id.au
getenforce
podman --version
```

## DNS and TLS

Configure before starting services. Details: [Plan DNS and TLS](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_rhem_gateway_dns_and_certs).

`/etc/flightctl/service-config.yaml`:

```yaml
global:
  baseDomain: rhem.hex.blasco.id.au
  generateCertificates: builtin
```

Distribute `/etc/flightctl/pki/ca.crt` to admin clients that must trust the builtin CA.

## Install packages

[Install the RHEM RPM package](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel).

```bash
sudo podman login registry.redhat.io
sudo subscription-manager repos --enable edge-manager-1.4-for-rhel-10-x86_64-rpms
sudo dnf install -y flightctl-services
```

## Start and verify services

```bash
sudo systemctl enable --now flightctl.target
sudo systemctl list-units 'flightctl-*.service'
sudo systemctl status flightctl-gateway.service
```

Troubleshoot: `sudo journalctl -u <unit> -b --no-pager`

Web UI (after auth): `https://rhem.hex.blasco.id.au/`

## Authentication credentials

RHEM sign-in uses the **PAM issuer** container, not RHEL host accounts in `/etc/passwd`. Create users **inside** `flightctl-pam-issuer` with `podman exec`. Group `flightctl-admin` grants global administrator. See [Configure authentication credentials](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel).

Lab admin: username `user1`, password `redhat123`.

```bash
sudo podman exec flightctl-pam-issuer groupadd flightctl-admin
HASH=$(openssl passwd -6 redhat123)
sudo podman exec flightctl-pam-issuer useradd -m --groups flightctl-admin -p "$HASH" user1
```

Prefer interactive hashing in non-lab environments: `mkpasswd -m sha-512` (package `whois`) or `openssl passwd -6`.

## Flight Control CLI

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

## Verification

```bash
dig +short rhem.hex.blasco.id.au
sudo ss -tlnp | grep -E ':443|:7443'
flightctl version
getenforce
```

## References

- [Install RHEM on RHEL](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_install_rhem_on_rhel)
- [Install the RHEM RPM package](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel)
- [Plan DNS and TLS](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_rhem_gateway_dns_and_certs)
- [HTTPS gateway](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_rhem_standalone_gateway)

Disconnected/air-gapped install is out of scope for this document.