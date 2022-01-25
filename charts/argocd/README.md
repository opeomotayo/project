helm upgrade --install argocd argo/argo-cd -n argocd -f argocd-values.yaml

helm show values argo/argo-cd > argocd-values.yaml

kubectl apply -n argocd -f argocd-server-svc.yaml

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
6-0yiiaZS7QKCNPh


ref:
https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd