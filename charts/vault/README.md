helm upgrade --install vault hashicorp/vault -n vault -f vault-values.yaml

kubectl -n vault exec -it vault-0 -- vault operator init
Unseal Key 1: /6DWC1SuFhb8KBW0ZOgPUYT87E/IpmdrQVk9w+2isUBm
Unseal Key 2: 2xQ50/RgekLg9L+j3/2F/nayFUG+sEMr45qiibEZmSZA
Unseal Key 3: fQbJlSSILku8VhHvxuBz2u5qmOv8+DatQP11SJF6FR9g
Unseal Key 4: inicfwWsurQsK4gmXlvyNXRhORs1sfv36sLhZhE/wFjw
Unseal Key 5: op2ChRJLpT211SsE1ncQnUk6H/Frc/bv36Puq+JuaaVz

token: s.HJNzvS8csSYmvdS71CtEZlhW

kubectl -n vault exec -it vault-0 -- vault operator init
kubectl -n vault exec -it vault-0 -- sh
$ VAULT_ADDR=http://192.168.56.50:30004 vault status
VAULT_ADDR=http://192.168.56.50:30004 vault operator unseal
VAULT_ADDR=http://192.168.56.50:30004 vault login





https://marcofranssen.nl/install-hashicorp-vault-on-kubernetes-using-helm-part-1