
- [Configure Helm repository](#configure-helm-repository)
- [Install Consul](#install-consul)
- [Install Vault HA-mode (Consul Backend)](#install-vault-ha-mode-consul-backend)
- [Initialize Vault](#initialize-vault)
- [Unseal servers](#unseal-servers)
- [Check Vault status](#check-vault-status)
- [Test Secret engine](#test-secret-engine)

Hashicorp Vault High Availability Installation

## Configure Helm repository

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

## Install Consul

Because of the [issue](https://github.com/hashicorp/consul-k8s/issues/1306) we need to have customized version of Consul.

Сlone the repository and install Consul from the cloned repository folder.

```bash
helm upgrade --install consul vault/consul --values vault/consul/values-custom.yaml -n consul --create-namespace
```

Check Consul pods

```bash
oc project consul
oc wait --for=condition=ready pod -l app=consul -n consul --timeout=240s
oc get pods

```

Output

```text
NAME                         READY   STATUS    RESTARTS   AGE
consul-consul-client-6j7sh   1/1     Running   0          15m
consul-consul-client-msxxs   1/1     Running   0          14m
consul-consul-client-zlpch   1/1     Running   0          15m
consul-consul-server-0       1/1     Running   0          14m
consul-consul-server-1       1/1     Running   0          15m
consul-consul-server-2       1/1     Running   0          15m
```

Check Consult cluster members (should be 3 servers and 3 clients)

```bash
oc rsh consul-consul-server-0 consul members
```

Output

```text
Node                                           Address           Status  Type    Build   Protocol  DC            Partition  Segment
consul-consul-server-0                         10.129.2.42:8301  alive   server  1.12.2  2         vault-argocd  default    <all>
consul-consul-server-1                         10.128.2.40:8301  alive   server  1.12.2  2         vault-argocd  default    <all>
consul-consul-server-2                         10.131.0.78:8301  alive   server  1.12.2  2         vault-argocd  default    <all>
argo-n7t2q-worker-a-8fxlm.c.fsi-env2.internal  10.129.2.41:8301  alive   client  1.12.2  2         vault-argocd  default    <default>
argo-n7t2q-worker-b-wthzd.c.fsi-env2.internal  10.128.2.39:8301  alive   client  1.12.2  2         vault-argocd  default    <default>
argo-n7t2q-worker-c-kspz9.c.fsi-env2.internal  10.131.0.77:8301  alive   client  1.12.2  2         vault-argocd  default    <default>
```

## Install Vault HA-mode (Consul Backend)

[Recommended storage backends](https://www.vaultproject.io/docs/concepts/ha#storage-support)

[Comparison of storage backend](https://www.vaultproject.io/docs/configuration/storage#storage-stanza)

This guide uses the Consul Backend with 3 cluster servers.

```bash
helm -n vault install vault hashicorp/vault \
  --set="global.openshift=true" \
  --set="server.ha.enabled=true" \
  --create-namespace
```

## Initialize Vault

> Note: Vault pods will in `Ready:False` status until we initialize Vault

Run the following commands, save unseal keys and root token in a secure place.

```bash
oc project vault
export VAULT_KEYTHRESHOLDS=3
oc exec -ti vault-0 -- vault operator init --format json | jq '.' | cat > /tmp/vaultkeys
```

## Unseal servers

Unseal the Vault servers with the key shares until the key threshold is met (3 keys by default)

```bash
for pod in $(oc get pods --no-headers -n vault -l component=server -o jsonpath='{.items[*].metadata.name}');
do
  echo "Working with $pod"
  for i in $(seq 1 $VAULT_KEYTHRESHOLDS)
  do
    echo "Unseal Key #$i:"
    oc exec -ti $pod -- vault operator unseal $(cat /tmp/vaultkeys | jq -r '.unseal_keys_b64'[$i])
    sleep 1
  done
done
```

## Check Vault status

```bash
oc exec -ti vault-0 -- vault status
```

Output:

```text
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.10.3
Storage Type    consul
Cluster Name    vault-cluster-1995e450
Cluster ID      74a4f50c-b678-b4c2-5686-72a1609fcf52
HA Enabled      true
HA Cluster      https://vault-0.vault-internal:8201
HA Mode         active
```

## Test Secret engine

Export `root_token`

```bash
export VAULT_TOKEN=$(cat /tmp/vaultkeys | jq -r ".root_token")
```

Login with `root_token` and enable `kv-v2` secrets

```bash
oc exec -ti vault-0 -- vault login $VAULT_TOKEN
oc rsh -n vault vault-0
vault secrets enable kv-v2
```

Create a simple secret and policy

```bash
vault secrets enable -path=avp -version=2 kv

vault kv put avp/test sample=password
vault kv get avp/test 

cat << EOF > /tmp/policy.hcl 
path "avp/data/test" { capabilities = ["read"] } 
EOF

vault policy write argocd-repo-server /tmp/policy.hcl
vault policy read argocd-repo-server

vault write auth/kubernetes/role/argocd-repo-server \
  bound_service_account_names=argocd-repo-server \
  bound_service_account_namespaces=openshift-gitops policies=argocd-repo-server
```
