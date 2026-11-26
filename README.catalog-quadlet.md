# Software Catalog: quadlet application (Grafana demo)

Register an **application** catalog item with `spec.type: quadlet` so fleets deploy Grafana through Podman Quadlet (systemd) instead of a raw single-container app type. This lab item mirrors the manual demo:

```bash
podman run -d --rm --name grafana-demo -p 3000:3000 docker.io/grafana/grafana:latest
```

Catalog registration only. Assigning the item to a fleet or device is a separate step.

## Quadlet vs container catalog type

| Field | OS image catalog (this repo) | Grafana (this doc) |
|-------|------------------------------|--------------------|
| `spec.type` | `os` | `quadlet` |
| `spec.category` | `system` | `application` |
| Artifact | bootc container in your registry | `docker.io/grafana/grafana` |
| Fleet `appType` | N/A (OS slot) | `quadlet` (must match catalog `type`) |

The workload image is still declared as artifact `type: container` with a versionless `uri`; the catalog version supplies the tag in `references.container`.

## Prerequisites

- RHEM installed and reachable over HTTPS from the Ansible controller (`https://` + your `rhem_fqdn`).
- After [ansible/playbooks/rhem-install-rpm.yml](ansible/playbooks/rhem-install-rpm.yml): PAM user credentials in `ansible/inventory/group_vars/rhem_servers/vault.yml`.
- Controller: Python 3.12+, Galaxy collection `flightctl.core`, and pip packages in [ansible/requirements-controller.txt](ansible/requirements-controller.txt) (see [ansible/README.md](ansible/README.md)).
- Optional on the RHEM host: `flightctl login` and `flightctl get catalog/default` (every org gets a `default` catalog).

## Manifest in this repo

[`catalog/applications/grafana-demo/catalogitem.yaml`](catalog/applications/grafana-demo/catalogitem.yaml) defines:

- Catalog item name: `grafana-demo` in catalog `default`
- Version `1.0.0` on channel `testing`, OCI tag `latest`
- `defaults.config.runAs: root` (quadlet catalog defaults allow `envVars`, `runAs`, `volumes` only)
- Port `3000:3000` and quadlet unit content belong in the **fleet/device** application spec when you deploy (separate task)

## Publish to the Software Catalog

### Ansible (recommended for this lab)

After [ansible/playbooks/rhem-install-rpm.yml](ansible/playbooks/rhem-install-rpm.yml):

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
python3 -m pip install -r requirements-controller.txt
ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --ask-vault-pass
```

The catalog playbook runs API modules on **localhost** using `rhem_flightctl_connection.yml` and the repo manifest file (no manifest copy to the gateway). See [ansible/README.md](ansible/README.md) for tags and variables (`rhem_catalog_grafana.yml`, `rhem_flightctl_connection.yml`).

### Manual (`flightctl` on the RHEM host)

```bash
flightctl get catalog/default
flightctl apply -f catalog/applications/grafana-demo/catalogitem.yaml
flightctl get catalogitem grafana-demo --catalog default
flightctl get catalogitems --catalog default
```

Confirm in the Flight Control UI under **Software Catalog** → **Grafana demo (quadlet)**.

### Verification

Controller (repeat apply should report no change when the spec already matches):

```bash
cd ansible
ansible-playbook playbooks/rhem-catalog-grafana-demo.yml --tags verify --ask-vault-pass
python3 --version
```

RHEM gateway host (optional CLI):

```bash
flightctl get catalogitem grafana-demo --catalog default -o yaml
```

## Fleet (Ansible)

After the catalog item exists, apply a demo fleet from the Ansible controller (no `flightctl apply` on the gateway):

```bash
cd ansible
ansible-playbook playbooks/rhem-fleet-grafana-demo.yml --ask-vault-pass
```

Variables: `inventory/group_vars/rhem_servers/rhem_fleet_grafana.yml`. The manifest shape matches the manual example below (`appType` must be `quadlet`).

## Manual fleet manifest (reference)

Reference the catalog item; `appType` must be `quadlet`:

```yaml
spec:
  template:
    spec:
      applications:
        - name: grafana-demo
          appType: quadlet
          catalogItemRef:
            catalog: default
            item: grafana-demo
            version: "1.0.0"
            channel: testing
          ports:
            - "3000:3000"
          inline:
            - path: grafana-demo.container
              content: |
                [Container]
                Image=docker.io/grafana/grafana:latest
                PublishPort=3000:3000

                [Install]
                WantedBy=default.target
```

Devices need outbound access to pull `docker.io/grafana/grafana:latest` unless you change `artifacts[].uri` and the quadlet `Image=` line to a mirror (for example `micro.lan:5000/...`).

## Lab notes

- **`latest` tag**: fine for a home lab; pin a tag or digest for reproducibility.
- **Grafana login**: default UI credentials are `admin` / `admin` (change on first login).
- **Related docs in this repo**: [ansible/README.md](ansible/README.md) (API playbooks), [README.image-builds-advanced.md](README.image-builds-advanced.md), [README.scratchpad.md](README.scratchpad.md).