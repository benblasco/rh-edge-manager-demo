# Red Hat Edge Manager UI and CLI use

## CLI login

On the RHEM host (or anywhere you’ve installed flightctl-cli and can reach the gateway):

```bash
flightctl login https://rhem.hex.blasco.id.au --username user1 --password '<your-rhem_pam_password>'
```

Use the same PAM issuer user as the UI (rhem_pam_username in rhem.yml, default user1), not a local Linux account.

If the client doesn’t trust the builtin CA (likely scenario in a lab):

```bash
flightctl login https://rhem.hex.blasco.id.au --username user1 --password '<password>' --insecure-skip-tls-verify
That matches what the playbook does when rhem_flightctl_login_insecure_tls: true.
```

# Web UI login

For your lab host, the UI is:

[](https://rhem.hex.blasco.id.au/)

Sign in with the PAM issuer user (default playbook vars: `user1` and the password in `vault.yml` as `rhem_pam_password`), not a RHEL host account.

If you change `rhem_fqdn` in `ansible/inventory/group_vars/rhem_servers/rhem.yml`, use `https://<that-fqdn>/`.