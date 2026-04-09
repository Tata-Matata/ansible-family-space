# vault-human-auth

This note explains the end-to-end human administrator access flow for Vault-backed SSH certificates and how that flow maps to the `92-vault-human-auth.yaml` and `93-vault-human-ssh-role.yaml` playbooks.

## The Normal Admin Flow

The intended day-to-day human administrator flow is:

1. The administrator connects through WireGuard so private services like Vault and Keycloak are reachable.
2. The administrator runs `vault login -method=oidc`.
3. Vault receives the login request on its OIDC auth mount.
4. Vault uses the configured OIDC backend settings to redirect the administrator to Keycloak.
5. Keycloak authenticates the human user and returns identity claims to Vault.
6. Vault checks the matching Vault OIDC role to decide whether this identity is allowed in and which policies the resulting Vault token should receive.
7. If the role checks pass, Vault issues a limited Vault token.
8. The administrator uses that Vault token to request SSH signing for a local SSH public key.
9. Vault checks whether that Vault token is allowed to call the SSH signing endpoint for the configured SSH role.
10. If allowed, Vault signs the SSH public key and returns a short-lived SSH certificate.
11. The administrator uses the SSH private key plus the signed SSH certificate to connect to infrastructure hosts.
12. The hosts accept that login because they already trust the Vault SSH CA public key distributed earlier.

The important distinction is that the human does not SSH with the Vault token directly.

The Vault token is an intermediate authorization credential that allows the human to ask Vault for the final SSH certificate.

## What `92-vault-human-auth.yaml` Configures

`92-vault-human-auth.yaml` configures Vault's OIDC backend.

It does two main things:

1. Enables the OIDC auth method at `auth/{{ vault_human_auth_mount_path }}` if it is not already enabled.
2. Writes the backend configuration to `auth/{{ vault_human_auth_mount_path }}/config`.

That backend config tells Vault how to talk to Keycloak. It includes:

- `oidc_discovery_url`
- `oidc_client_id`
- `oidc_client_secret`
- `default_role`

This is protocol and connectivity setup.

It does not yet decide which users are allowed in or what those users may do after login.

## What `93-vault-human-ssh-role.yaml` Configures

`93-vault-human-ssh-role.yaml` is the authorization step.

It creates three Vault objects that connect Keycloak-authenticated identity to SSH certificate signing:

1. A Vault policy for human SSH signing.
2. A Vault SSH signing role.
3. A Vault OIDC role.

### 1. Vault Human SSH Policy

The playbook writes the policy content from `vault_human_policy_hcl` into Vault.

That policy allows a Vault token to call the SSH sign endpoint for the human SSH role.

In practical terms, this is what allows the user, after OIDC login, to ask Vault to sign an SSH public key.

Without this policy, the human could authenticate to Vault but still would not be allowed to request an SSH certificate.

### 2. Vault SSH Signing Role

The playbook creates the SSH role under the SSH secrets engine, for example under:

- `ssh/roles/{{ vault_human_ssh_role_name }}`

This role defines what kind of SSH certificate Vault is allowed to issue, including:

- allowed Linux usernames
- default Linux username
- TTL
- max TTL
- that the role uses the SSH CA signing mode

This role controls the SSH certificate that comes out at the end of the flow.

### 3. Vault OIDC Role

The playbook also creates the OIDC role under the auth method, for example under:

- `auth/{{ vault_human_auth_mount_path }}/role/{{ vault_human_oidc_role_name }}`

This role defines:

- which identity claims Vault reads from the OIDC token
- which groups claim Vault uses
- which groups are required
- which Vault policies get attached to the resulting Vault token
- how long the Vault token lives

In this repo, the important effect is:

- only Keycloak users in `infra-admins` are accepted
- successful login receives the `human-ssh` Vault policy

That is the step that converts Keycloak identity into Vault authorization.

## Mapping The Real User Flow To `93`

The normal user flow maps to the playbook objects like this:

1. User logs in through Keycloak.
2. Vault receives the identity claims.
3. Vault checks the Vault OIDC role created by `93`.
4. If the claims match, Vault issues a token with the policy created by `93`.
5. User calls the SSH signing endpoint.
6. Vault checks whether the token's attached policy allows access to that endpoint.
7. Vault uses the SSH signing role created by `93` to decide what certificate may be issued.
8. Vault signs the key and returns the SSH certificate.

So `93` is the playbook that makes this sentence true:

If a Keycloak-authenticated human belongs to the required group, Vault should issue a token that may request a short-lived SSH certificate for the approved SSH role.

## Are The OIDC Role And SSH Role The Same Kind Of Thing?

They are both called `role` in Vault, but conceptually they are different role objects in different subsystems.

### OIDC Role

This lives under the auth method.

Its job is:

- evaluate identity claims after OIDC login
- decide whether login is allowed
- attach policies to the resulting Vault token

This role is about authentication-to-authorization mapping.

### SSH Signing Role

This lives under the SSH secrets engine.

Its job is:

- define what kind of SSH certificate may be issued
- define allowed usernames and TTL limits

This role is about secret issuance behavior.

So they are both Vault roles, but not the same class of role conceptually.

One role decides what Vault token a human gets after login.

The other role decides what SSH certificate Vault may issue when a token calls the SSH sign endpoint.

## Why It Is Not "A Role Using Another Role"

When the flow says Vault checks whether the Vault token may use the SSH signing role, what is really happening is:

1. the OIDC role attached the `human-ssh` policy to the Vault token
2. that policy grants permission on the SSH sign endpoint for the named SSH role
3. Vault then uses the SSH role definition to decide how to issue the certificate

So the relationship is:

- OIDC role -> attaches policy to token
- policy -> grants access to a specific SSH signing endpoint
- SSH role -> defines the certificate issuance rules at that endpoint

It is not that one Vault role directly invokes another Vault role.

The connecting object is the Vault policy attached to the token.

## Summary

The human access design has three separate concepts:

1. OIDC backend config in `92`: how Vault talks to Keycloak.
2. OIDC role in `93`: which Keycloak-authenticated humans get which Vault token policies.
3. SSH role in `93`: what kind of SSH certificate Vault may issue when a permitted token asks for signing.

That separation is why the flow feels more complex than plain tokens and policies at first, but it is still the same core idea:

- authenticate identity
- issue a limited Vault token
- use that token to authorize a tightly scoped action
- return a short-lived access artifact