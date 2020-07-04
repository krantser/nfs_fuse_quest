# **Секция SHELL в файле vagrant для NFS сервера.**

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

# **Секция SHELL в файле vagrant для NFS клиента.**
yum install -y nfs-utils
systemctl enable rpcbind
systemctl start rpcbind
mkdir -p /mnt/nfs_share
sh -c "echo '192.168.0.32:/var/nfs_upload  /mnt/nfs_share  nfs  defaults  0 0' >> /etc/fstab"
mount -t nfs 192.168.0.32:/var/nfs_upload /mnt/nfs_share

[vagrant@nfs-server ~]$ sudo cat /proc/fs/nfsd/versions 
-2 +3 -4 -4.0 -4.1 -4.2

