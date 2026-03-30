# 80-keycloak

This playbook stream installs and bootstraps a private Keycloak instance on the `oidc` VM using the upstream Keycloak tarball, OpenJDK 21, a local PostgreSQL service, and a native systemd unit.

## Files

- `playbooks/80-keycloak.yaml`
  Wrapper playbook that imports the split Keycloak playbooks.

- `playbooks/keycloak/00_preflight.yaml`
  Validates required inventory and variables before installation.

- `playbooks/keycloak/10_install_runtime.yaml`
  Installs OpenJDK, PostgreSQL, the Keycloak service user, and local directories.

- `playbooks/keycloak/20_deploy_stack.yaml`
  Downloads Keycloak, configures PostgreSQL, renders the Keycloak environment and systemd unit, and starts the services.

- `playbooks/keycloak/30_bootstrap_realm.yaml`
  Creates the Keycloak realm and the initial admin group.

- `playbooks/keycloak/40_configure_vault_client.yaml`
  Creates the OIDC client for Vault and the groups claim mapper.

- `playbooks/keycloak/90_verify.yaml`
  Verifies readiness and prints the discovery URL details needed by Vault.

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
- The target VM provides the `{{ keycloak_java_package | default('openjdk-21-jre-headless') }}` package in its apt repositories.
- The initial deployment uses internal HTTP on port `8080`.
- Vault OIDC integration will later consume the Keycloak discovery URL printed by the verify step.

## Follow-up

After Keycloak is deployed and verified:

1. Update `vars/vault-human-auth.yaml` with the Keycloak discovery URL, client ID, and client secret.
2. Run the Vault human-auth playbooks through `playbooks/90-wireguard.yaml` or the split playbooks under `playbooks/wireguard/`.
3. Later harden Keycloak with private TLS if you do not want internal HTTP for steady state.