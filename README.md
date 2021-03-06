https://ohmyz.sh

STEP1:
<!-- In case you do not have helm here is the installation procedure, as I will be using helm later on: -->
{
wget https://get.helm.sh/helm-v3.5.3-linux-amd64.tar.gz;tar -xzvf helm-v3.5.3-linux-amd64.tar.gz;sudo mv linux-amd64/helm /usr/local/bin/helm;
}

STEP2:
<!-- ETCD CLUSTER: As indicated I need an Etcd instance in order to install Portworx. I will use the bitnami helm chart and local storage from the worker nodes. On each of these workers I create a folder and assign permissions: -->
{
sudo mkdir -p  /data/etcd
sudo chmod 771 /data/etcd
}

STEP3:
<!-- Apply the YAML spec to create three Local PVs exclusively associated with each Worker Node of the cluster.
kubectl apply -f pv-etcd.yaml
A PVC associated with each of these PVs will also be created beforehand. It’s important to use the naming convention that matches the etcd StatefulSet. This will ensure that the Pods from the StatefulSet use existing PVCs that are already bound to the PVs -->
kubectl apply -n kube-system -f etcd-volumes/pv-etcd.yaml 

apiVersion: v1
kind: PersistentVolume
metadata:
  name: etcd-vol-0
spec:
...

STEP4:
<!-- Let’s create three PVCs bound to these PVs.
Make sure that the PVs are created and PVCs from the kube-system Namespace are bound to them.
I can now create my three persistent volume claims using this definition file: -->
kubectl apply -n kube-system -f etcd-volumes/pvc-etcd.yaml 
kubectl get pv
kubectl get pvc -n kube-system

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-px-etcd-0
spec:
...


STEP5:
<!-- With the PVCs in place, we are ready to create the etcd cluster. We will use a Helm 3 etcd Chart for this step.
now add the Bitnami Helm repo and deploy the etcd cluster to the pxeckube-system namespace. Etcd will use the three PVC created in the prior steps:
Note that the Chart name (px-etcd) matches a part of the PVC (data-px-etcd-X). This is important to make sure that the Chart uses existing PVCs.
We are creating three Pods for the StatefulSet which will ensure that the etcd cluster is highly available.-->
helm repo add bitnami https://charts.bitnami.com/bitnami
<!-- helm upgrade --install px-etcd bitnami/etcd --debug --set replicaCount=3 --set auth.rbac.enabled=false --set persistence.enabled=true --set persistence.size=1Gi --set image.tag=3.4.15-debian-10-r33 --version 6.2.2 --set image.debug=true --namespace=kube-system -f ./helm/charts/etcd/values.yaml -->
<!-- helm upgrade --install px-etcd bitnami/etcd --debug --set replicaCount=3 --set auth.rbac.enabled=false --set persistence.enabled=true --set persistence.size=1Gi --set image.tag=3.5.1-debian-10-r52 --version 6.10.5 --set image.debug=true --namespace=kube-system -f ./helm/charts/etcd/values.yaml -->

helm upgrade --install px-etcd bitnami/etcd --debug --set replicaCount=3 --set auth.rbac.enabled=false --set persistence.enabled=true --set persistence.size=1Gi --set image.tag=3.5.1-debian-10-r52 --version 6.10.5 --set image.debug=true --set initialClusterState=new --namespace=kube-system -f ./helm/charts/etcd/values.yaml

<!-- takes about 2 minutes -->
kubectl get pods -l app.kubernetes.io/name=etcd -n=kube-system

<!-- to delete -->
<!-- helm delete px-etcd -n=kube-system -->

