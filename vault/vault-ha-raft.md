
- [Configure Helm repository](#configure-helm-repository)
- [Install Vault HA-mode (Integrated Storage (Raft) Backend)](#install-vault-ha-mode-integrated-storage-raft-backend)
- [Initialize Vault](#initialize-vault)
- [Unseal servers](#unseal-servers)
- [Joining Vault servers to the Raft cluster](#joining-vault-servers-to-the-raft-cluster)
- [Check Raft cluster](#check-raft-cluster)

Hashicorp Vault High Availability Installation

## Configure Helm repository

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

## Install Vault HA-mode (Integrated Storage (Raft) Backend)

[Recommended storage backends](https://www.vaultproject.io/docs/concepts/ha#storage-support)

This guide uses the Integrated Storage (Raft) Backend.

```bash
helm -n vault install vault hashicorp/vault \
  --set="global.openshift=true" \
  --set="server.ha.enabled=true" \
  --set="server.ha.raft.enabled=true" \
  --create-namespace
```

## Initialize Vault

Run the following commands, save unseal keys and root token in a secure place.

```bash
oc project vault
oc exec -ti vault-0 -- vault operator init
```

## Unseal servers

Unseal the Vault server with the key shares until the key threshold is met (3 keys by default)

```bash
oc exec -ti vault-0 -- vault operator unseal # ... Unseal Key 1
oc exec -ti vault-0 -- vault operator unseal # ... Unseal Key 2
oc exec -ti vault-0 -- vault operator unseal # ... Unseal Key 3
```

Output:

```text
oc exec -ti vault-0 -- vault operator unseal
Unseal Key (will be hidden): 
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            5
Threshold               3
Version                 1.10.3
Storage Type            raft
HA Enabled              true
```

## Joining Vault servers to the Raft cluster

Vault-1

Unseal the Vault server with the key shares until the key threshold is met.

```bash
oc exec -ti vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
oc exec -ti vault-1 -- vault operator unseal
```

Vault-2
Unseal the Vault server with the key shares until the key threshold is met.

```bash
oc exec -ti vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
oc exec -ti vault-2 -- vault operator unseal
```

## Check Raft cluster

Login using the `root` token on the `vault-0` pod

```bash
oc exec -ti vault-0 -- vault login
```

List all the raft peers

```bash
oc exec -ti vault-0 -- vault operator raft list-peers
```

Output:

```text
Node                                    Address                        State       Voter
----                                    -------                        -----       -----
aefc4c40-29cc-899b-fdc5-b102de0ccdee    vault-0.vault-internal:8201    leader      true
b84437af-2036-f0bd-0901-1d6896916fd6    vault-1.vault-internal:8201    follower    true
47ac49c1-00e0-8f21-50af-f77509e02036    vault-2.vault-internal:8201    follower    true
```
