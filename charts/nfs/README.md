kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl -n jenkins exec -it jenkins-0 -c config-reload -- cat /var/jenkins_home/casc_configs/jcasc-default-config.yaml kubectl -n jenkins exec -it jenkins-0 -c jenkins -- cat /var/jenkins_home/casc_configs/jcasc-default-config.yaml


{
wget https://get.helm.sh/helm-v3.5.3-linux-amd64.tar.gz;tar -xzvf helm-v3.5.3-linux-amd64.tar.gz;sudo mv linux-amd64/helm /usr/local/bin/helm;
}

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

sudo mount -t nfs 192.168.56.50:/srv/nfs/kubedata /mnt
sudo mount | grep kubedata
sudo umount /mnt

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.56.50 --set nfs.path=/srv/nfs/kubedata -f values.yaml -n nfs

kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'