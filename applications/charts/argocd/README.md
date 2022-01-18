https://bitnami.com/stack/argo-cd/helm

helm install argocd bitnami/argo-cd -f argocd-values.yaml --set config.secret.argocdServerAdminPassword=Ayodeji79 -n argocd 

echo "Username: \"admin\""
  echo "Password: $(kubectl -n argocd get secret argocd-secret -o jsonpath="{.data.clearPassword}" | base64 -d)"

<!-- 2. Access The Argo CD API Server -->
kubectl apply -n argocd -f argocd/argocd-server-svc.yaml
#kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

<!-- 3. Download Argo CD CLI -->
brew install argocd

<!-- 4. Login Using The CLI -->
argocd login <ARGOCD_SERVER>
argocd account update-password

<!-- 5. Register A Cluster To Deploy Apps To -->
kubectl config get-contexts -o name
argocd cluster add docker-desktop