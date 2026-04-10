# Ed25519 To ECDSA P-256 Transition

This document explains why the bootstrap internal TLS PKI is being changed from Ed25519-based X.509 keys to ECDSA P-256 (`secp256r1`), what was changed in the repo, and what operators must expect during the transition.

## Why This Transition Is Needed

The original bootstrap TLS CA and several issued service/client certificates were generated with Ed25519 keys.

That worked for some CLI tooling such as OpenSSL and curl, but it caused compatibility problems in browser-based workflows. In particular, Firefox rejected the Keycloak certificate chain with:

- `SEC_ERROR_UNSUPPORTED_KEYALG`

In practice, this means:

- the TLS chain could be mathematically valid and trusted by OpenSSL
- but the browser could still reject it because of unsupported or incompletely supported key/signature algorithms in the X.509 chain

## Technical Background

Ed25519 is widely used and well supported for SSH and many modern cryptographic use cases.

However, browser support for Ed25519 in X.509 certificate chains has historically lagged behind support for more conventional Web PKI algorithms.

The important distinction is:

- Ed25519 support for SSH is not the same as Ed25519 support for X.509 TLS certificates
- support in OpenSSL-based tooling is not identical to support in browser TLS stacks such as Firefox/NSS

So a certificate chain may work in:

- `openssl s_client`
- `curl`

while still failing in:

- Firefox
- other browser or NSS-backed consumers

For browser-facing OIDC login flows, that incompatibility is operationally unacceptable because the administrator must be able to open Keycloak in a browser without certificate algorithm errors.

## Why ECDSA P-256

The replacement algorithm is ECDSA P-256, implemented here as:

- `type: ECC`
- `curve: secp256r1`

This choice is practical because:

- it is broadly supported in browsers
- it is widely supported across common TLS libraries
- it remains modern and efficient
- Keycloak TLS generation in this repo was already using `secp256r1`

RSA would also have been a viable compatibility choice, but ECDSA P-256 is a better fit for the repo's existing direction.

## What Changed In The Repo

The following X.509 key generation tasks were changed from Ed25519 to ECDSA P-256:

- `tasks/bootstrap-tls-vault-consul/10_generate_ca.yaml`
- `tasks/bootstrap-tls-vault-consul/20_issue_vault_cert.yaml`
- `tasks/bootstrap-tls-vault-consul/21_issue_consul_cert.yaml`
- `tasks/bootstrap-tls-vault-consul/22_issue_certs_for_consul_clients.yaml`
- `tasks/bootstrap-tls-vault-consul/23_issue_bastion_vault_client_cert.yaml`

Each now uses:

```yaml
community.crypto.openssl_privatekey:
  type: ECC
  curve: secp256r1
```

## What Did Not Change

WireGuard was not changed.

That is intentional.

WireGuard does not use X.509 certificates for peer identity. It uses its own Curve25519-based key exchange and peer key model internally, so the browser/TLS/X.509 compatibility problem does not apply there.

In other words:

- X.509 TLS certificate algorithm compatibility is the problem being fixed here
- WireGuard key generation and authentication are unrelated to that problem

## Operational Impact

Changing the bootstrap CA key algorithm means the old CA and all certificates issued by it must be treated as replaced.

This is effectively a PKI rotation event.

Operators should expect to:

1. remove the old bootstrap TLS material
2. regenerate the bootstrap CA
3. reissue Vault, Consul, and client certificates
4. redistribute the new certificates
5. re-import the new CA on administrator workstations and browsers

Because the CA changes, simply replacing a leaf certificate is not enough.

Any trust store that previously contained the old CA must be updated with the new CA certificate.

## Why This Is Better

After this transition, the bootstrap TLS chain should be acceptable to:

- browser-based OIDC login flows
- OpenSSL-based CLI tooling
- typical Web PKI consumers in the local administration workflow

That makes the internal Keycloak and Vault HTTPS endpoints usable both from automation and from administrator browsers.

## Summary

The old design used Ed25519 for internal X.509 bootstrap TLS certificates.

That was modern, but not sufficiently compatible for browser-facing OIDC workflows.

The new design uses ECDSA P-256 for the bootstrap CA and issued internal TLS certificates so that:

- browser support is reliable
- TLS behavior is more uniform across tools
- WireGuard remains unchanged because it does not depend on X.509 certificate algorithms