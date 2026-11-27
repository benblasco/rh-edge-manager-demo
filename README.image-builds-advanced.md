# Advanced bootc image builds (Containerfile + Podman)

Custom OS layers for Red Hat Edge Manager (RHEM) on lab host `rhem.hex.blasco.id.au`. This path **complements** the simplified [`ImageBuild`](README.image-builds-basic.md) worker flow: you build and push the image yourself with **Podman** and a **Containerfile**, then use RHEM for enrollment, fleets, catalog, and rollouts.

**Prerequisites from the simplified doc still apply:** working RHEM install, entitled RHEL 10 build host, `registry.redhat.io` login, and lab registry `micro.lan:5000`. See [README.image-builds-basic.md](README.image-builds-basic.md) (worker, OCI `Repository` resources, variables table).

**Automated build (early binding, Containerfile, podman build/push, QCOW2 via bootc-image-builder):** [ansible/README.md](ansible/README.md) — playbook `ansible/playbooks/rhem-image-builds-advanced.yml`, variables in `ansible/inventory/group_vars/rhem_servers/rhem_advanced_imagebuilds.yml`, Jinja2 under `ansible/templates/advancedbuilds/`.

**Product guide:** [Build a bootc operating system image for the Red Hat Edge Manager](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-assembly_edge_mgr_build_images).

## How this differs from simplified `ImageBuild`

| | Simplified (`testbuild1`) | Advanced (`advancedbuild1`) |
| --- | --- | --- |
| Custom RPMs / Containerfile | No | Yes |
| Build mechanism | RHEM image builder worker | `podman build` on an entitled RHEL host |
| Agent + binding | Worker adds automatically | You install `flightctl-agent` in the Containerfile; optional `config.yaml` for early binding |
| Push | Worker → OCI `Repository` | `podman push` to the same lab registry |
| QCOW2 disk | `ImageExport` on the gateway | `bootc-image-builder` on the build host (`qcow2` tag in Ansible) |
| RHEM afterward | Catalog via `ImagePromotion` | Software Catalog via `CatalogItem` (`flightctl apply`); Ansible tag `promotion` |

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
| Catalog item | `advancedbuild1` in `default` | Ansible `promotion` tag (`rhem_advanced_imagebuilds.yml`) |

Example image on the registry: `micro.lan:5000/advancedbuild1:latest`.

## Prerequisites (advanced-specific)

On the **build host** (the RHEM VM is entitled and a practical default):

- RHEL 10 with active subscription and `edge-manager-1.4-for-rhel-10-x86_64-rpms` available (same as the image builder worker).
- **Podman** 5.0+ and **skopeo** 1.14+ (`dnf list installed podman skopeo`).
- `podman login registry.redhat.io` (same credentials as in [README.image-builds-basic.md](README.image-builds-basic.md)).
- Network reachability to `micro.lan:5000` from the build host.
- **`flightctl-cli`** logged in if you use **early binding** (enrollment cert in the image). See [README.rhem-use.md](README.rhem-use.md).

