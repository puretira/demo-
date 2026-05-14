Настройка Samba DC на BR-SRV
 Установка необходимых пакетов
apt-get update && apt-get install -y task-samba-dc
Развёртывание домена
Запустите интерактивное развёртывание:
samba-tool domain provision
При запросе параметров:
Realm: AU-TEAM.IRPO (подставится автоматически)
Domain: AU-TEAM (подставится автоматически)
Server Role: dc (нажмите Enter)
DNS backend: SAMBA_INTERNAL (нажмите Enter)
DNS forwarder IP address: 192.168.100.2 (IP HQ-SRV или другой DNS)
Настройка служб
Включаем и добавляем в автозагрузку службу samba:
systemctl enable --now samba
Настройка Kerberos:
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
Перезагружаем службу samba:
systemctl restart samba

Редактируем resolv.conf для интерфейса:
echo "search au-team.irpo" > /etc/net/ifaces/ens192/resolv.conf
echo "nameserver 127.0.0.1" >> /etc/net/ifaces/ens192/resolv.conf
Перезагружаем сеть:
systemctl restart network

Просмотр информации о домене:
samba-tool domain info 127.0.0.1
Проверка SMB-шар:
smbclient -L 127.0.0.1 -U administrator
apt-get install -y bind-utils

Проверка DNS-записей:
host au-team.irpo
host -t SRV _kerberos._udp.au-team.irpo
host -t SRV _ldap._tcp.au-team.irpo
host br-srv.au-team.irpo

Создание группы hq
samba-tool group add hq

for i in {1..5}; do
  samba-tool user add hquser$i P@ssw0rd
  samba-tool user setexpiry hquser$i --noexpiry
  samba-tool group addmembers "hq" hquser$i
done
 поменяйте  выдаваемый DHCP-сервером адрес DNS-сервера:  hq-rtr /etc/dhcp/dhcpd.conf
меняем адрес dns  с 192.168.100.2 на 192.168.0.2
systemctl restart dhcpd.service

ставим пакет автоматизации ввода клиентов  в домен
hq-cli    apt-get update && apt-get install -y task-auth-ad-sssd



RAID
lsblk
mdadm --zero-superblock --force /dev/sdb /dev/sdc
п это нормально
mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sdb /dev/sdc 

/dev/md0 — устройство RAID, которое появится после сборки
-l 0 — уровень RAID (0 = striping, чередование)
-n 2 — количество дисков в массиве
/dev/sdb /dev/sdc — диски 
Сохраняем конфигурацию /etc/mdadm.conf:
mdadm --detail --scan --verbose | tee -a /etc/mdadm.conf

создаем точку монтирования
mkdir /raid
редактируем файл /etc/fstab:
vi  /etc/fstab
вставляем строку
/dev/md0    /raid    ext4    defaults    0    0
 Монтируем

mount -av
проверяем что живой
mdadm --detail /dev/md0



Настройка NFS-сервера на HQ-SRV
Установка пакетов
apt-get install -y nfs-server nfs-utils
Создание директории общего доступа
Создаём директорию внутри RAID-массива:
mkdir /raid/nfs
Назначение прав
chmod 777 /raid/nfs
Настройка экспорта
Редактируем файл /etc/exports:
vim /etc/exports
Добавляем строку:
/raid/nfs    192.168.200.0/24(rw,no_root_squash)

/raid/nfs — директория для хранения
192.168.200.0/24 — сеть из  которой разрешено монтирование
rw — разрешены чтение и запись
no_root_squash — отключение ограничения прав root на клиенте = root на сервере
монтировние ФС
exportfs -avr
-avr — экспортировать все каталоги из /etc/exports, повторный экспорт всех каталогов (синхронизация), подробный вывод
автозагрузка
systemctl enable --now nfs-server

ansible на сервере BR-SRV
apt-get install -y ansible sshpass
настройка ансибли
cat <<EOF > /etc/ansible/ansible.cfg
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = Flase
EOF
cat <<EOF > /etc/ansible/hosts
HQ-SRV ansible_host=192.168.100.2 ansible_user=sshuser ansible_password=P@ssw0rd ansible_port=2026
HQ-CLI  ansible_host=192.168.200.2 ansible_user=user ansible_password=resu
HQ-RTR ansible_host=10.10.10.1 ansible_user=user ansible_password=resu
BR-RTR ansible_host=192.168.0.1  ansible_user=user ansible_password=resu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF

cd /etc/ansible/
ansible -m ping all



сервис сетевого времени

ISP
sed -i 's/^pool/#pool/' /etc/chrony.conf
 cat <<EOF >> /etc/chrony.conf
server ntp0.ntp-servers.net iburst prefer minstratum 4
local straum 5
allow 0.0.0.0/0
EOF
systemctl restart chronyd

HQ-rtr
sed -i 's/^pool/#pool/' /etc/chrony.conf
echo "server 172.16.1.1 iburst" >> /etc/chrony.conf
systemctl restart chronyd

на остальных
симметрично настраиваем hq-rtr но  в офисе branch меняем ip 172.16.2.1
проверка chronyc sources


Дохер

br-srv 
apt-get install -y docker -engine docker-compose-v2
systemctl enable --now docker.service

ссылка на сервер https://www.softportal.com/get-46141-xampp.html
образ ложим в папку  C:\xampp\htdocs\dashboard\(локальную папку создаем сами обзываем ее сами как вам нравиться в нутрь кладем iso файл)
wget http://192.168.23.70/dashboard/test/Additional.iso
монтируем образ mount /xxx/xxx - (откуда монтируем) /mnt/-(куда монтируем)
docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar
проверить что они есть
docker images ls
ни забывайти пробелов везде
Композе
cat <<EOF >compose.yaml
services:
   database:
        container_name: db
         image: mariadb: 10.11
         restart: always
         ports:
               - "3306:3306"
         environment:
            MARIADB_DATABASE: "testdb"
            MARIADB_USER: "testc"
            MARIADB_PASSWORD:"P@ssw0rd"
            MARIADB_ROOT_PASSWORD: "toor"

   app:
       container_name: testapp
       images: site: latest
       restart: always
       port: 
            - "8080:8000"
       environment:
             DB_TYPE: "maria"
             DB_HOST: "192.168.0.2"
             DB_PORT: "3306"
             DB_NAME: "testdb"
             DB_USER: "testc"
             DB_PASS: "P@ssw0rd"
       depends_on:
             - database
EOF
запуск контейнера
docker compose up -d
если все задеплоило норм то на HQ-CLI  в браузере по адресу 192.168.0.2:8080
будет сайт

             
            


