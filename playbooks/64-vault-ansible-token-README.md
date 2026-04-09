# 64-vault-ansible-token

This playbook creates a narrower Vault token for Ansible automation so that later setup playbooks do not need to keep using the initial Vault root token.

## Purpose

At Vault initialization time, Vault returns a root token. That token is necessary for bootstrap, but it is too powerful for normal day-to-day automation.

The intended model is:

1. Use the root token for initial bootstrap and emergency recovery.
2. Create a narrower Ansible automation policy and token.
3. Store that narrower token separately.
4. Use the narrower token for playbooks that manage only a limited set of Vault configuration.

Rerunning the playbook rotates the Ansible automation token.

- A fresh token is always created.
- The token file is overwritten with the new token.
- If an older token file existed, the previous token is revoked after the new one is persisted.

This keeps the workflow predictable if the policy definition changes over time.

## Recommended Token Storage

The automation token is stored on the Vault host under the Vault state directory with root-only permissions.

- Token file: `/var/lib/vault-state/ansible-automation.token`
- Permissions: `0600`
- Owner: `root`

This is a pragmatic operational choice because:

- the token stays local to the Vault host
- it follows the same state-storage pattern already used for Vault bootstrap artifacts
- later playbooks can read it without depending on the root token file



## Intended Run Order

The expected sequence is:

1. Initialize and unseal Vault.
2. Run `playbooks/64-vault-ansible-token.yaml` once the root token is available.
3. Optionally run `playbooks/65-revoke-vault-root-token.yaml` once the narrower token has been created and verified.
4. Use the narrower Ansible token for later configuration playbooks such as the SSH CA setup and human auth setup.

## Optional Root Token Revocation

If you want to fully retire the initial bootstrap root token after the narrower automation token is in place, run `playbooks/65-revoke-vault-root-token.yaml`.

That playbook performs a few safety checks before revocation:

1. Vault is initialized and unsealed.
2. The stored Ansible automation token exists and can authenticate.
3. The stored bootstrap root token still authenticates, so the revoke step is meaningful.

After revocation:

- the original root token value may still exist in `init.json`, but it is no longer valid
- `init.json` still matters for unseal-key storage
- future root-level break-glass access must use Vault's root generation workflow rather than the original bootstrap token

## Policy Paths And Why They Are Needed

The policy is rendered from the template at `playbooks/vault-policies/templates/ansible-automation-policy.hcl.j2`.

### `sys/mounts`

Capability: `read`

Why: Ansible checks which secrets engines are already enabled before trying to enable the SSH secrets engine.

### `sys/mounts/ssh`

Capabilities: `create`, `read`, `update`, `sudo`

Why: Ansible may need to enable the SSH secrets engine at the `ssh` mount path.

### `ssh/config/ca`

Capabilities: `create`, `read`, `update`

Why: This path holds the SSH CA configuration for Vault's SSH secrets engine. During bootstrap, Ansible may need to write this path to ask Vault to generate the SSH signing key pair if no CA has been configured yet. During later runs and verification, Ansible reads the same path to confirm the CA already exists and to retrieve the public key that gets distributed to hosts so they can trust user certificates signed by Vault.

### `ssh/roles/human-admin`

Capabilities: `create`, `read`, `update`

Why: Ansible creates and later verifies the SSH signing role used for human administrator certificates.

### `sys/auth`

Capability: `read`

Why: Ansible checks which auth methods are enabled before enabling the OIDC auth backend.

### `sys/auth/oidc`

Capabilities: `create`, `read`, `update`, `sudo`

Why: Ansible may need to enable the OIDC auth method at the `oidc` mount path.

### `auth/oidc/config`

Capabilities: `read`, `update`

Why: Ansible configures Vault to trust the Keycloak OIDC provider.

### `auth/oidc/role/human-admin`

Capabilities: `create`, `read`, `update`

Why: Ansible creates and later verifies the Vault OIDC role used for human administrator login.

### `sys/policies/acl/human-ssh`

Capabilities: `create`, `read`, `update`

Why: Ansible creates and later verifies the policy that allows authenticated humans to request SSH certificates for the allowed SSH role.

## What This Token Is For

This token is intended for:

- enabling the SSH secrets engine if missing
- generating or reading the Vault SSH CA
- enabling Vault OIDC auth if missing
- configuring the OIDC backend
- writing the human SSH policy
- writing the human SSH role
- writing the human OIDC role
- reading those same objects during verification

## What This Token Is Not For

This token is not meant to replace Vault root-level privileges for all possible administrative tasks.

It should not be treated as a permanent superuser credential.

If you keep the initial root token, it should be limited to bootstrap and break-glass recovery.

If you run `playbooks/65-revoke-vault-root-token.yaml`, the initial root token is intentionally invalidated and any future break-glass root access must be generated through Vault's recovery workflow.