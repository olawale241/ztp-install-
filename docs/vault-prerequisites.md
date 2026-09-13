# Vault prerequisites for `ove604a`

The repository creates **four `VaultStaticSecret` resources** in the complete lifecycle:

1. Hub: `pull-secret` -> Kubernetes `Secret/pull-secret`
2. Hub: `bmc-credentials` -> Kubernetes `Secret/bmc-credentials`
3. Hub: `vault-bootstrap-credentials` -> Kubernetes `Secret/vault-bootstrap-credentials`
4. Spoke: `cluster-config-repository` -> Kubernetes `Secret/cluster-config-repository`

## Vault KV-v2 data

The `acm` KV-v2 mount must contain:

```text
acm/ove604a/pull-secret
acm/ove604a/bmc
acm/ove604a/vault-bootstrap
acm/ove604a/day2/git
```

Expected keys:

```text
acm/ove604a/pull-secret
  .dockerconfigjson=<OpenShift pull secret JSON>

acm/ove604a/bmc
  username=<BMC username>
  password=<BMC password>

acm/ove604a/vault-bootstrap
  VAULT_TOKEN=<scoped bootstrap token>

acm/ove604a/day2/git
  url=https://github.com/<org>/<day2-config-repo>.git
  username=<optional for HTTPS auth>
  password=<optional token/password for HTTPS auth>
  # or sshPrivateKey=<key> when using SSH repository authentication
```

## Vault endpoint configuration

`VAULT_ADDR` is configuration, not secret material. For `ove604a` it is set to `http://192.168.1.123:8200` in the cluster overlay:

```text
overlays/cluster/acm/acm704a/ove604a/provisioning-vault-connection-patch.yaml
overlays/cluster/acm/acm704a/ove604a/day2-bootstrap-configmap-patch.yaml
```

The Hub bootstrap Job reads `.spec.address` from `VaultConnection/provisioning-vault`. The spoke bootstrap Job reads `VAULT_ADDR` from `ConfigMap/day2-site-config`, which is injected as an extra manifest by the overlay. Therefore the `acm/ove604a/vault-bootstrap` KV entry only needs the scoped `VAULT_TOKEN`.

## Hub Vault Kubernetes-auth prerequisite

The Hub-side `VaultAuth/provisioning-vault-auth` uses Vault's `kubernetes` auth mount. That mount must already be configured to trust the ACM Hub before Argo syncs this overlay.

Vault must also already have the role:

```text
ove604a-provisioning
```

Bind it to:

```text
service account: provisioning-vault
namespace:       ove604a
audience:        vault
```

Its policy must allow `read` on the three Hub bootstrap values:

```hcl
path "acm/data/ove604a/pull-secret" {
  capabilities = ["read"]
}

path "acm/data/ove604a/bmc" {
  capabilities = ["read"]
}

path "acm/data/ove604a/vault-bootstrap" {
  capabilities = ["read"]
}
```

The `/data/` segment is required in the ACL because `acm` is KV-v2. The `VaultStaticSecret.spec.path` values intentionally omit `/data/`.

## Bootstrap token policy

The `VAULT_TOKEN` stored at `acm/ove604a/vault-bootstrap` must **not** be a root token. Give it only the permissions required to create/update the dedicated spoke JWT auth mount, policy, and role:

```hcl
path "sys/auth/jwt-ove604a" {
  capabilities = ["create", "read", "update", "sudo"]
}

path "auth/jwt-ove604a/config" {
  capabilities = ["create", "read", "update"]
}

path "auth/jwt-ove604a/role/ove604a-day2" {
  capabilities = ["create", "read", "update"]
}

path "sys/policies/acl/ove604a-day2" {
  capabilities = ["create", "read", "update"]
}
```

The token TTL must cover the **entire provisioning window**, because the Hub Job does not use it until `AgentClusterInstall` reports `Completed=True`. Revoke or rotate that token after `Job/vault-bootstrap` succeeds.

The Hub bootstrap Job creates the resulting Day-2 policy as:

```hcl
path "acm/data/ove604a/day2/*" {
  capabilities = ["read"]
}
```

and creates:

```text
JWT auth mount: jwt-ove604a
Vault role:     ove604a-day2
Vault policy:   ove604a-day2
Bound subject:  system:serviceaccount:openshift-gitops:day2-vault
Audience:       vault
```

## Vault-to-spoke connectivity

The Vault server must be able to resolve and reach the new spoke API endpoint on port 6443. The Hub bootstrap Job configures Vault to read the spoke's service-account JWKS from:

```text
https://api.ove604a.walelab.smoad.net:6443/openid/v1/jwks
```

The extra-manifest RBAC grants unauthenticated read access only to the Kubernetes service-account issuer discovery endpoint, whose content is public signing-key material.

## TLS

The lab manifests currently use `skipTLSVerify: true` for the VSO `VaultConnection`, and the Hub bootstrap script uses `curl -k` when calling Vault. For production, provide the Vault CA and remove TLS verification bypasses.

## Vault Enterprise namespace note

This design creates a **per-cluster JWT auth mount, policy, role, and KV path**. It does not create a HashiCorp Vault Enterprise Namespace. If your Vault deployment uses Enterprise Namespaces, the VSO resources and bootstrap API calls must also be configured with the appropriate Vault namespace/header.
