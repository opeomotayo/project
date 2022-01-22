helm upgrade --install minio --set persistence.enabled=false,rootUser=admin,rootPassword=Ayodeji79,resources.requests.memory=256Mi,replicas=4 --namespace minio minio/minio -f minio-values.yaml
helm show values minio/minio > minio_values.yaml


PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $PX_POD -n kube-system -- \
    /opt/pwx/bin/pxctl credentials create  \
    --provider s3   \
    --s3-access-key admin \
	--s3-secret-key Ayodeji79 \
    --s3-region minio --s3-endpoint http://192.168.56.50:32000 minio_creds

kubectl exec -it $PX_POD -n kube-system -- \
    /opt/pwx/bin/pxctl credentials list