# astra-2024-demo

## Настройка имен

Зайти в файл - nano /etc/hosts - Файл будет иметь вид

127.0.0.1	localhost

127.0.0.1 	astra

astra – необходимо изменить в соответствии с таблицей

Зайти в файл - nano /etc/hostname – файл будет иметь вид

astra – необходимо изменить в соответствии с таблицей
на машине которая будет использоваться в качестве сервера имен (dns) написать hq-srv.hq.work

после проделанных действий машину необходимо перезагрузить

---
## Настройка Сети

nano /etc/network/interfaces – в этом файле необходимо добавить записи

auto eth0 – номер интерфейса

iface eth0 inet static/dhcp – в зависимости от типа адреса (в случае если выбран dhcp 
следующие записи не нужны)

address 10.10.10.10 – ip адрес в соответствии с таблицей

netmask 255.255.255.0 – в соответсвии с таблицей

gateway 10.10.10.1 – если есть\

#### Далее мы включаем и запускаем службу resolvconf:

sudo apt install resolvconf

$ sudo systemctl enable resolvconf.service

$ sudo systemctl start resolvconf.service

Затем мы можем проверить состояние службы:

$ sudo systemctl status resolvconf.service

Зайти в файл - Nano /etc/resolv.conf – где необходимо прописать

Nameserver 172.16.100.10

Nameserver 8.8.8.8

### Следующие действия выполняются на роутерах

Зайти в файл - nano /etc/network/interfaces

auto gre1

iface gre1 inet static

address 10.0.0.1
        
netmask 255.255.255.252

mtu 1400

up ifconfig gre1 multicast

pre-up iptunnel add gre1 mode gre remote 20.20.20.10 local 10.10.10.10 dev eth0

pointopoint 10.0.0.2

post-down iptunnel del gre1
        
сохранить и выйти

Systemctl restart networking

---
## Включить пересылку пакетов (выполняется только на роутерах)

Зайти в файл - Nano /etc/sysctl.conf

Найти строчку «net.ipv4.ip_forward=1» и раскоментировать

sudo sysctl -p

Systemctl restart networking 

---
## Установка frr

apt-cdrom add

apt update 

apt install curl -y

mkdir /usr/share/keyrings – создать каталог для ключей 

Установить ключи - curl –s https://deb.frrouting.org/frr/keys.gpg | sudo tee /usr/share/keyrings/frrouting.gpg > /dev/null