STEP6:
<!-- Additional test commands. The etcd Pods and related objects are deployed in the kube-system Namespace which is also used by Portworx deployment. Use px-etcd cluster ip to check etcd version -->
kubectl get svc -l app.kubernetes.io/name=etcd -n=kube-system
curl -L http://10.109.64.82:2379/version -GET -v
curl -L http://px-etcd-0.kube-system.svc:2379/version -GET -v
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 get "" --prefix --keys-only
kubectl run px-etcd-client --restart='Never' --image docker.io/bitnami/etcd:3.4.15-debian-10-r33 --env ETCDCTL_ENDPOINTS="px-etcd.kube-system.svc.cluster.local:2379" --namespace kube-system --command -- sleep infinity
kubectl exec -it px-etcd-client -n kube-system -- bash
kubectl exec -it px-etcd-0 -n kube-system -- bash

STEP7:
<!-- Installing Portworx Storage Cluster
Sign up at Portworx hub to access the Portworx installation wizard. Once logged in, click on the new spec to launch the wizard.
The first step is to provide the version of Kubernetes and the details of the etcd cluster. Copy the ClusterIP of etcd service available within the kube-system namespace and paste it in the wizard’s etcd textbox. Don’t forget to append the port of the service, Don't forget to update the cluster ip -->

#git clone https://github.com/portworx/helm.git
<!-- modify set params accordingly -->
helm upgrade --install px-cluster --debug --set managementInterface=eth1 --set dataInterface=eth1 --set drives=/dev/sdb --set etcdEndPoint=etcd:http://10.106.97.31:2379,clusterName=px-cluster -n=kube-system ./helm/charts/portworx/ -f ./helm/charts/portworx/values.yaml --dry-run

<!-- takes about 10 minutes -->
kubectl get pods -n kube-system -l name=portworx

<!-- curl -fsL "https://install.portworx.com/px-wipe" | bash -->
<!-- helm delete px-cluster -n kube-system -->

STEP8:
<!-- To use storage capacity allocated to Portworx, you should create a Portworx volume and expose it to your Pod. It can be done through manual pre-provisioning or using Kubernetes dynamic volume provisioning.
Pre-provisioning a Portworx Volume
You can pre-provision a Portworx volume using the pxctl tool. To access the tool, you can either get a shell to one of the PX Pods as we did in the example above or ssh to one of the nodes in your K8s cluster and access pxctl directly at /opt/pwx/bin/pxctl of your instance. Below is the example of the second option: -->
kubectl exec -it -n kube-system portworx-48m2l -- bash
/opt/pwx/bin/pxctl status
/opt/pwx/bin/pxctl volume create --size=1 --repl=2 --fs=ext4 test-disk1
/opt/pwx/bin/pxctl volume create --size=1 --repl=2 --fs=ext4 test-disk2
/opt/pwx/bin/pxctl volume create --size=1 --repl=2 --fs=ext4 test-disk3
kubectl apply -f test-applications/ -n kube-system

STEP9: 
<!-- To create backup -->
kubectl exec -it -n kube-system px-etcd-0 -- bash
etcdctl member list -w table
etcdctl endpoint health
etcdctl get "" --prefix --keys-only | sed '/^\s*$/d'
etcdctl --endpoints=http://localhost:2379 get "" --prefix --keys-only (testing from etcd-client remove --endpoints)
etcdctl --endpoints http://127.0.0.1:2379 snapshot save /tmp/mybackup
etcdctl --endpoints http://127.0.0.1:2379 snapshot status /tmp/mybackup-w table
kubectl cp kube-system/px-etcd-0:/tmp/ /vagrant/etcd-volumes


Step 10: REPEAT STEP1 TO 7, Copy the snapshot to a PVC
<!-- The etcd cluster can be restored from the snapshot created in the previous step. There are different ways to do this; one simple approach is to make the snapshot available to the pods using a Kubernetes PersistentVolumeClaim (PVC). Therefore, the next step is to create a PVC and copy the snapshot file into it. Further, since each node of the restored cluster will access the PVC, it is important to create the PVC using a storage class that supports ReadWriteMany access, such as NFS.
Begin by installing the NFS Server Provisioner. The easiest way to get this running on any platform is with the stable Helm chart. Use the command below, remembering to adjust the storage size to reflect your cluster's settings: -->

