# image name to be used  as the base image for the hosts
IMAGE_NAME = "bento/ubuntu-20.04"
# subnet to be used for the nodes
SUBNET = "192.168.56."
# pod cidr to be used with kubeadm
POD_CIDR = "10.10.0.0/16"
# number of workers to be deployed
N = 1

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false
    config.vm.box_check_update = false

    config.vm.define "master" do |master|
        master.vm.box = IMAGE_NAME
        master.vm.network "private_network", ip: SUBNET + "#{50}"
        master.vm.hostname = "master"
		master.vm.synced_folder "data/", "/vagrant_data"
        master.vm.provider "virtualbox" do |vb|
            vb.memory = "2048"
            vb.name = "master"
            vb.cpus = 2
        end
		#master.vm.synced_folder "data/", "/vagrant_data", smb_username: "me", smb_password: "hophop"
        master.vm.provision "shell", path: "scripts/master-pre-req.sh" do |s|
		  s.args = [POD_CIDR, SUBNET + "#{50}"]
        end
    end

    (1..N).each do |i|
        config.vm.define "worker-#{i}" do |node|
            file_to_disk1 = "disks/sdb_disk_worker#{i}.vdi"
            # file_to_disk2 = "disks/sdc_disk_worker#{i}.vdi"
            # file_to_disk = File.realpath( "." ).to_s + "/disk.vdi"
            node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: SUBNET + "#{i + 50}"
            node.vm.hostname = "worker-#{i}"
			node.vm.synced_folder "data/", "/vagrant_data"
            node.vm.provider "virtualbox" do |vb|
                vb.memory = "8192"
                vb.name = "worker-#{i}"
                vb.cpus = 6

            end
			#node.vm.synced_folder "data/", "/vagrant_data", smb_username: "me", smb_password: "hophop"
			node.vm.provision "shell", path: "scripts/worker-pre-req.sh"
        end
    end

end


  