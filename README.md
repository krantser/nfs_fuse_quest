# **Секция SHELL в файле vagrant для NFS сервера.**
```
# Устанавливаем пакет для работы NFS.
yum install -y nfs-utils
# Создаём директорию для общего доступа.
mkdir -p /var/nfs_upload
# Устанавливаем необходимые права и владельца на директорию общего доступа.
chown -R nfsnobody:nfsnobody /var/nfs_upload
chmod -R 777 /var/nfs_upload
# Добавляем в файл конфигурации для NFS сервера параметры.
# /var/nfs_upload - директория, для которой предоставляется общий доступ.
# 192.168.0.42 - ip адрес или это может быть подсеть, для которой разрешён доступ.
# (rw,async,root_squash) - набор опций для общего доступа.
# rw – доступ на чтение и запись
# async - установка ассинхронных операций ввода / вывода
# root_squash - подмена запросов от пользователя с uid/gid = 0 (root) на анонимные
# uid/gid.
sh -c "echo '/var/nfs_upload 192.168.0.42(rw,async,root_squash)' >> /etc/exports"
# Указываем в конфигурации NFS сервера запрет на использование протокола NFS v2 и v4
sed -i 's/RPCNFSDARGS=""/RPCNFSDARGS="-N 2 -N 4"/g' /etc/sysconfig/nfs
# Указываем системе автозапуск демонов NFS, а так же запускаем их.
systemctl enable rpcbind nfs-server
systemctl start rpcbind nfs-server
# Указываем демону nfs-server перечитать конфигурацию.
exportfs -r
# Указываем системе автозапуск демона файервола и запускаем его.
systemctl enable firewalld
systemctl start firewalld
# Задаём правила для файервола, что бы разрешались соедиения для NFS.
firewall-cmd --permanent --zone=public --add-service=nfs
firewall-cmd --permanent --zone=public --add-service=mountd
firewall-cmd --permanent --zone=public --add-service=rpc-bind
firewall-cmd --reload
```
# **Секция SHELL в файле vagrant для NFS клиента.**
```
# Устанавливаем пакет для работы NFS.
yum install -y nfs-utils
# Указываем системе автозапуск демона rpcbind для работы NFS и запускаем его.
systemctl enable rpcbind
systemctl start rpcbind
# Создаём директорию куда будет монтироваться ресурс общего доступа NFS.
mkdir -p /mnt/nfs_share
# Добавляем параметры автоматического монтирования ресурса при загрузке системы
# в /etc/fstab.
# Паратметры монтирования:
# auto - файловая системы будет смонтирована автоматически при загрузке.
# rw - монтировать файловую систему для чтения и записи.
# async - операции ввода/вывода должны выполняться асинхронно. 
# user - разрешить любому пользователю монтировать файловую систему.
# _netdev- файловая система находится на устройстве, которое требует доступ к сети                 
# (используется для предотвращения попытки системы монтировать ФС пока сеть не 
# будет доступна.
sh -c "echo '192.168.0.32:/var/nfs_upload  /mnt/nfs_share  nfs  auto,rw,async,user,_netdev  0 0' >> /etc/fstab"
# Монтируем ресурс.
mount -t nfs 192.168.0.32:/var/nfs_upload /mnt/nfs_share
```

# **Проверка работоспособности.**
Проверим созданную нами конфигурацию Vagrantfile:
```
vagrant validate
```
Вывод команды:
```
Vagrantfile validated successfully.
```
Всё в порядке и можно запускать виртуальные машины:
```
vagrant up
```
Когда машины успешно созданы и запущены подключаемся к NFS серверу:
```
vagrant ssh nfsserver
```
После подключения проверим какие версии NFS поддерживает сервер:
```
[vagrant@nfs-server ~]$ sudo cat /proc/fs/nfsd/versions 
```
Из вывода команды, видно, что поддерживается только версия 3:
```
-2 +3 -4 -4.0 -4.1 -4.2
```
Выполнив команду exportfs убедимся в том, что ресурс опубликован:
```
[vagrant@nfs-server ~]$ sudo exportfs
```
Вывод команды:
```
/var/nfs_upload
		192.168.0.42
```
Проверим статус работы файервола:
```
[vagrant@nfs-server ~]$ sudo firewall-cmd --state
```
Вывод команды:
```
running
```
Теперь проверим содержимое общего ресурса NFS:
```
[vagrant@nfs-server ~]$ ls -lh /var/nfs_upload/
```
Вывод команды:
```
total 0
```
Теперь выходим из серверной машины:
```
[vagrant@nfs-server ~]$ exit
```
Далее подключаемся к NFS клиенту:
```
vagrant ssh nfsclient
```
Проверяем примонтирован ли общий сетевой ресурс:
```
[vagrant@nfs-client ~]$ df -h
```
Вывод команды показывает, что ресурс примонтирован:
```
Filesystem                    Size  Used Avail Use% Mounted on
devtmpfs                      237M     0  237M   0% /dev
tmpfs                         244M     0  244M   0% /dev/shm
tmpfs                         244M  4.5M  240M   2% /run
tmpfs                         244M     0  244M   0% /sys/fs/cgroup
/dev/sda1                      40G  3.1G   37G   8% /
192.168.0.32:/var/nfs_upload   40G  3.1G   37G   8% /mnt/nfs_share
tmpfs                          49M     0   49M   0% /run/user/1000
```
Посмотрим более подробную информацию о монитровании ресурса:
```
[vagrant@nfs-client ~]$ mount | grep nfs
```
Вывод команды:
```
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw,relatime)
192.168.0.32:/var/nfs_upload on /mnt/nfs_share type nfs (rw,relatime,vers=3,
rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,
mountaddr=192.168.0.32,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,
addr=192.168.0.32)
```
Просмотрим содержимое общего сетевого ресурса:
```
[vagrant@nfs-client ~]$ ls -lh /mnt/nfs_share/
```
Вывод команды:
```
total 0
```
Теперь в общем сетевом ресурсе создадим файл:
```
[vagrant@nfs-client ~]$ cat /proc/cpuinfo >> /mnt/nfs_share/cpu_inf 
```
Ещё раз просмотрим содержимое общего сетевого ресурса:
```
[vagrant@nfs-client ~]$ ls -lh /mnt/nfs_share/
```
Вывод команды:
```
total 4.0K
-rw-rw-r--. 1 vagrant vagrant 787 Jul  4 20:39 cpu_inf
```
Как видим файл успешно создан.
Далее попробуем примонтировать ресурс используя в работе протокол NFS версии 4.
Сперва размонтируем наш ресурс:
```
[vagrant@nfs-client ~]$ sudo umount /mnt/nfs_share
```
А теперь смонтируем напрямую указав в параметрах работу по протоколу версии 4:
```
[vagrant@nfs-client ~]$ sudo mount -t nfs -o vers=4 192.168.0.32:/var/nfs_upload
```
В выводе команды видно, что работают наши ограничения указанные в настройках в
соответствии с заданием:
```
mount.nfs: Protocol not supported
```
Выходим из клиентской машины:
```
[vagrant@nfs-client ~]$ exit
```
Теперь снова зайдём на NFS сервер:
```
vagrant ssh nfsserver
```
И проверим содержимое общего ресурса:
```
[vagrant@nfs-server ~]$ ls -lh /var/nfs_upload/
```
В выводе команды видим созданный клиентом файл, это поддтверждает работоспособность
системы:
```
total 4.0K
-rw-rw-r--. 1 vagrant vagrant 787 Jul  4 20:39 cpu_inf
```
На данном этапе работа завершена.

