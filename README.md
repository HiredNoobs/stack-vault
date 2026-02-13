# stack-vault

Vault deployment.

## Initial setup

The deployment will initialise and unseal Vault automatically but the only valid authentication method will the ``root_token`` that can be found in ``$SECRETS/keys.json``.

To create a user:

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
