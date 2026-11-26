# Advanced bootc image builds (Containerfile + Podman)

Custom OS layers for Red Hat Edge Manager (RHEM) on lab host `rhem.hex.blasco.id.au`. This path **complements** the simplified [`ImageBuild`](README.image-builds-basic.md) worker flow: you build and push the image yourself with **Podman** and a **Containerfile**, then use RHEM for enrollment, fleets, catalog, and rollouts.

**Prerequisites from the simplified doc still apply:** working RHEM install, entitled RHEL 10 build host, `registry.redhat.io` login, and lab registry `micro.lan:5000`. See [README.image-builds-basic.md](README.image-builds-basic.md) (worker, OCI `Repository` resources, variables table).

**Automated build (early binding, Containerfile, podman build/push):** [ansible/README.md](ansible/README.md) — playbook `ansible/playbooks/rhem-image-builds-advanced.yml`, variables in `ansible/inventory/group_vars/rhem_servers/rhem_advanced_imagebuilds.yml`, Jinja2 under `ansible/templates/advancedbuilds/`.

**Product guide:** [Build a bootc operating system image for the Red Hat Edge Manager](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-assembly_edge_mgr_build_images).

## How this differs from simplified `ImageBuild`

| | Simplified (`testbuild1`) | Advanced (`advancedbuild1`) |
| --- | --- | --- |
| Custom RPMs / Containerfile | No | Yes |
| Build mechanism | RHEM image builder worker | `podman build` on an entitled RHEL host |
| Agent + binding | Worker adds automatically | You install `flightctl-agent` in the Containerfile; optional `config.yaml` for early binding |
| Push | Worker → OCI `Repository` | `podman push` to the same lab registry |
| RHEM afterward | Catalog via `ImagePromotion` | Register image in Software Catalog (Git sync or catalog items) for fleet rollouts |

## Lab variables (`advancedbuild1`)

Shared with [README.image-builds-basic.md](README.image-builds-basic.md) unless noted:

| Variable | Lab value | Notes |
| --- | --- | --- |
| `RHEM_FQDN` | `rhem.hex.blasco.id.au` | Gateway / API |
| `BASE_IMAGE` | `registry.redhat.io/rhel10/rhel-bootc:1795492183` | Same pinned base as `testbuild1` |
| `RHSM_REPO_ID` | `edge-manager-1.4-for-rhel-10-x86_64-rpms` | Required to install `flightctl-agent` during build |
| `BUILT_REGISTRY` | `micro.lan:5000` | Same destination registry (this lab: **HTTPS**, no auth) |
| `BUILD_NAME` | `advancedbuild1` | Local build context directory and pushed image name |
| `OCI_IMAGE_REPO` | `micro.lan:5000/advancedbuild1` | Full push reference |
| `OCI_IMAGE_TAG` | `latest` | Use a new tag per version in production |

Example image on the registry: `micro.lan:5000/advancedbuild1:latest`.

## Prerequisites (advanced-specific)

On the **build host** (the RHEM VM is entitled and a practical default):

- RHEL 10 with active subscription and `edge-manager-1.4-for-rhel-10-x86_64-rpms` available (same as the image builder worker).
- **Podman** 5.0+ and **skopeo** 1.14+ (`dnf list installed podman skopeo`).
- `podman login registry.redhat.io` (same credentials as in [README.image-builds-basic.md](README.image-builds-basic.md)).
- Network reachability to `micro.lan:5000` from the build host.
- **`flightctl-cli`** logged in if you use **early binding** (enrollment cert in the image). See [README.rhem-use.md](README.rhem-use.md).

Optional later steps (out of scope for this lab recipe): `bootc-image-builder` for ISO/QCOW2/VMDK, [Sigstore signing](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/secure-proc_sign_publish_bootc_image_sigstore).

## Build context and Containerfile

Working copy in this repo:

```text
advancedbuild1/
  Containerfile    # base + agent + tmux, bash-completion, cockpit
  config.yaml      # optional; generated on the build host for early binding
```

The Containerfile starts from the same RHEL 10 bootc tag as the simplified build, enables the Edge Manager RPM repo, installs the **RHEM agent** and lab packages, and enables services:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:1795492183

RUN dnf config-manager --set-enabled edge-manager-1.4-for-rhel-10-x86_64-rpms && \
    dnf -y install flightctl-agent tmux bash-completion cockpit && \
    dnf -y clean all && \
    systemctl enable flightctl-agent.service && \
    systemctl enable cockpit.socket && \
    systemctl mask bootc-fetch-apply-updates.timer
