# -*- mode: ruby -*-
# vim: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "nfsserver", primary: true do |nfsserver|
    nfsserver.vm.box = "centos/7"
    nfsserver.vm.host_name = "nfs-server"
    nfsserver.vm.provider "virtualbox" do |v|
        v.memory = "1024"
        v.cpus = "2"
    end
    nfsserver.vm.network "private_network", ip: "192.168.0.32"
    nfsserver.vm.provision "shell", inline: <<-SHELL
        yum install -y nfs-utils
        mkdir -p /var/nfs_upload
        chown -R nfsnobody:nfsnobody /var/nfs_upload
        chmod -R 777 /var/nfs_upload
        sh -c "echo '/var/nfs_upload 192.168.0.42(rw,async,root_squash)' >> /etc/exports"
        sed -i 's/RPCNFSDARGS=""/RPCNFSDARGS="-N 2 -N 4"/g' /etc/sysconfig/nfs
        systemctl enable rpcbind nfs-server
        systemctl start rpcbind nfs-server
        exportfs -r
        systemctl enable firewalld
        systemctl start firewalld
        firewall-cmd --permanent --zone=public --add-service=nfs
        firewall-cmd --permanent --zone=public --add-service=mountd
        firewall-cmd --permanent --zone=public --add-service=rpc-bind
        firewall-cmd --reload
    SHELL
  end

  config.vm.define "nfsclient" do |nfsclient|
    nfsclient.vm.box = "centos/7"
    nfsclient.vm.host_name = "nfs-client"
    nfsclient.vm.provider "virtualbox" do |v|
        v.memory = "512"
        v.cpus = "1"
    end
    nfsclient.vm.network "private_network", ip: "192.168.0.42"
    nfsclient.vm.provision "shell", inline: <<-SHELL
        yum install -y nfs-utils
        systemctl enable rpcbind
        systemctl start rpcbind
        mkdir -p /mnt/nfs_share
        sh -c "echo '192.168.0.32:/var/nfs_upload  /mnt/nfs_share  nfs  defaults  0 0' >> /etc/fstab"
        mount -t nfs 192.168.0.32:/var/nfs_upload /mnt/nfs_share
    SHELL
  end
end
