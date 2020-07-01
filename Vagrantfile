# -*- mode: ruby -*-
# vim: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "nfsserver" do |nfsserver|
    nfsserver.vm.box = "centos/7"
    nfsserver.vm.host_name = "nfs-server"
    nfsserver.vm.provider "virtualbox" do |v|
        v.memory = "1024"
        v.cpus = "2"
    end
    nfsserver.vm.provision "shell", inline: <<-SHELL
        yum install -y nfs-utils
        mkdir -p /var/nfs_zone
        chmod -R 777 /var/nfs_zone
    SHELL
  end

  config.vm.define "nfsclient" do |nfsclient|
    nfsclient.vm.box = "centos/7"
    nfsclient.vm.host_name = "nfs-server"
    nfsclient.vm.provider "virtualbox" do |v|
        v.memory = "512"
        v.cpus = "1"
    end
    nfsclient.vm.provision "shell", inline: <<-SHELL
          yum install -y nfs-utils
    SHELL
  end
end