```

- **`flightctl-agent`** — required for RHEM-managed devices; masks automatic bootc fetch-apply so RHEM owns OS updates ([build with bootc](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_build_bootc_image)).
- **`cockpit`** — web UI; `cockpit.socket` is enabled for socket activation.
- **`tmux`**, **`bash-completion`** — illustrative customizations (replace or extend for your workload).

Copy the directory to the build host, or clone this repo on `rhem.hex.blasco.id.au`:

```bash
sudo mkdir -p /var/tmp/advancedbuild1
# Copy Containerfile (and optional config.yaml) into /var/tmp/advancedbuild1
cd /var/tmp/advancedbuild1
```

With Ansible, the build context is rendered on the host at `rhem_advanced_build_dir` (default `/etc/flightctl/advancedbuilds/advancedbuild1`).

### Optional: early binding (matches `testbuild1` lab)

The simplified `ImageBuild` embeds enrollment settings via the worker. On the advanced path, generate `config.yaml` and add it to the image before build.

1. From a host with `flightctl-cli`:

```bash
export RHEM_API_SERVER_URL=rhem.hex.blasco.id.au
flightctl login "https://${RHEM_API_SERVER_URL}" --username user1 --password '<password>' --insecure-skip-tls-verify
flightctl certificate request --signer=enrollment --expiration=365d --output=embedded > config.yaml
```

2. Uncomment or add to the Containerfile:

```dockerfile
ADD config.yaml /etc/flightctl/
```

Procedure: [Optional: Requesting an enrollment certificate for early binding](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_request_enrollment_cert_early_binding).

Early binding does **not** auto-approve enrollment; devices still need an approved enrollment request ([Enrollment methods](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_enrollment_methods)).

For **late binding**, omit `config.yaml` and provision enrollment at install time (`cloud-init`, Ignition, kickstart, or onboarding wizard).

## Configure Podman for the lab registry (HTTPS, self-signed)

If `micro.lan:5000` uses HTTPS with a certificate that Podman does not trust, allow insecure pulls/pushes for this lab host only:

```bash
sudo tee /etc/containers/registries.conf.d/998-micro-lan-lab.conf <<'EOF'
[[registry]]
location = "micro.lan:5000"
insecure = true
EOF
```

Adjust if your registry is plain HTTP instead (see [README.image-builds-basic.md](README.image-builds-basic.md) OCI `Repository` `scheme` notes).

Confirm reachability:

```bash
curl -kfsS "https://micro.lan:5000/v2/" || curl -fsS "http://micro.lan:5000/v2/"
getenforce
```

## Build and push

On the entitled build host, in the directory that contains the Containerfile:

```bash
export OCI_IMAGE_REPO=micro.lan:5000/advancedbuild1
export OCI_IMAGE_TAG=latest

sudo podman build -t "${OCI_IMAGE_REPO}:${OCI_IMAGE_TAG}" .
sudo podman push "${OCI_IMAGE_REPO}:${OCI_IMAGE_TAG}"
```

If your environment requires Sigstore signatures (production or fleet policy), follow [Sign and publish the bootc operating system image by using Sigstore](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/secure-proc_sign_publish_bootc_image_sigstore) instead of a plain `podman push`.

Confirm the image on the registry:

```bash
curl -kfsS "https://micro.lan:5000/v2/advancedbuild1/tags/list"
# or: podman pull micro.lan:5000/advancedbuild1:latest
```

RHEM’s OCI `Repository` resource `micro-lan-built` (from the simplified setup) already points at this registry; advanced images are consumed by **catalog/fleet references**, not by creating another `ImageBuild`.

## After push: using the image with RHEM

1. **Software Catalog** — Publish so fleets and templates can reference the image by catalog item (Git-managed catalog YAML or UI), not via `ImagePromotion` tied to an `ImageBuild`. See [Software Catalog](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/deploy-con_software_catalog) and [Catalog reference rendering](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/deploy-con_catalog_reference_rendering).
2. **Fleets / devices** — Roll out `catalogItemRef` (or equivalent) to devices; agents pull from the OCI URL resolved by RHEM.
3. **Optional disk media** — [Build the operating system disk image](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-assembly_edge_mgr_build_images) with `bootc-image-builder` from the pushed bootc reference.

Runtime application and device configuration can still change without rebaking; see [Special considerations for building images](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-con_edge_mgr_images_consider).

## Alternative: layer on `testbuild1`

If you already have `micro.lan:5000/testbuild1:latest` from the simplified worker (agent + early binding baked in), you can add packages only:

```dockerfile
FROM micro.lan:5000/testbuild1:latest

RUN dnf -y install tmux bash-completion cockpit && \
    dnf -y clean all && \
    systemctl enable cockpit.socket
```

Push to a **new name or tag** (for example `advancedbuild1:v2`) to avoid overwriting the worker-managed `testbuild1` image.

## Verification

```bash
dnf list installed podman skopeo flightctl-cli
sudo podman images micro.lan:5000/advancedbuild1:latest
curl -kfsS "https://micro.lan:5000/v2/advancedbuild1/tags/list"
getenforce
```

On a device or test VM booted from the image (after enrollment): confirm packages and Cockpit socket:

```bash
rpm -q tmux bash-completion cockpit flightctl-agent
systemctl is-enabled flightctl-agent.service cockpit.socket
getenforce
```

## References

- [Choosing an image building path](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-con_edge_mgr_images_intro)
- [Build the operating system image with bootc](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_build_bootc_image)
- [Build for specific target platforms](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_edge_mgr_build_images_platform)
- [Sign and publish bootc image (Sigstore)](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/secure-proc_sign_publish_bootc_image_sigstore)
- [Simplified worker flow (this repo)](README.image-builds-basic.md)
- [RHEM CLI login (this repo)](README.rhem-use.md)