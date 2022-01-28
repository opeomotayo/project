helm repo add argo https://argoproj.github.io/argo-helm

helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace -f argocd-values.yaml

helm show values argo/argo-cd > values.yaml

kubectl apply -n argocd -f argocd-server-svc.yaml

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
Wcp9-lCYxDx-c52H


ref:
https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd