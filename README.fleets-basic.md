# Fleet: `testbuild1` OS + Grafana from the Software Catalog

Define a **Fleet** on the RHEM gateway that targets devices with a label selector and rolls out:

1. **OS** — catalog item `testbuild1` @ `1.0.0` ([simplified image build + promotion](README.image-builds-basic.md)).
2. **Application** — `grafana-demo` @ `1.0.0` (`spec.type: quadlet`) ([Grafana catalog](README.catalog-quadlet.md)).

You can create and apply the fleet **before any image-mode device exists**. Membership stays empty until a device is enrolled, approved, in **image mode**, and labeled to match `spec.selector.matchLabels`.

## Parameter checklist

| Parameter | Where | Lab value |
| --- | --- | --- |
| Fleet name | `metadata.name` | `lab-testbuild1-grafana` |
| Device selector | `spec.selector.matchLabels` | `fleet: lab-testbuild1-grafana` (label on **devices** in Phase B) |
| OS catalog ref | `spec.template.spec.os.catalogItemRef` | `default` / `testbuild1` / `"1.0.0"` |
| Application | `spec.template.spec.applications[]` | `grafana-demo`, `appType: quadlet`, `catalogItemRef`, `ports` (no `inline` when using `catalogItemRef` on this gateway) |

Grafana `ports` live in the fleet template; quadlet unit content comes from the **catalog item** when you use `catalogItemRef`. Do not add `inline` together with `catalogItemRef` — the API rejects it (`oneOf` schema error). See [README.catalog-quadlet.md](README.catalog-quadlet.md).

Optional for a home lab: `configProviders`, `spec.resources`, `rolloutPolicy` — defer until you run multiple devices.

## Prerequisites

### Gateway (Phase A — no device required)

| Step | Check |
| --- | --- |
| RHEM installed, API reachable | [README.rhem-install-rpm.md](README.rhem-install-rpm.md), `flightctl login` or Ansible `rhem_flightctl_connection.yml` |
| OS in catalog | `testbuild1` in `default` @ `1.0.0` — [README.image-builds-basic.md](README.image-builds-basic.md) or `ansible/playbooks/rhem-image-builds.yml` |
| Grafana in catalog | `grafana-demo` in `default` @ `1.0.0` — [README.catalog-quadlet.md](README.catalog-quadlet.md) or `ansible/playbooks/rhem-catalog-grafana-demo.yml` |

Publish Grafana when ready:

```bash
cd ansible
ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --ask-vault-pass
```

### First device (Phase B — later)

| Step | Check |
| --- | --- |
| Image-mode device | Enrolled, **approved**, `status.capabilities.osMode: image` |
| Labels | `fleet: lab-testbuild1-grafana` (must match selector) |
| Egress | Device can pull `docker.io/grafana/grafana:latest` (or use a mirror in catalog + quadlet `Image=`) |
| Selector hygiene | No second fleet selecting the same device (`MultipleOwners`) |

## Optional: `flightctl` catalog checks (when the gateway is available)

Run these on the **RHEM gateway** (or any host where `flightctl` is logged in to the same API) **before** you apply the fleet—or anytime you want to confirm the Software Catalog matches this fleet template. You do **not** need an enrolled device. If you only use Ansible from the controller, you can skip this section and use the [controller verify playbooks](#controller-alternative-no-gateway-ssh) instead.

### Before you run

1. RHEM API is up (`flightctl login` if the session expired — [README.rhem-use.md](README.rhem-use.md)).
2. OS catalog item came from [README.image-builds-basic.md](README.image-builds-basic.md) (or `ansible/playbooks/rhem-image-builds.yml` with promotion enabled).
3. Grafana catalog item came from [README.catalog-quadlet.md](README.catalog-quadlet.md) (or `ansible/playbooks/rhem-catalog-grafana-demo.yml`).

On the host where you run `flightctl`:

```bash
dnf list installed flightctl-cli
```

### Quick presence check

```bash
flightctl get catalog/default
flightctl get catalogitem testbuild1 --catalog default
flightctl get catalogitem grafana-demo --catalog default
flightctl get catalogitems --catalog default
```

`NotFound` or an empty name in the list means that item is not in the catalog yet—fix that before applying [`fleets/lab-testbuild1-grafana.yaml`](fleets/lab-testbuild1-grafana.yaml) (fleet apply can still succeed, but rollout will fail once devices join).

### Confirm version and type (YAML)

The fleet manifest pins **`testbuild1`** and **`grafana-demo`** at version **`1.0.0`**. Grafana uses catalog channel **`testing`** in `catalogItemRef` (see [`fleets/lab-testbuild1-grafana.yaml`](fleets/lab-testbuild1-grafana.yaml)).

```bash
flightctl get catalogitem testbuild1 --catalog default -o yaml
flightctl get catalogitem grafana-demo --catalog default -o yaml
```

| Catalog item | Expected for this fleet | What to look for in YAML |
| --- | --- | --- |
| `testbuild1` | OS @ `1.0.0` | A published version `1.0.0` after `ImagePromotion` completes; item is the lab RHEL image promoted to `default` |
| `grafana-demo` | Application @ `1.0.0` | `spec.type: quadlet`, `spec.category: application`, version `1.0.0` with channel `testing`, artifact `docker.io/grafana/grafana` (matches [catalog manifest](catalog/applications/grafana-demo/catalogitem.yaml)) |

In the UI: **Software Catalog** should list both items with the versions above.

### If an item is missing

| Item | Remediation |
| --- | --- |
| `testbuild1` | Follow [README.image-builds-basic.md](README.image-builds-basic.md) through image build and `ImagePromotion`, or `cd ansible && ansible-playbook playbooks/rhem-image-builds.yml --ask-vault-pass` |
| `grafana-demo` | `cd ansible && ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --ask-vault-pass`, or `flightctl apply -f catalog/applications/grafana-demo/catalogitem.yaml` on the gateway |

### Controller alternative (no gateway SSH)

From the Ansible controller (API only):

```bash
cd ansible
ansible-playbook playbooks/rhem-image-builds.yml --tags verify --ask-vault-pass
ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --tags verify --ask-vault-pass
```

The image-build verify tag lists `CatalogItem` resources when promotion is enabled; the Grafana verify tag fetches `grafana-demo` over the API.

## Canonical manifest

Repo file: [`fleets/lab-testbuild1-grafana.yaml`](fleets/lab-testbuild1-grafana.yaml).

## Phase A — Prepare the fleet (no image-mode device)

### 1. Confirm catalog items

When the gateway is reachable, run the checks in [Optional: `flightctl` catalog checks](#optional-flightctl-catalog-checks-when-the-gateway-is-available) (or the controller `verify` playbooks in that section). Minimum CLI smoke test:

```bash
flightctl get catalogitem testbuild1 --catalog default
flightctl get catalogitem grafana-demo --catalog default
```

### 2. Apply the fleet

**Manual** (manifest in repo root):

```bash
flightctl apply -f fleets/lab-testbuild1-grafana.yaml
```

**Ansible** (controller API — no `flightctl apply` on the gateway):

```bash
cd ansible
ansible-playbook playbooks/rhem-fleet-grafana-demo.yml --ask-vault-pass
```

Variables: [`ansible/inventory/group_vars/rhem_servers/rhem_fleet_grafana.yml`](ansible/inventory/group_vars/rhem_servers/rhem_fleet_grafana.yml). Template: [`ansible/templates/fleets/lab-testbuild1-grafana.yaml.j2`](ansible/templates/fleets/lab-testbuild1-grafana.yaml.j2).

The playbook applies the fleet over **gateway REST** (`tasks/rhem_flightctl_apply_fleet_rest.yml`) because `flightctl.core` **1.7.0** `flightctl_resource` rejects the same quadlet+catalog shape client-side (see comment in that task file). Tags: `fleet` (apply), `verify` (REST read-back).

Set `rhem_fleet_grafana_enabled: false` to skip the playbook.

### 3. Verify control plane only

```bash
flightctl get fleet lab-testbuild1-grafana -o yaml
flightctl get devices -l 'fleet=lab-testbuild1-grafana'
```

An **empty device list is expected** until Phase B. In the UI, the fleet should exist with **member count 0**.

Repeat Ansible apply with `--tags verify` for a API read-back summary:

```bash
cd ansible
ansible-playbook playbooks/rhem-fleet-grafana-demo.yml --tags verify --ask-vault-pass
```

## Phase B — First image-mode device (when you have hardware or a VM)

Checklist (not automated in this repo unless you add enrollment playbooks later):

1. Provision from **`testbuild1`** — bootc image, disk export ([README.image-builds-basic.md](README.image-builds-basic.md)), or lab install media.
2. Complete enrollment and **approve** the request ([Enrollment methods](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/discover-con_enrollment_methods)).
3. Confirm image mode: `flightctl get device/<name> -o yaml` → `status.capabilities.osMode: image`.
4. Add label **`fleet: lab-testbuild1-grafana`** on the `Device` resource ([selecting devices into a fleet (CLI)](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/configure-select_devices_into_a_fleet_by_using_the_cli)).
5. Watch rollout: `status.updated.status` → `UpToDate`, `status.applicationsSummary.status` → `Healthy`.
6. Open Grafana: `http://<device-ip>:3000` — default login `admin` / `admin` (change on first sign-in).

Dry-run membership after labeling:

```bash
flightctl get devices -l 'fleet=lab-testbuild1-grafana'
```

On the device (RHEL):

```bash
sudo systemctl status grafana-demo.service
curl -fsS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/login
```

## Related documentation

| Topic | Link |
| --- | --- |
| OS catalog (`testbuild1`) | [README.image-builds-basic.md](README.image-builds-basic.md) |
| Grafana quadlet catalog | [README.catalog-quadlet.md](README.catalog-quadlet.md) |
| Ansible fleet playbook | [ansible/README.md](ansible/README.md) |
| Managing device fleets | [Product doc — managing device fleets](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/plan-managing_device_fleets) |
| `catalogItemRef` fields | [Catalog item reference specification](https://docs.redhat.com/en/documentation/red_hat_edge_manager/1.4/deploy-ref_catalog_item_ref) |