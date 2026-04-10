# Certificate Rotation Runbook

This document describes how to rotate the temporary bootstrap TLS certificate chain used by Vault, Consul, Bastion client mTLS, and Keycloak in this repository.

It is intended for cases such as:

- the bootstrap TLS CA is about to expire
- the bootstrap TLS CA algorithm changes
- a browser or client compatibility problem requires a new bootstrap CA and reissued leaf certificates

This runbook is about the temporary bootstrap X.509 TLS material.

It does **not** apply to the Vault SSH CA used for SSH user certificates.

## What This Rotation Affects

The bootstrap TLS CA signs temporary X.509 certificates for:

- Vault HTTPS
- Consul HTTPS / mTLS
- Bastion client certificate for Vault
- Bastion client certificate for Consul
- Vault client certificate for Consul
- Keycloak HTTPS

Because all of those depend on the same bootstrap CA, changing the CA means a full reissue and redistribution cycle.

## When Cleanup Is Required

The CA generation task refuses to reuse an existing bootstrap CA key.

That means a full CA rotation requires cleanup first.

The cleanup playbook is:

- `playbooks/00-optional-remove-tls-material.yaml`

It uses two separate flags:

- `cleanup_bootstrap_tls=yes`
  - removes CA and generated TLS material on the primary Bastion
- `cleanup_nodes=yes`
  - removes distributed TLS material on Vault, Consul, Bastion, and OIDC hosts

For a full bootstrap CA rotation, use both.

## Full Rotation Sequence

The intended rotation sequence is:

1. Remove old bootstrap TLS material.
2. Regenerate the bootstrap CA and issued certificates on Bastion.
3. Re-distribute TLS material to Bastion, Vault, Consul, and Keycloak.
4. Reconfigure and restart services so they load the new files.
5. Re-trust the new CA on administrator workstations and browsers.
6. Verify Vault, Consul, and Keycloak over the new TLS chain.

## Step 1: Remove Old Bootstrap TLS Material

Run the cleanup playbook with both cleanup flags enabled:

```bash
ansible-playbook playbooks/00-optional-remove-tls-material.yaml -e cleanup_bootstrap_tls=yes -e cleanup_nodes=yes
```

What this removes:

- Bastion bootstrap CA and generated temporary TLS material
- Vault TLS directory contents
- Consul TLS directory contents
- Bastion distributed client TLS contents
- Keycloak TLS directory contents

This step is necessary for a full CA rotation because `10_generate_ca.yaml` refuses to reuse an existing CA private key.

## Step 2: Regenerate Bootstrap TLS Material On Bastion

Run:

```bash
ansible-playbook playbooks/40-generate-tmp-tls-vault-consul-on-bastion.yaml
```

This regenerates on the primary Bastion:

- bootstrap TLS CA
- Vault server TLS cert/key
- Consul server TLS cert/key
- Vault client cert/key for Consul
- Bastion client cert/key for Consul
- Bastion client cert/key for Vault

## Step 3: Reinstall Bastion Client TLS Material

Run:

```bash
ansible-playbook playbooks/41-distribute-tls-to-bastions.yaml
```

This copies the new CA and client TLS material to Bastion hosts and updates the Bastion system trust store.

## Step 4: Reinstall And Restart Consul TLS

Run:

```bash
ansible-playbook playbooks/42-reconfigure-consul-for-tls.yaml
```

This does two things:

1. copies the new Consul TLS material from Bastion
2. re-renders configuration and restarts Consul

## Step 5: Reinstall And Restart Vault TLS

Run:

```bash
ansible-playbook playbooks/43-reconfigure-vault-for-tls.yaml
```

This does two things:

1. copies the new Vault TLS material from Bastion
2. re-renders configuration and restarts Vault

### Important Note About Vault State

Vault restart is enough for Vault to load the new certificate files.

However, depending on how Vault is currently operating, restart may leave it sealed.

If Vault comes back sealed after the restart, run the normal unseal steps again using your existing unseal workflow.

## Step 6: Reissue And Restart Keycloak TLS

Because Keycloak TLS is also signed by the same bootstrap CA, it must be rotated too.

Run:

```bash
ansible-playbook playbooks/keycloak/60-configure-tls.yaml
ansible-playbook playbooks/keycloak/70-redeploy-stack-tls.yaml
ansible-playbook playbooks/keycloak/80-verify-tls.yaml
```

This will:

- generate a new Keycloak certificate signed by the new bootstrap CA
- install it on the OIDC host
- refresh trust
- restart Keycloak in TLS mode
- verify the resulting HTTPS endpoint

If you normally use the wrapper flow, remember that Keycloak TLS is the phase-two part of the `80-keycloak.yaml` workflow.

For pure TLS rotation after Keycloak is already deployed, rerunning `60`, `70`, and `80` is the targeted path.

## Step 7: Update Workstation And Browser Trust

Any workstation or browser that previously trusted the old bootstrap CA must be updated to trust the new one.

That includes:

- the OS trust store used by CLI tools such as curl and Vault CLI
- browser trust stores, especially Firefox if it uses its own certificate store

If browser access to Keycloak is part of the workflow, this step is mandatory.

## Step 8: Post-Rotation Verification

After rotation, verify the important endpoints.

### Vault

- CLI can connect to `https://vault.service.internal:8200`
- service health endpoint responds over HTTPS

### Consul

- Bastion can reach Consul API over HTTPS
- Consul service is active on each node

### Keycloak

- `https://keycloak.service.internal:8443/` opens without certificate errors
- OIDC discovery works
- browser OIDC login works

### VPN Client Experience

If administrators access these services through WireGuard, also verify:

- internal DNS still resolves service names
- browsers trust the new CA
- `vault login -method=oidc` works again

## Commands Summary

Full cleanup:

```bash
ansible-playbook playbooks/00-optional-remove-tls-material.yaml -e cleanup_bootstrap_tls=yes -e cleanup_nodes=yes
```

Regenerate bootstrap material:

```bash
ansible-playbook playbooks/40-generate-tmp-tls-vault-consul-on-bastion.yaml
```

Reinstall Bastion client TLS:

```bash
ansible-playbook playbooks/41-distribute-tls-to-bastions.yaml
```

Reinstall Consul TLS:

```bash
ansible-playbook playbooks/42-reconfigure-consul-for-tls.yaml
```

Reinstall Vault TLS:

```bash
ansible-playbook playbooks/43-reconfigure-vault-for-tls.yaml
```

Reissue Keycloak TLS:

```bash
ansible-playbook playbooks/keycloak/60-configure-tls.yaml
ansible-playbook playbooks/keycloak/70-redeploy-stack-tls.yaml
ansible-playbook playbooks/keycloak/80-verify-tls.yaml
```

## Notes

- This is a bootstrap TLS rotation only.
- The Vault SSH CA does not need to be rotated as part of this process.
- WireGuard is unrelated to the X.509 bootstrap TLS chain and does not need certificate changes.
- Once Vault-managed PKI replaces this bootstrap material, this runbook should become unnecessary for normal operations.