# TODO

## Infrastructure Improvements

### Bastion Security Hardening
- [ ] Refactor `playbooks/01-bastion-security-hardening.yaml` into a reusable role
  - Consider incorporating `playbooks/01-add-known-hosts-to-bastion.yaml` into the role as well
  - Extract SSH hardening tasks to `roles/bastion_security/tasks/ssh.yaml`
  - Extract iptables INPUT chain rules to `roles/bastion_security/tasks/firewall.yaml`
  - Extract Fail2Ban setup to `roles/bastion_security/tasks/fail2ban.yaml`
  - Extract port/upgrade checks to `roles/bastion_security/tasks/verification.yaml`
  - Create defaults for configurable options (SSH port, allowed ports, etc.)
  - Document in `roles/bastion_security/README.md`
  - Update playbook to simply call the role

### TLS File Permissions Config
- [ ] Refactor duplicated file/directory permissions from ensure and assert tasks
  - Extract permissions definitions to defaults/variables for reusability
  - Reduce duplication in file ownership/mode specifications across:
    - `playbooks/41-distribute-tls-to-bastions.yaml` (ensure + assert tasks)
    - `tasks/bootstrap-tls-vault-consul/31_distribute_tls_consul.yaml` (ensure + assert tasks)
    - `tasks/bootstrap-tls-vault-consul/30_distribute_tls_vault.yaml` (ensure + assert tasks)
  - Create a centralized source of truth for TLS file permissions
  - Makes future permission changes simpler and less error-prone

### Bootstrap Secret Migration
- [ ] Move bootstrap secrets from Ansible Vault files into HashiCorp Vault after the platform is fully operational
  - Start with Keycloak bootstrap secrets currently stored in `vars/keycloak-secrets.yaml`
  - Define which secrets remain deployment-time only versus runtime-managed in Vault
  - Update playbooks to read steady-state secrets from HashiCorp Vault instead of repo-managed encrypted files where appropriate
  - Remove or minimize long-term reliance on repo-stored bootstrap secrets once the Vault-based secret flow is proven

### Vault Automation Auth
- [ ] Revisit how Ansible authenticates to Vault for steady-state automation
  - Avoid relying long-term on a persisted bearer token on file system on Vault host
  - Evaluate a machine-oriented auth method such as AppRole, TLS certificate auth, or another non-human Vault auth flow
  - AppRole flow: store Role ID plus Secret ID, authenticate to Vault with them, and let Vault issue a token carrying the policies attached to that AppRole
  - TLS certificate auth flow: present a client certificate to Vault, let Vault map that certificate identity to policies, and let Vault issue a token for that machine identity
  - In both designs, policy still exists; the difference is that automation obtains tokens dynamically through a machine auth method instead of reusing one persisted long-lived token directly
  - Define the rotation and recovery model for automation credentials after the initial bootstrap phase

## Developer Workflow

- [ ] Add `lefthook` with a Bash validation script for Ansible playbooks
  - Run syntax and YAML validation before commit
  - Scope checks to playbooks and related Ansible files when possible
  - Keep the hook implementation in a plain Bash script committed to the repo
  - Document local setup and expected hook behavior