Зайти в файл - nano /etc/apt/sources.list – в нем написать следующее
deb [signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr stretch frr-stable

Обновляемся и устанавливаем frr

apt update && apt install frr

---
## Настройка frr

nano /etc/frr/daemons

ospf = no – исправить на yes

ospf6d = no – исправить на yes

systemctl restart frr

Vtysh

configure

ip forwarding

int eth0 (интерфейс, который смотрит к конечному пользователю)

Ip ospf passive

Router ospf

Network 10.0.0.0/30 (подсеть gre туннеля и все подсети которые знает роутер кроме сетей которые смотрят на isp) area 0
Повторить для всех подсетей которые знает роутер

end

write

Повторить на всех роутерах 

Для того чтобы проверить необходимо написать команду

show ip route

Также после настройки на всех роутерах все узлы должны пинговаться друг с другом. 

---
## Установка и настройка dhcp

apt-cdrom add

apt install isc-dhcp-server –y

nano /etc/default/isc-dhcp-server

INTERFACES = “eth0” – указать интерфейс смотрящий во внутреннюю сеть.

Выйти и сохранить

nano /etc/dhcp/dhcp.conf

раскоментировать authoritative

создать запись типа 

subnet 172.16.100.0 netmask 255.255.255.0 {

range 172.16.100.10 172.16.100.50;

option routers 172.16.100.1;} – диапазон раздачи адресов

создать запись типа 

host hq-srv { 

hardware Ethernet 00:00:00:00:00:00; - mac адресс интерфеса хоста

fixed-address 172.16.100.10;} – выдача фиксированного адреса

сохранить и выйти

проверка файлов на ошибки 

dhcpd -t -cf /etc/default/isc-dhcp-server 

dhcpd -t -cf /etc/dhcp/dhcp.conf

systemctl restart isc-dhcp-server

---
## Создание пользователей

adduser username (имя пользователя)

---
## Установка iperf3 и замер пропускной способности

apt-cdrom add

apt install debian-archive-keyring dirmngr

nano /etc/apt/sources.list - добавить ссылку на репозиторий Debian:

deb https://archive.debian.org/debian/ stretch main contrib non-free

apt update

apt install iperf3

Iperf3 работает в клиент-серверном режиме. 

На ISP прописать:

iperf3 -s -запустить серверный режим

На HQ-R:

script -c ‘iperf3 -c 10.10.10.1(IPv4 ISP, смотрящий в сторону HQ-R) –-get-server-output' iperf3logfile.txt (файлжурнала)

cat iperf3_logfile.txt

#### Так же существует более простой способ установить iperf 1 версии

apt-cdrom add

apt install iperf

---
## Настройка SSH

### На сервере ssh

apt-cdrom add

apt update && apt install ssh

nano /etc/ssh/sshd_config

Найти строку #Port 22, раскоментировать и изменить на Port 2222

Найти строку #PermitRootLogin prohibit, раскоментировать изменить на PermitRootLogin no

добавить строку

AllowUsers admin netadmin

Сохранить и выйти

systemctl enable ssh

systemctl restart ssh 

### На хостах

ssh-keygen -t rsa (Жать enter, необходимо запомнить путь куда сохраняются ключи)

scp root/.ssh/id_rsa.pub admin@172.16.100.10 -p 2222:root/.ssh/authorized_keys

### на сервере 

nano /etc/ssh/sshd_config

найти строку и раскомментировать  - PubkeyAutentification yes

найти строку и раскомментировать - PasswordAuthentification yes

сохранить выйти 

### на hq-r

iptables -t nat -A PREROUTING -i eth2 -p tcp --dport 22 -j DNAT --to-destination 172.16.100.10:2222

В каталоге /etc/network/if-post-down.d/ создать файл, например iptables, со следующим содержимым:

nano /etc/network/if-post-down.d/iptables

#!/bin/sh

touch /etc/iptables.rules

chmod 640 /etc/iptables.rules

iptables-save > /etc/iptables.rules

exit 0

Сделать созданный сценарий исполнимым, выполнив в терминале команду:

sudo chmod +x /etc/network/if-post-down.d/iptables

В каталоге /etc/network/if-pre-up.d/ создать файл, например iptables, со следующим содержимым:

nano /etc/network/if-pre-up.d/iptables

#!/bin/sh

iptables-restore < /etc/iptables.rules

exit 0

Сделать созданный сценарий исполнимым, выполнив в терминале команду:

sudo chmod +x /etc/network/if-pre-up.d/iptables

### ограничение доступа для CLI

### на сервере ssh 

файл: sudo nano /etc/hosts.deny и запретить все подключения от CLI:

SSHD: 30.30.30.10

SSHD: 172.16.200.10

Чтобы разрешить подключения со всех устройств, кроме CLI нужно зайти в файл: sudo nano /etc/hosts.allow

SSHD: 172.16.100.0/26

SSHD: 192.168.10.0/28

SSHD: 10.10.10.0/24

SSHD: 20.20.20.0/24

SSHD: 30.30.30.1

---
## Создание Backup скриптов

создадим простой bash-скрипт: sudo nano backup-script.sh

#!/bin/sh

backup_files="/home /etc /root /boot /opt"

dest="/home/admin "

day=$(date +%A)

hostname=$(hostname -s)

archive_file="$hostname-$day.tgz"

echo "Backing up $backup_files to $dest/$archive_file"

date

echo

sudo tar -cvpz $backup_files | ssh 172.16.100.50 –p 2222 "( cat > $dest/$archive_file)" (На BR-R убрать часть –p 2222)

echo

echo "Backup finished"

date

сохранить и выйти

chmod +x backup-script.sh

nano /etc/crontab 

0 1 * * * ~/backup-script.sh

---
## Настройка синхронизации времени NTP сервер

НА HQ-R:

systemctl enable ntp

systemctl start ntp

nano /etc/ntp.conf

Стереть адреса ntp-серверов, а именно удалить строки pool 0, pool 1… и строки server.

На месте, где были прописаны записи server, написать

Server 127.0.0.1 - Loopback интерфейс HQ-R

Fudge 127.0.0.1 stratum 5 - стратум 5 Найти строку 

#restrict 192.168.1.0 mask 255.255.255.0 notrust

раскоментировать и изменить адрес подсети restrict 172.16.100.0 mask 255.255.255.192 notrust (172.16.100.0 – адрес сети hq)

Сохранить и выйти

systemctl restart ntp

НА КЛИЕНТАХ:

systemctl enable ntp

systemctl start ntp

systemctl nano /etc/ntp.conf

Стереть адреса ntp-серверов, а именно удалить строки pool 0, pool 1… и строки server изменить на server hq-r.hq.work iburst

Сохранить и выйти

sudo systemctl restart ntp

sudo ntpq -p – посмотреть список синхронизированных серверов ntp

0 */3 * * * root /usr/sbin/ntpdate hq-r.hq.work

sudo nano /etc/cron.d/ntpdate – создать автоопрос сервера времени

---
## Настройка dns и домена посредством freeipa

apt update

apt install astra-freeipa-server

krb5_newrealm

astra-freeipa-server –o

Переходим на CLI открываем браузер и пишем https://hq-srv.hq.work – имя машины на которой установлен freeipa
