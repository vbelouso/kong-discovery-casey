# ArgoCD Vault and Consult bootstrapping

## Repo structure

`argocd` folder contains applications for `Consul`, `Vault`, `Prepers`, `PostInstall` steps and `Kong Secret`.

`vault/charts` contains customized version of Consul chart because of the [issue](https://github.com/hashicorp/consul-k8s/issues/1306).

## Usesul guides

[Installation process for Vault and Consul](./vault-ha-consul.md)

[Installation process for Vault and Raft](./vault-ha-raft.md)

## Installation process

- Install `prepers`

```bash
oc apply -f vault/application-prereqs.yaml
```

- Install `Consul`

```bash
oc apply -f vault/argocd/consul/application.yaml
```

- Install `Vault`

```bash
oc apply -f vault/argocd/vault/application.yaml
```

- Init and unseal Vault manually of via Job `job-vault-init`

> NOTE: To get `root_token` or `unseal keys` you need to login to Job pod and check tmp file in `/tmp` directory.

```bash
oc apply -f vault/argocd/postinstall/rbac.yaml
oc apply -f vault/argocd/postinstall/sa.yaml
oc apply -f vault/argocd/postinstall/job-vault-init.yaml
```

- Configure Vault auth and policy

> NOTE: Not ready yet, need some cosmetic changes.

```bash
oc apply -f vault/argocd/postinstall/job-vault-secret.yaml
```