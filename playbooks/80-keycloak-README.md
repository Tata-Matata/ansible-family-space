# 80-keycloak

This playbook stream installs and bootstraps a private Keycloak instance on the `oidc` VM using the upstream Keycloak tarball, OpenJDK 21, a local PostgreSQL service, and a native systemd unit.

It currently runs in two phases:

1. Deploy Keycloak without TLS, bootstrap the realm, configure the Vault OIDC client, and verify basic readiness.
2. Issue the private TLS certificate, re-deploy Keycloak in TLS mode, and verify the TLS-enabled service.

## Files

- `playbooks/80-keycloak.yaml`
  Wrapper playbook that imports the split Keycloak playbooks.

- `playbooks/keycloak/00-preflight.yaml`
  Validates required inventory and variables before installation.

- `playbooks/keycloak/10-install-runtime.yaml`
  Installs OpenJDK, PostgreSQL, the Keycloak service user, and local directories.

- `playbooks/keycloak/20-deploy-stack.yaml`
  Downloads Keycloak, configures PostgreSQL, renders the Keycloak environment and systemd unit, and starts the services.

- `playbooks/keycloak/30-bootstrap-realm.yaml`
  Creates the Keycloak realm and the initial admin group.

- `playbooks/keycloak/40-configure-vault-client.yaml`
  Creates the OIDC client for Vault and the groups claim mapper.

- `playbooks/keycloak/50-verify.yaml`
  Verifies non-TLS readiness and prints the discovery URL details needed by Vault.

- `playbooks/keycloak/60-configure-tls.yaml`
  Issues a private TLS certificate for Keycloak from the existing internal CA on Bastion, installs it on the OIDC VM, and refreshes CA trust.

- `playbooks/keycloak/70-redeploy-stack-tls.yaml`
  Re-deploys the Keycloak service configuration after TLS material is installed.

- `playbooks/keycloak/80-verify-tls.yaml`
  Verifies TLS-enabled readiness and discovery after Keycloak is restarted with HTTPS.

## Variables

Non-secret configuration lives in:

- `vars/keycloak.yaml`

Secrets live in:

- `vars/keycloak-secrets.yaml`

Required secrets:

- `keycloak_postgres_password`
- `keycloak_admin_password`
- `keycloak_vault_client_secret`

## Encrypt Secrets With Ansible Vault

1. Edit `vars/keycloak-secrets.yaml` and set real values.
2. Encrypt the file:

```bash
cd /ansible/project/dir/root
ansible-vault encrypt vars/keycloak-secrets.yaml
```

3. Edit later with:

```bash
cd /home/tati/projects/infra/ansible
ansible-vault edit vars/keycloak-secrets.yaml
```

## Run

Run the wrapper playbook and let Ansible prompt for the Vault password used to decrypt `vars/keycloak-secrets.yaml`:

```bash
cd /home/tati/projects/infra/ansible
ansible-playbook playbooks/80-keycloak.yaml --ask-vault-pass
```

## Current Assumptions

- The `oidc` VM already exists in inventory and is reachable by Ansible.
- Keycloak is private-only and intended to be reached over the private network or VPN.
- The existing internal CA on Bastion has already been created, because Keycloak TLS reuses it to issue the OIDC server certificate.
- The target VM provides the `{{ keycloak_java_package | default('openjdk-21-jre-headless') }}` package in its apt repositories.
- Before the TLS phase, Keycloak serves browser and admin CLI traffic over HTTP on port `8080`.
- After the TLS phase, Keycloak serves browser and admin CLI traffic over private HTTPS on port `8443`, while the management endpoint remains on port `9000`.
- Vault OIDC integration will later consume the Keycloak discovery URL printed by the verify step.

## Follow-up

After Keycloak is deployed and verified:

1. Update `vars/vault-human-auth.yaml` with the Keycloak discovery URL, client ID, and client secret.
2. Run the Vault human-auth playbooks through `playbooks/90-wireguard.yaml` or the split playbooks under `playbooks/wireguard/`.
3. Trust the internal CA certificate on the browser clients that will reach the private OIDC service over WireGuard.