`bootc-image-builder` (QCOW2 and other disk formats) is covered in [Build QCOW2 disk image](#build-qcow2-disk-image). Optional production steps: [Sigstore signing](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/secure-proc_sign_publish_bootc_image_sigstore).

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

## Build QCOW2 disk image

Convert the **local** bootc container (still in rootful Podman storage after `podman build`) into a QCOW2 with Red Hat’s `bootc-image-builder` container. You do **not** need an Image Builder blueprint; **`config.toml` is optional** in general but **used by default in this lab** to add a console login user on the disk.

### Mandatory configuration

| Required | Notes |
| --- | --- |
| Entitled RHEL build host | Same as `podman build` ([prerequisites](#prerequisites-advanced-specific)). |
| `podman login registry.redhat.io` | Pull `registry.redhat.io/rhel10/bootc-image-builder`. |
| Bootc image in **local** storage | `bootc-image-builder` mounts `/var/lib/containers/storage`; it does not pull `micro.lan:5000` by itself. Build (or `podman pull`) on this host before export. |
| Privileged Podman + SELinux | Use `--privileged`, `--security-opt label=type:unconfined_t`, and `:Z` volume mounts. |
| **Not required** | RHEM `ImageExport`, Composer blueprint, or extra RHEM manifests for the disk file. |

Product context: [Build for specific target platforms](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_edge_mgr_build_images_platform). RHEL 10 procedure: [Creating bootc-compatible base disk images](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index).

Default lab `config.toml` (replace the password; do not commit real secrets):

```toml
[[customizations.user]]
name = "labuser"
password = "changeme"
groups = ["wheel"]
```

Working directory on the build host (matches Ansible `rhem_advanced_build_dir`):

```bash
cd /var/tmp/advancedbuild1   # or /etc/flightctl/advancedbuilds/advancedbuild1 when using Ansible
export OCI_IMAGE_REPO=micro.lan:5000/advancedbuild1
export OCI_IMAGE_TAG=latest

mkdir -p output
sudo tee config.toml <<'EOF'
[[customizations.user]]
name = "labuser"
password = "changeme"
groups = ["wheel"]
EOF

sudo podman pull registry.redhat.io/rhel10/bootc-image-builder:latest

sudo podman run --rm --privileged --pull=newer \
  --security-opt label=type:unconfined_t \
  -v /var/lib/containers/storage:/var/lib/containers/storage:Z \
  -v "${PWD}/config.toml:/config.toml:Z" \
  -v "${PWD}/output:/output:Z" \
  registry.redhat.io/rhel10/bootc-image-builder:latest \
  --type qcow2 --config /config.toml \
  "${OCI_IMAGE_REPO}:${OCI_IMAGE_TAG}"
```

Ansible stores the QCOW2 under `${rhem_advanced_build_dir}/output/qcow2/disk.qcow2` and reads the user password from vault (`rhem_advanced_qcow2_user_password`).

Verify:

```bash
dnf list installed podman
sudo podman images registry.redhat.io/rhel10/bootc-image-builder
test -f output/qcow2/disk.qcow2
```

## After push: Software Catalog (`promotion` tag)

Advanced images are **not** tied to an `ImageBuild`, so you cannot use `ImagePromotion`. Register the pushed OCI reference with a `CatalogItem` instead (same shape as a promoted worker build). Ansible applies the manifest when you run `playbooks/rhem-image-builds-advanced.yml` with tag **`promotion`** (or a full playbook run after `push`).

Create `catalogitem-advancedbuild1.yaml`:

```yaml
apiVersion: flightctl.io/v1alpha1
kind: CatalogItem
metadata:
  name: advancedbuild1
  catalog: default
spec:
  type: os
  category: system
  displayName: Lab RHEL 10 advanced image
  artifacts:
    - type: container
      uri: micro.lan:5000/advancedbuild1
  versions:
    - version: "1.0.0"
      channels:
        - testing
      references:
        container: latest
```

```bash
flightctl apply -f catalogitem-advancedbuild1.yaml
flightctl get catalogitem advancedbuild1 --catalog default
flightctl get catalogitems --catalog default
```

See [Managing a catalog repository on the CLI](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-managing_a_catalog_repository_on_the_cli) and [Catalog item reference rendering](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/deploy-con_catalog_reference_rendering).

## After catalog registration: fleets and devices

Reference the catalog item (for example `catalogItemRef`) in fleet or device specs. Provision test VMs from the QCOW2 built in [Build QCOW2 disk image](#build-qcow2-disk-image); the Software Catalog item remains the **container** reference only unless you add a separate disk packaging flow.

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
sudo podman images registry.redhat.io/rhel10/bootc-image-builder
curl -kfsS "https://micro.lan:5000/v2/advancedbuild1/tags/list"
test -f /etc/flightctl/advancedbuilds/advancedbuild1/output/qcow2/disk.qcow2
flightctl get catalogitem advancedbuild1 --catalog default
flightctl get catalogitems --catalog default
```

On a device or test VM booted from the image (after enrollment): confirm packages and Cockpit socket:

```bash
rpm -q tmux bash-completion cockpit flightctl-agent
systemctl is-enabled flightctl-agent.service cockpit.socket
```

## References

- [Choosing an image building path](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-con_edge_mgr_images_intro)
- [Build the operating system image with bootc](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_build_bootc_image)
- [Build for specific target platforms](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_edge_mgr_build_images_platform)
- [Sign and publish bootc image (Sigstore)](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/secure-proc_sign_publish_bootc_image_sigstore)
- [Simplified worker flow (this repo)](README.image-builds-basic.md)
- [RHEM CLI login (this repo)](README.rhem-use.md)