# stack-vault

Vault deployment.

## Initial setup

The deployment will initialise and unseal Vault automatically but the only valid authentication method will the ``root_token`` that can be found in ``$SECRETS/keys.json``.

At this point Vault can be manaully configured in the UI/CLI or the ``tools/initial-setup`` script can be used to setup a simple kv secret engine with policies applied to a new user ``$VAULT_INIT_USERNAME`` and ``$VAULT_INIT_PASSWORD`` must be defined in your environment or in the deployments ``secret.env`` for this setup.

To manually setup a kv secret engine and user run the following:

```bash
# Enter the CLI of the container
$ docker exec -it VAULT sh

# If not already set, or it needs to be changed
$ export VAULT_ADDR="https://vault.xyz"

# Login using the root token
$ vault login
$ vault secrets enable -path=lab kv

$ vault policy write lab-policy /vault/config/lab-policy.hcl

$ vault auth enable userpass
$ vault write auth/userpass/users/USERNAME password=PASSWORD policies=lab-policy
```
