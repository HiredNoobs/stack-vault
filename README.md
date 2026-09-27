# stack-vault

Vault deployment for k8s, with raft storage.

One Vault node runs on each node labelled `vault_core_vault` (set in iac-homelab), the count decides the mode when deploying:

- 1 node - a single Vault node. Its data is only replicated by Longhorn.
- 3+ nodes - a raft cluster. `vault.json` lists every node in `retry_join`, so they join each other, and a PodDisruptionBudget keeps drains (e.g. Talos upgrades) to one node at a time.
- 2 nodes are refused, raft needs both for a quorum so it's less available than one.

Going from 1 to 3 nodes is labelling two more nodes and deploying again, the new nodes join and are unsealed. Going down isn't automatic, `deploy` refuses until the extra nodes are removed from raft (`vault operator raft remove-peer vault-<n>`) and the StatefulSet is scaled down (`kubectl scale statefulset vault --replicas=<n>`).

The data is on a `$VAULT_STORAGE_CLASS` volume per node, Longhorn (`stack-longhorn`) has to be deployed first. `clean` keeps the volumes, `purge` removes them and everything in Vault with them.

| Service | Type | Used for |
| --- | --- | --- |
| `vault-active` | ClusterIP | The active node (labelled by Vault's kubernetes service registration), nginx proxies `vault.$DOMAIN` here |
| `vault-internal` | Headless | Each node, `vault-<n>.vault-internal.vault-core.svc`, for raft and nginx's `vault1`, `vault2`, ... |

## Sealing

Pods start sealed and aren't ready (so get no traffic) until they're unsealed. The keys are only in `$SECRETS/keys.json` on the management host.

- `deployment deploy` initialises Vault on the first deploy (writing `keys.json`), then unseals every pod, including pods replaced one at a time when the config or certs change. It won't overwrite an existing `keys.json` or carry on without one.
- `deployment vault_unseal` unseals the pods after a restart, e.g. a node reboot or Talos upgrade.
- `deployment vault_status` shows each pod with its active, sealed and initialised labels.

Back up `keys.json` somewhere other than Vault itself (a sealed Vault can't return its own keys), without it Vault's data can't be unsealed.

## Initial setup

After initialising, the only valid authentication method is the ``root_token`` in ``$SECRETS/keys.json``.

`deploy` then runs ``tools/initial-setup`` if ``$VAULT_INIT_USERNAME`` and ``$VAULT_INIT_PASSWORD`` are set (in the deployment's ``secrets.env``). It sets up the ``lab`` (kv) and ``labv2`` (kv v2) secret engines, their policies (``configs/*.hcl``, mounted at ``/vault/policies``) and a userpass user with both. It's safe to re-run: ``deployment tools initial-setup``.

To do it manually:

```bash
kubectl exec -it vault-0 -n vault-core -- sh

# VAULT_ADDR and VAULT_CACERT are already set for the local node
vault login    # the root token
vault secrets enable -path=lab kv
vault policy write lab-policy /vault/policies/lab-policy.hcl
vault auth enable userpass
vault write auth/userpass/users/USERNAME password=PASSWORD policies=lab-policy
```
