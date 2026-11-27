# Image builds using the RHEM image build worker

Connected RPM control plane on RHEL 10 for host `rhem.hex.blasco.id.au`. This procedure configures the image builder worker for subscription-managed RPM content, registers OCI repositories for base and output images, and runs a simplified `ImageBuild` from the CLI.

**Requires a working RHEM install:** [README.rhem-install-rpm.md](README.rhem-install-rpm.md). **CLI login:** [README.rhem-use.md](README.rhem-use.md).

**Automated setup (worker, OCI repos, ImageBuild, QCOW2 `ImageExport`, catalog promotion):** [ansible/README.md](ansible/README.md) — playbook `ansible/playbooks/rhem-image-builds.yml` and Jinja2 manifests under `ansible/templates/imagebuilds/`. Lab defaults publish to the Software Catalog via `ImagePromotion` after the build completes (`rhem_imagebuild_promotion_enabled: true` in `rhem_imagebuilds.yml`) and optionally export QCOW2 (`rhem_imageexport_enabled` in the same file).

Official guides:

- [Operating system images](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-con_edge_mgr_images_intro)
- [Configure the image builder worker for subscription-managed RPM content](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_configuring_imagebuilder_worker_subscription_rhel)
- [Configuring an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository)
- [Build images with the simplified workflow](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_edge_mgr_build_images_simple)

## Simplified workflow (capabilities and limits)

This document uses the **simplified** path only. See [Choosing an image building path](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-con_edge_mgr_images_intro).

**Capabilities**

- Pull a trusted `rhel-bootc` base from a configured OCI `Repository`, build on the **image builder worker**, push to a **ReadWrite** destination registry.
- Install and enable **`flightctl-agent`** and required system services automatically (no Containerfile in this flow).
- **Binding:** `early` embeds enrollment endpoint + shared enrollment certificate in the image at build time; `late` leaves enrollment for provisioning (`cloud-init`, Ignition, kickstart). See [Enrollment methods](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_enrollment_methods).
- **Software Catalog** publication (`ImagePromotion`) after a successful build so fleets can reference a catalog item; build status and logs via UI or `flightctl`.
- Optional **disk export** (`ImageExport`) for ISO, QCOW2, or VMDK artifacts.

**Limitations**

- **No custom OS layers** — no Containerfile, extra RPMs, drivers, or manual signing in `ImageBuild`; use the [advanced bootc build](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-assembly_edge_mgr_build_images) and push your own image instead. Lab walkthrough: [README.image-builds-advanced.md](README.image-builds-advanced.md).
- **Early binding** does not approve enrollment; devices still need an approved enrollment request. The enrollment cert only secures the enrollment *request* ([Enrollment methods](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_enrollment_methods)).
- **`ImageBuild` is immutable** — new image versions need a new resource or `flightctl new-version` ([simplified workflow](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_edge_mgr_build_images_simple)).
- Registry access for base and push is defined on **OCI `Repository`** resources, not host `podman login` alone.

## Prerequisites

