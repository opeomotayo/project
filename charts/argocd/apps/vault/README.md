helm repo add hashicorp https://helm.releases.hashicorp.com
helm upgrade --install vault hashicorp/vault -n vault --create-namespace=true -f vault-values.yaml

kubectl -n vault exec -it vault-0 -- vault operator init

Unseal Key 1: kEmlt0Dx1R0BHHqkKatr8gFk4cxNGih2u1PGc1Gf4LZD
Unseal Key 2: KWRvO6MuLxKn5EzRvT8Py/eTMOBp9m8qY2dYMJqWsmfC
Unseal Key 3: 9i4EWgWYvrS8mppaPTSFk5XOzu64FCc4S2HoDJY07pe/
Unseal Key 4: wKAc5kEZZYnQMyrfuPORzqRjI2gKNm4VguQrvjeqSgHt
Unseal Key 5: zHua21VVE9xVCxrsBNO7czW4cYMz7/cDyicLgWjnUHl7

Initial Root Token: s.0lgn3p03x6XJPcVdjKG6v7bg

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 keys to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.

kubectl -n vault exec -it vault-0 -- sh
VAULT_ADDR=http://192.168.56.50:30001 vault status
VAULT_ADDR=http://192.168.56.50:30001 vault operator unseal
VAULT_ADDR=http://192.168.56.50:30001 vault login

https://marcofranssen.nl/install-hashicorp-vault-on-kubernetes-using-helm-part-1


https://www.vaultproject.io/docs/auth/approle
vault auth enable approle
vault write auth/approle/role/argocd-role \
    secret_id_ttl=10m \
    token_num_uses=10 \
    token_ttl=20m \
    token_max_ttl=30m \
    secret_id_num_uses=40

vault read auth/approle/role/argocd-role/role-id
vault write -f auth/approle/role/argocd-role/secret-id