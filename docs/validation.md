# Validation checklist

The overlay intentionally stays compatible with the Kustomize 5.6 generation embedded in OpenShift 4.20 `oc`. It uses normal manifest patches plus `replacements` for resource names. It does not use a global namespace transformer; namespace changes are explicit so the cluster-scoped `ManagedCluster` remains namespace-free and RoleBinding subjects remain correct.

This repository has been statically validated for YAML syntax, local file references, Kustomize patch targets, embedded shell syntax, and cross-resource names.

## Required values before Argo sync

- Vault endpoint in:
  - `base/secrets/provisioning-vault-connection.yaml`
  - `base/extra-manifests/13-day2-bootstrap-job.yaml`
- SSH public key in the `ove604a` overlay/base.
- Six `ens224` MAC addresses in `overlays/cluster/acm/acm704a/ove604a/nmstate/`.
- HA API/Ingress VIP or external load-balancer/user-managed-networking design.
- Vault data, Hub Kubernetes-auth configuration, policies, and Vault-to-spoke API reachability described in `docs/vault-prerequisites.md`.
- `ClusterImageSet/openshift-4.21` present on the Hub.

## Hub preflight

```bash
oc get crd agentclusterinstalls.extensions.hive.openshift.io
oc get crd infraenvs.agent-install.openshift.io
oc get crd nmstateconfigs.agent-install.openshift.io
oc get crd baremetalhosts.metal3.io
oc get crd vaultconnections.secrets.hashicorp.com
oc get crd vaultauths.secrets.hashicorp.com
oc get crd vaultstaticsecrets.secrets.hashicorp.com
oc get clusterimageset openshift-4.21
oc get catalogsource -n openshift-marketplace redhat-operators certified-operators
```

## Render before apply

Run this on the ACM Hub or any workstation with `oc`/Kustomize available:

```bash
oc kustomize ztp-install/overlays/cluster/acm/acm704a/ove604a > /tmp/ove604a-rendered.yaml

grep -nE 'CHANGE_ME|YOUR_SSH_PUBLIC_KEY' /tmp/ove604a-rendered.yaml
```

The grep should return nothing before production deployment.

Then server-side validate where supported:

```bash
oc apply --dry-run=server -f /tmp/ove604a-rendered.yaml
```

## Hub runtime checks

```bash
oc -n ove604a get vaultconnection,vaultauth,vaultstaticsecret
oc -n ove604a get secret pull-secret bmc-credentials vault-bootstrap-credentials
oc -n ove604a get infraenv,clusterdeployment,agentclusterinstall
oc -n ove604a get bmh,nmstateconfig
oc -n ove604a get job vault-bootstrap
oc -n ove604a logs job/vault-bootstrap
```

Successful install gate:

```bash
oc -n ove604a get agentclusterinstall ove604a \
  -o jsonpath='{.status.conditions[?(@.type=="Completed")].status}{"\n"}'
```

Expected: `True`.

## Spoke runtime checks

```bash
oc get subscription -n openshift-operators openshift-gitops-operator vault-secrets-operator
oc get csv -n openshift-operators
oc -n openshift-gitops get job day2-bootstrap
oc -n openshift-gitops logs job/day2-bootstrap
oc -n openshift-gitops get vaultconnection day2-vault
oc -n openshift-gitops get vaultauth day2-vault-auth
oc -n openshift-gitops get vaultstaticsecret cluster-config-repository
oc -n openshift-gitops get secret cluster-config-repository
oc -n openshift-gitops get application ove604a-cluster-config
```