{
  sudo apt update -y
  sudo apt install nfs-kernel-server -y
}
{
  sudo apt update -y
  sudo apt install nfs-common -y
}
{
sudo mkdir -p /srv/nfs/kubedata
sudo chown nobody: /srv/nfs/kubedata
}
sudo vi /etc/exports
/srv/nfs/kubedata  *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
sudo exportfs -rav

sudo mount -t nfs 192.168.56.25:/srv/nfs/kubedata /mnt
sudo mount | grep kubedata
sudo umount /mnt

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.56.25 \
    --set nfs.path=/srv/nfs/kubedata -n kube-system

<!-- Create a Kubernetes manifest file named etcd.yaml to configure an NFS-backed PVC and a pod that uses it, as below: -->
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: etcd-backup-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
      ...




kubectl apply -f etcd-volumes/etcd-nfs-claim.yaml -n kube-system
kubectl cp etcd-volumes/mybackup kube-system/etcd-backup-pod:/backup/mybackup
kubectl exec -it etcd-backup-pod -n kube-system -- ls -al /backup

{
sudo chmod -R 777 etcd-volumes/mybackup
sudo chown -R 1001 etcd-volumes/mybackup
}
kubectl -n kube-system delete po etcd-backup-pod 
<!-- Step 11: Restore the snapshot in a new cluster
The next step is to create an empty etcd deployment on the destination cluster and restore the data snapshot into it. The Bitnami etcd Helm chart provides built-in capabilities to do this, via its startFromSnapshot.* parameters.
Create a new etcd deployment. Replace the PASSWORD placeholder with the same password used in the original deployment. -->
helm delete px-etcd -n kube-system
kubectl -n kube-system apply -f etcd-volumes/pv-etcd.yaml
kubectl -n kube-system apply -f etcd-volumes/pvc-etcd.yaml 



helm upgrade --install px-etcd bitnami/etcd --debug --set replicaCount=3 --set auth.rbac.enabled=false --set persistence.enabled=true --set persistence.size=1Gi --set image.tag=3.5.1-debian-10-r52 --version 6.10.5 --set image.debug=true --set startFromSnapshot.enabled=true --set startFromSnapshot.existingClaim=etcd-backup-pvc --set startFromSnapshot.snapshotFilename=mybackup --namespace=kube-system -f ./helm/charts/etcd/values.yaml
kubectl -n kube-system get po
kubectl -n kube-system logs px-etcd-0
kubectl -n kube-system exec -it px-etcd-0 -- bash

Repeat step2 to 4
<!-- helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install px-etcd bitnami/etcd --debug \
  --set replicaCount=3 \
  --set startFromSnapshot.enabled=true \
  --set startFromSnapshot.existingClaim=data-px-etcd-0 \
  --set startFromSnapshot.snapshotFilename=mybackup \
  --set auth.rbac.enabled=false \
  --set persistence.enabled=true \
  --set persistence.size=2Gi \
  --set image.tag=3.4.15-debian-10-r33 \
  --version 6.2.2 \
  --set image.debug=true -n=kube-system -f ./helm/charts/etcd/values.yaml --dry-run -->

<!-- This command creates a new etcd cluster and initializes it using an existing data snapshot. The startFromSnapshot.existingClaim and startFromSnapshot.snapshotFilename define the source PVC and source filename for the snapshot respectively.
It is important to create the new deployment on the destination cluster using the same credentials as the original deployment on the source cluster. -->

Step12:
<!-- Connect to the new deployment and confirm that your data has been successfully restored. Replace the PASSWORD placeholder with the administrator password set at deployment-time. -->

export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=etcd,app.kubernetes.io/instance=etcd-new" -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it $POD_NAME -- etcdctl --user root:PASSWORD get /message1
kubectl exec -it $POD_NAME -- etcdctl --user root:PASSWORD get /message2


delete data-dir

<!-- Here, we used pxctl volume create command that has the following format: pxctl volume create[command options] volume-name . This command creates a 5GB volume named “test-disk” with the ext4 file system and two copies across the cluster. We specified the number of copies to create using the--repl argument. Please check the official pxctl CLI reference for more information about this command.
Dynamic Volume Provisioning

