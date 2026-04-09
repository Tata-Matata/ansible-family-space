# 92-vault-human-auth

This note explains the OIDC backend configuration written by `92-vault-human-auth.yaml`, what each field means, and how that configuration fits into the human administrator login flow.

## What This Playbook Configures

The playbook enables Vault's OIDC auth method at the configured mount path and writes the backend configuration to:

- `auth/{{ vault_human_auth_mount_path }}/config`

That backend config tells Vault how to talk to the OIDC provider.

It does not yet define which users are allowed in or which policies they get. That is handled later by the OIDC role created in `93-vault-human-ssh-role.yaml`.

## Fields In The OIDC Config Payload

The template `templates/vault-human-oidc-config.json.j2` renders the following JSON payload:

```json
{
  "oidc_discovery_url": "...",
  "oidc_client_id": "...",
  "oidc_client_secret": "...",
  "default_role": "..."
}
```

### `oidc_discovery_url`

This is the OIDC issuer or discovery base URL for the identity provider.

In this repo, that provider is Keycloak.

Vault uses this URL to discover the provider's standard OIDC endpoints, such as:

- authorization endpoint
- token endpoint
- userinfo endpoint
- JWKS signing keys

Without this field, Vault would not know where to send the user for login or how to validate the returned tokens.

### `oidc_client_id`

This is the client ID that Vault uses when acting as an OIDC client against Keycloak.

It must match the Keycloak client that was created for Vault.

When a user starts `vault login -method=oidc`, Vault identifies itself to Keycloak with this client ID.

### `oidc_client_secret`

This is the client secret paired with `oidc_client_id`.

Vault uses it when completing the OIDC authorization-code flow, especially when exchanging the returned authorization code for tokens.

Operationally, this is a provider-side integration secret between Vault and Keycloak.

### `default_role`

This is the default Vault OIDC role name to use when a user logs in without specifying a Vault role explicitly.

That role is created later by `93-vault-human-ssh-role.yaml`.

The role is where Vault decides things like:

- which claims to read from the OIDC token
- which Keycloak groups are required
- which Vault policies should be attached to the resulting Vault token
- how long that Vault token should live

So `default_role` is the bridge between:

- backend connectivity to Keycloak
- authorization decisions inside Vault

## How These Fields Are Used In The Login Flow

The intended human administrator flow is:

1. The operator connects through WireGuard so private services like Vault and Keycloak are reachable.
2. The operator runs `vault login -method=oidc`.
3. Vault receives the login request on the configured OIDC auth mount.
4. Vault uses `oidc_discovery_url` to locate the Keycloak OIDC endpoints.
5. Vault starts the OIDC flow using `oidc_client_id`.
6. Vault authenticates to Keycloak as that client using `oidc_client_secret` during the code exchange.
7. Vault applies `default_role` if the login did not specify another Vault OIDC role.
8. That Vault role checks claims from the Keycloak-issued identity token, including the configured groups claim and required group membership.
9. If the role checks pass, Vault issues a Vault token with the policies attached to that role.
10. The human then uses that Vault token to request a signed SSH certificate from Vault's SSH secrets engine.

## How This Fits The Repo's Use Case

In this repo, the goal is not just "let a human log into Vault".

The goal is:

1. authenticate the human against Keycloak
2. translate that identity into a limited Vault token
3. let that Vault token request a short-lived SSH certificate
4. use that certificate to access infrastructure hosts that trust the Vault SSH CA

In that larger flow, the OIDC backend config in `92-vault-human-auth.yaml` is the piece that makes Vault trust and communicate with Keycloak.

The later OIDC role and SSH role configuration then turns that authenticated identity into actual permissions.

## Important Boundary

This backend config is about connectivity and protocol setup.

It is not the place where access restrictions are enforced.

Restrictions such as required groups like `infra-admins`, token policies, TTLs, and the mapping to SSH-signing permissions live in the Vault OIDC role and Vault policy definitions configured later in the flow.