- RHEM 1.4 installed and `flightctl.target` active ([install guide](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-con_installing_on_rhel))
- Image builder worker enabled (shipped with `flightctl-services`; unit `flightctl-imagebuilder-worker.service`)
- Active RHEL subscription on the RHEM host with `edge-manager-1.4-for-rhel-10-x86_64-rpms` enabled
- `flightctl-cli` installed and logged in to `https://rhem.hex.blasco.id.au`
- `registry.redhat.io` account (same credentials used at install for `podman login`) — **read** access for base `rhel-bootc` images
- Lab output registry `micro.lan:5000` reachable from the RHEM host — **read/write** for pushed builds (plain [Docker Distribution registry](https://hub.docker.com/_/registry))

Registry authentication during a build is configured on **OCI `Repository` resources**, not only on the host’s `podman login`. See [Configuring an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository).

Builder service images (`podman`, `bootc-image-builder`, `syft`) on a **private mirror** need a separate `auth.json` mount on the worker. Connected installs pulling from Red Hat defaults do not need that step. See [Configure registry authentication for builder service images (RHEL)](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/secure-proc_configuring_imagebuilder_registry_auth_rhel).

## Variables used in this lab

| Variable | Lab value | Notes |
| --- | --- | --- |
| `RHEM_FQDN` | `rhem.hex.blasco.id.au` | Gateway / API |
| `RHSM_REPO_ID` | `edge-manager-1.4-for-rhel-10-x86_64-rpms` | Worker RPM repo enablement |
| `BASE_REGISTRY` | `registry.redhat.io` | Red Hat bootc base images |
| `BASE_IMAGE` | `rhel10/rhel-bootc` | Image name under the base registry |
| `BOOTC_TAG` | `1795492183` | Pinned lab base tag (`podman pull registry.redhat.io/rhel10/rhel-bootc:1795492183`) |
| `BUILT_REGISTRY` | `micro.lan:5000` | Private lab registry (DNS); no auth |
| `BUILD_NAME` | `testbuild1` | `ImageBuild` metadata name |
| `DEST_IMAGE` / `DEST_TAG` | `testbuild1` / `latest` | Pushed as `micro.lan:5000/testbuild1:latest` when the OCI repository has no `repository` or `namespace` override |
| OCI repo resource names | `registry-redhat-io`, `micro-lan-built` | Names in `flightctl apply` manifests |
| `PROMOTION_NAME` | `testbuild1-promotion` | `ImagePromotion` metadata name |
| `CATALOG_NAME` / `CATALOG_ITEM` | `default` / `testbuild1` | Target Software Catalog and item name |
| `CATALOG_DISPLAY_NAME` | Lab RHEL 10 early-binding image | UI label for the catalog item |
| `CATALOG_ITEM_VERSION` | `1.0.0` | Catalog item version string |
| `IMAGE_EXPORT_NAME` | `testbuild1-qcow2` | `ImageExport` metadata name (QCOW2 disk) |

Replace registry credentials in the YAML below with your values (for example the same `registry.redhat.io` user as in `ansible/inventory/group_vars/rhem_servers/vault.yml`). Do not commit real passwords to git.

Non-secret image-build defaults live in `ansible/inventory/group_vars/rhem_servers/rhem_imagebuilds.yml`. ImageBuild and ImageExport are applied from the controller via `flightctl.core` (see `ansible/README.md` for the vault refresh token). The Ansible playbook uses **HTTPS** and `skipServerVerification: true` for `micro.lan:5000` per those vars; manual examples below may show plain HTTP where that matches an older lab setup—align `scheme` on the OCI `Repository` with your registry.

## Mount subscription data into the image builder worker

Product procedure: [Configuring on RHEL with Podman Quadlets](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_configuring_imagebuilder_worker_subscription_rhel#edge-manager-config-imagebuilder-rhel-podman__edge-manager-config-imagebuilder-podman).

On the RHEM host:

```bash
sudo mkdir -p /etc/containers/systemd/flightctl-imagebuilder-worker.container.d
sudo tee /etc/containers/systemd/flightctl-imagebuilder-worker.container.d/subscription.conf <<'EOF'
[Container]
Volume=/etc/pki/entitlement:/etc/pki/entitlement:ro,z
Volume=/etc/yum.repos.d:/etc/yum.repos.d:ro,z
Volume=/etc/rhsm/:/etc/rhsm/:ro,z
EOF
```

Edit `imagebuilderWorker` in `/etc/flightctl/service-config.yaml` (merge into the existing file; do not replace the full file). Use the RHSM repository ID that matches your RHEL major version and architecture:

```yaml
imagebuilderWorker:
  rpmRepoAdd: false
  rpmRepoEnable: edge-manager-1.4-for-rhel-10-x86_64-rpms
```

Reload and restart the worker:

```bash
sudo systemctl daemon-reload
sudo systemctl restart flightctl-imagebuilder-worker.service
sudo systemctl is-active flightctl-imagebuilder-worker.service
```

Troubleshooting: `sudo subscription-manager status`, `sudo journalctl -u flightctl-imagebuilder-worker.service -b --no-pager`. See the worker doc for `RLIMIT_NOFILE` and subscription errors.

## Flight Control CLI session

From a host with `flightctl-cli` and network access to the gateway:

```bash
flightctl login https://rhem.hex.blasco.id.au --username user1 --password '<password>' --insecure-skip-tls-verify
flightctl version
```

## Configure OCI repositories

RHEM does not use ad hoc `podman pull` paths for builds. You must create **`Repository` resources** (kind `Repository`, `spec.type: oci`) for the registries you reference in `ImageBuild`. Base pulls use `accessMode: Read`; the build output registry uses `ReadWrite`. See [Configuring an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository).

### Base image registry (`registry.redhat.io`)

Yes — register Red Hat’s registry as an OCI repository so the worker can pull `rhel10/rhel-bootc` with the credentials you supply in `spec.ociAuth`. Optional `spec.baseImages` improves the UI picker; the CLI can set `imageName` and `imageTag` on the `ImageBuild` directly. See [Configure trusted base images on an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_configure_oci_base_images).

Confirm the pinned base tag is pullable (optional):

```bash
podman login registry.redhat.io
podman pull registry.redhat.io/rhel10/rhel-bootc:1795492183
```

Create `repository-redhat-base.yaml`:

```yaml
apiVersion: flightctl.io/v1beta1
kind: Repository
metadata:
  name: registry-redhat-io
spec:
  type: oci
  registry: registry.redhat.io
  scheme: https
  accessMode: Read
  ociAuth:
    authType: docker
    username: <registry.redhat.io_username>
    password: <registry.redhat.io_password_or_token>
  baseImages:
    - imageName: rhel10/rhel-bootc
      tags:
        - "1795492183"
```

Apply and verify:

```bash
flightctl apply -f repository-redhat-base.yaml
flightctl get repository/registry-redhat-io
```

### Built image registry (`micro.lan:5000`)

Private lab registry on `micro.lan:5000` ([Docker Distribution `registry`](https://hub.docker.com/_/registry)): no authentication, reachable from the RHEM host by DNS. Use `scheme: http` if the registry serves plain HTTP on port 5000 (typical for `registry:2`). If you terminate TLS in front of the registry instead, set `scheme: https`, omit `ociAuth`, and set `skipServerVerification: true` for this lab (not for production). See [Configuring an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository).

Create `repository-micro-lan.yaml` (HTTP example):

```yaml
apiVersion: flightctl.io/v1beta1
kind: Repository
metadata:
  name: micro-lan-built
spec:
  type: oci
  registry: micro.lan:5000
  scheme: http
  accessMode: ReadWrite
```

Apply and verify:

```bash
flightctl apply -f repository-micro-lan.yaml
flightctl get repository/micro-lan-built
```

Confirm the RHEM host can reach the registry (from the RHEM host):

```bash
curl -sS "http://micro.lan:5000/v2/" || true
```

If your registry uses authentication or HTTPS, add `spec.ociAuth` and/or `spec.ca.crt` per [Configuring an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository).

## Early binding (enrollment credentials)

This lab uses **early binding**: the simplified build embeds enrollment settings in the image so devices can request enrollment without cloud-init injection. See [Enrollment methods](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_enrollment_methods).

For the simplified CLI workflow, set `spec.binding.type: early` on the `ImageBuild`; the image builder worker generates and embeds the agent configuration during the build. You do not need a separate `config.yaml` step unless you use the advanced Containerfile workflow. Background and manual certificate export: [Optional: Requesting an enrollment certificate for early binding](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_request_enrollment_cert_early_binding).

Devices still need **enrollment request approval** after boot ([Enroll devices](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/get_started-con_enrolling_devices)).

## Build images with the simplified workflow (CLI)

Product procedure: [Creating an image build from the CLI](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_edge_mgr_build_images_simple#creating-an-image-build-from-the-cli_edge-manager-simple-image-workflow).

Create `imagebuild-testbuild1.yaml`:

```yaml
apiVersion: flightctl.io/v1alpha1
kind: ImageBuild
metadata:
  name: testbuild1
spec:
  source:
    repository: registry-redhat-io
    imageName: rhel10/rhel-bootc
    imageTag: "1795492183"
  destination:
    repository: micro-lan-built
    imageName: testbuild1
    imageTag: latest
  binding:
    type: early
```

Apply, monitor, and follow logs:

```bash
flightctl apply -f imagebuild-testbuild1.yaml
flightctl get imagebuild/testbuild1
flightctl logs imagebuild/testbuild1 -f
```

When `status` shows completion, confirm the image on the lab registry (from a host that resolves `micro.lan`):

```bash
curl -sS "http://micro.lan:5000/v2/testbuild1/tags/list"
# or: podman pull micro.lan:5000/testbuild1:latest
```

Build lifecycle and retries: [Build status and logs](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_edge_mgr_build_images_simple#build-status-and-logs_edge-manager-simple-image-workflow).

## Export QCOW2 disk image (ImageExport)

You do **not** need a `config.toml`, osbuild blueprint, or changes to the `ImageBuild` manifest for a standard QCOW2 export. The image builder worker runs the export after you create an **`ImageExport`** resource that references the completed build.

### Mandatory configuration

| Required | Notes |
| --- | --- |
| Completed `ImageBuild` (`testbuild1`) | Export tasks fail if the build is not `Ready=Completed`. |
| Image builder worker active | Same worker used for the build ([mount subscription data](#mount-subscription-data-into-the-image-builder-worker)). |
| OCI `Repository` resources | Already required for the build; no extra registry for QCOW2 download. |
| **Not required** | `config.toml`, Composer blueprint, or `exportFormats` on `ImageBuild`. |

Product procedure: [Build images with the simplified workflow — Image export](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_edge_mgr_build_images_simple).

After `testbuild1` completes, create `imageexport-testbuild1-qcow2.yaml`:

```yaml
apiVersion: flightctl.io/v1alpha1
kind: ImageExport
metadata:
  name: testbuild1-qcow2
spec:
  source:
    type: imageBuild
    imageBuildRef: testbuild1
  format: qcow2
```

Apply, monitor, and download:

```bash
flightctl apply -f imageexport-testbuild1-qcow2.yaml
flightctl get imageexport/testbuild1-qcow2
flightctl logs imageexport/testbuild1-qcow2 -f
flightctl download imageexport/testbuild1-qcow2 ./disk.qcow2
test -f ./disk.qcow2
```

Other disk formats use the same manifest with `format: iso` or `format: vmdk`. Export can take several minutes after the container build finishes.

**Order with catalog promotion:** If the Software Catalog item should include a QCOW2 artifact, complete the `ImageExport` first, then apply `ImagePromotion` with `exportFormats: [qcow2]` (see below). Promotion may remain `WaitingForArtifacts` until the export succeeds.

## Publish to Software Catalog

The CLI does not mirror the UI “Publish to Software Catalog” toggle at create time. **After `testbuild1` completes**, publish with an `ImagePromotion` so fleets and rollouts can use a catalog item reference. See [Publish from the CLI](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_image_build_catalog_cli) and [Image promotion and catalog publication](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_image_promotion).

```bash
flightctl get imagebuild/testbuild1
flightctl get catalog/default
```

Create `imagepromotion-testbuild1.yaml`:

```yaml
apiVersion: flightctl.io/v1alpha1
kind: ImagePromotion
metadata:
  name: testbuild1-promotion
spec:
  source:
    imageBuildRef: testbuild1
    exportFormats:
      - qcow2
  target:
    type: NewCatalogItem
    catalogName: default
    catalogItemName: testbuild1
    displayName: Lab RHEL 10 early-binding image
    version: "1.0.0"
```

Create `ImageExport` for `qcow2` **before** applying this promotion if you use `exportFormats: [qcow2]` ([Publish from the CLI](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_image_build_catalog_cli)).

To publish only the bootc container reference, omit `exportFormats` entirely (lab Ansible default when QCOW2 catalog artifacts are disabled).

Apply and wait until `status.phase` is `Completed`:

```bash
flightctl apply -f imagepromotion-testbuild1.yaml
flightctl get imagepromotion/testbuild1-promotion -o yaml
flightctl get catalogitems --catalog default
```

On this lab gateway (1.4.1), `flightctl apply` rejects `exportFormats: [container]` even though the product docs show it—omit `container` or list only disk formats after matching `ImageExport` resources exist (`qcow2`, `iso`, `vmdk`).

## Verification

```bash
dnf list installed flightctl-cli
sudo systemctl is-active flightctl-imagebuilder-worker.service
flightctl get repository
flightctl get imagebuild/testbuild1
flightctl get imageexport/testbuild1-qcow2
flightctl get imagepromotion/testbuild1-promotion
flightctl get catalogitems --catalog default
curl -kfsS "https://micro.lan:5000/v2/testbuild1/tags/list"
test -f ./disk.qcow2
```

## References

- [Operating system images](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-con_edge_mgr_images_intro)
- [Configure the image builder worker for subscription-managed RPM content](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_configuring_imagebuilder_worker_subscription_rhel)
- [Configuring an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-configuring_an_oci_repository)
- [Configure trusted base images on an OCI repository](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_configure_oci_base_images)
- [Build images with the simplified workflow](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-con_edge_mgr_build_images_simple)
- [Enrollment methods](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_enrollment_methods)
- [Optional: Requesting an enrollment certificate for early binding](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_request_enrollment_cert_early_binding)
- [Publish from the CLI](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-proc_image_build_catalog_cli)
- [Configure registry authentication for builder service images (RHEL / Quadlets)](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/secure-proc_configuring_imagebuilder_registry_auth_rhel)
- [RHEM install (this repo)](README.rhem-install-rpm.md)
- [RHEM CLI login (this repo)](README.rhem-use.md)

Disconnected/air-gapped image builder configuration is out of scope. See [Configure the image builder worker in disconnected environments](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/install-proc_configure_imagebuilder_air_gapped_rhel).