With the DVP, you don’t need to pre-provision Portworx volumes before using them in your applications. Cluster administrators can create Storage Classes that define different classes of Portworx Volumes offered in the cluster. Thereafter, applications can request dynamic provisioning of these volumes.

Below is the example of the Portworx StorageClass: -->


helm install etcd bitnami/etcd \
--version 6.2.2 \
--set replicaCount=3 \
--set image.tag=3.4.15-debian-10-r33 \
--set image.debug=true \
--set updateStrategy.type=RollingUpdate \
--set metrics.enabled=true \
--set auth.rbac.enabled=false \
--set disasterRecovery.enabled=true \
--set disasterRecovery.cronjob.schedule='*/10 * * * *' \
--set disasterRecovery.pvc.existingClaim=etcd-snapshot-volume \
--set persistence.enabled=true \
--set persistence.storageClass=rook-ceph-block \
--set persistence.size=1Gi \
--set podAntiAffinity=soft \
--set startFromSnapshot.enabled=true \
--set startFromSnapshot.existingClaim=restore \
--set startFromSnapshot.snapshotFilename=db \
--set initialClusterState=new 



there's no need to modify the etcd ansible file
ref: https://ystatit.medium.com/backup-and-restore-kubernetes-etcd-on-the-same-control-plane-node-20f4e78803cb
**backup
kubectl exec -it -n kube-system portworx-48m2l -- bash
/opt/pwx/bin/pxctl volume create --size=1 --repl=2 --fs=ext4 test-disk1
/opt/pwx/bin/pxctl volume create --size=1 --repl=2 --fs=ext4 test-disk2
 kubectl exec -it -n kube-system px-etcd-0 -- bash
 etcdctl member list -w table
 etcdctl endpoint status --write-out=table
etcdctl endpoint health
etcdctl --endpoints=http://localhost:2379 get "" --prefix --keys-only (testing from etcd-client remove --endpoints)
etcdctl --endpoints http://127.0.0.1:2379 snapshot save /tmp/mysnapshot.db





kubectl port-forward --namespace kube-system svc/px-etcd 2379:2379 &
mkdir etcd-backup
chmod o+w etcd-backup
cd etcd-backup
docker run -it --env ALLOW_NONE_AUTHENTICATION=yes --rm --network host  -v $(pwd):/backup bitnami/etcd etcdctl --endpoints http://127.0.0.1:2379 snapshot save /backup/mybackup
sudo chmod -R 644 mybackup


etcdctl --endpoints=http://0.0.0.0:2379 snapshot save snapshot.db
etcdctl --endpoints=http://0.0.0.0:2379 snapshot status snapshot.db -w table
etcdctl --endpoints=http://0.0.0.0:2379 endpoint health

**restore
etcdctl snapshot restore snapshot-full.db
cp -r default.etcd/member/ /var/lib/etcd/
systemctl restart etcd
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 get "" --prefix --keys-only

docker run --restart=always --name macbook-cluster -d --net=host --privileged=true -v /run/docker/plugins:/run/docker/plugins -v /var/lib/osd:/var/lib/osd:shared -v /dev:/dev -v /etc/pwx:/etc/pwx -v /opt/pwx/bin:/export_bin:shared -v /var/run/docker.sock:/var/run/docker.sock -v /mnt:/mnt:shared -v /var/cores:/var/cores -v /usr/src:/usr/src portworx/px-enterprise -m team0:0 -d team0

current issues
==============
etcdctl member list is not showing the 3 members => done
metallb lb ip addr not connecting to any member => not needed
/opt/pwx/bin/pxctl status is not showing the 3 nodes => done
don't know how to customize portworx helm chart to use a specific disk drive (values.yaml?) 
don't know how to customize etcd helm chart to use a specific configuratin (values.yaml?)



ETCDCTL_API=3 etcdctl endpoint health


