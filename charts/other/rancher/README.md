helm upgrade --install rancher rancher-latest/rancher --namespace rancher --set hostname=rancher.demo.com --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=opeomotayo@gmail.com
kubectl -n rancher apply -f rancher-svc.yaml 
Ayodeji1979!