<!-- MetalLB: I will be using MetalLB to provide an IP address to an etcd instance. Portworx requires an externally reachable etcd for its kvdb when implemented in a disaggregated architecture.
To install MetalLB: -->
{
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml ;kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml ;
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)" ;
}

<!-- I then create and apply the following configmap where a set of available IPs is declared for use by MetalLB. -->
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.80.1-192.168.80.5

<!-- I then apply this to the Kubernetes storage cluster: -->
kubectl -n metallb-system apply -f metallb-cm.yaml

{
sudo chmod 777 /etc/cni/net.d
sudo chown -R $USER:$USER ~/.kube
sudo systemctl restart kubelet
}

sudo systemctl status kubelet


<!-- https://docs.bitnami.com/tutorials/backup-restore-data-etcd-kubernetes/ -->
helm install etcd-new bitnami/etcd \
  --set startFromSnapshot.enabled=true \
  --set startFromSnapshot.existingClaim=etcd-backup-pvc \
  --set startFromSnapshot.snapshotFilename=mybackup \
  --set auth.rbac.rootPassword=PASSWORD \
  --set replicaCount=3





helm upgrade --install px-etcd bitnami/etcd --debug --set replicaCount=3 --set auth.rbac.enabled=false --set persistence.enabled=true --set persistence.size=1Gi --set image.tag=3.5.1-debian-10-r52 --version 6.10.5 --set image.debug=true --set startFromSnapshot.enabled=true --set startFromSnapshot.existingClaim=etcd-backup-pvc --set startFromSnapshot.snapshotFilename=mybackup --namespace=kube-system -f ./helm/charts/etcd/values.yaml



netstat -tlpn | grep 9001
iptables -A INPUT -p tcp --match multiport --dports 9001:9022 -j ACCEPT
iptables -A INPUT -p tcp --match multiport --dports 111:2049 -j ACCEPT
Semyon Kolesnikov to Everyone (10:19)
sudo iptables -L
Semyon Kolesnikov to Everyone (10:28)
PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
kubectl describe pods -l name=portworx -n kube-system
Semyon Kolesnikov to Everyone (10:31)
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
journalctl -lu portworx* > /tmp/px-fail.log
Semyon Kolesnikov to Everyone (10:37)
kubectl get pods POD_NAME_HERE -o jsonpath='{.spec.containers[*].name}'
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl volume list


kubectl logs -n kube-system -l name=portworx -c portworx --tail=99999 > pwx.logs

 cd /etc/pwx/
 cat config.json

etcdctl get pwx/px-cluster/osdconfig/nodeConf/da283783-4532-43c5-b2bc-b6f326653bcd --prefix --print-value-only | sed '/^\s*$/d'
etcdctl get pwx/px-cluster/cluster/database --prefix --print-value-only | sed '/^\s*$/d'


pwx/px-cluster/osdconfig/nodeConf/3af61bce-cde8-4601-8636-247f9aa7f290 ==> 16
pwx/px-cluster/osdconfig/nodeConf/4440f236-5200-4a74-bdf8-9008378f1cb8
pwx/px-cluster/osdconfig/nodeConf/7aa19f7f-36db-448e-9f72-f41c0cfac149 ==>17
pwx/px-cluster/osdconfig/nodeConf/82e10104-1d2d-443b-ad0e-8d5acde335a1
pwx/px-cluster/osdconfig/nodeConf/da283783-4532-43c5-b2bc-b6f326653bcd ==>18

etcdctl del pwx/px-cluster/osdconfig/nodeConf/da283783-4532-43c5-b2bc-b6f326653bcd --prefix
etcdctl del pwx/px-cluster/storage/runtime/da283783-4532-43c5-b2bc-b6f326653bcd --prefix


<!-- rm -rf etc/pwx/.private.json
29  ls /etc/pwx/
30  ls -la /etc/pwx/
31  rm -rf /etc/pwx/.private.json
32  ls -la /etc/pwx/
33  /opt/pwx/bin/pxctl service node-wipe -->
/opt/pwx/bin/pxctl service node-wipe