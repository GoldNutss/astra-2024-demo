# astra-2024-demo
Топология сети
Предварительная настройка
Каждой машине в настройках вм выдать необходимое количество сетевых интерфесов. Все интерфейсы настроены как intent кроме 4го интерфейса у ISP через который будет осуществляться доступ в интернет, его настраиваем как сеть NAT.
Настройка имен
Зайти в файл
nano /etc/hosts - Файл будет иметь вид
127.0.0.1	localhost
127.0.1.1	astra 
astra – необходимо изменить в соответствии с таблицей
Зайти в файл
nano /etc/hostname – файл будет иметь вид
astra – необходимо изменить в соответствии с таблицей
на машине которая будет использоваться в качестве сервера имен (dns) написать hq-srv.hq.work
Настройка Сети
nano /etc/network/interfaces – в этом файле необходимо добавить записи
auto eth0 – номер интерфейса
iface eth0 inet static/dhcp – в зависимости от типа адреса (в случае если выбран dhcp 
следующие записи не нужны)
address 10.10.10.10 – ip адрес в соответствии с таблицей
netmask 255.255.255.0 – в соответсвии с таблицей
gateway 10.10.10.1 – если есть
dns-nameservers 172.16.100.10 – адрес днс сервера пишется 1 раз не для каждого интерфейса
выйти и сохранить
Включить пересылку пакетов (выполняется только на роутерах)
Зайти в файл
Nano /etc/sysctl.conf
Найти строчку «net.ipv4.ip_forward=1» и раскоментировать
Зайти в файл
Nano /etc/resolv.conf – где необходимо прописать
Nameserver 172.16.100.10
Nameserver 8.8.8.8
Выйти из файла и прописать 
Systemctl restart networking
Настройка доступа в интернет на ISP и раздача его в соседние сети 
iptables -t nat -A POSTROUTING -s 172.16.100.0/26 –o eth3 -j MASQUERADE
172.16.100.0 – сеть которой необходимо дать доступ к интернету
Eth3 – интерфейс который подключен к сети NAT
Повторить для каждой сети.
После этого надо сохранить эти правила. Для этого потребуется сделать следующие действия.
nano /etc/network/if-post-down.d/iptables
#!/bin/sh
touch /etc/iptables_rules
chmod 640 /etc/iptables_rules
iptables-save > /etc/iptables.rules
exit 0
Выйти из файла
chmod +x /etc/network/if-post-down.d/iptables
Далее
Sudo nano /etc/network/if-pre-up.d/iptables
#!/bin/sh
iptables-restore < /etc/iptables.rules
exit 0
Выйти из файла
chmod +x  /etc/network/if-pre-up.d/iptables
Установка frr
apt-cdrom add
apt update 
apt install curl -y
mkdir /usr/share/keyrings – создать каталог для ключей 
Установить ключи
curl –s https://deb.frrouting.org/frr/keys.gpg | sudo tee /usr/share/keyrings/frrouting.gpg > /dev/null
Зайти в файл 
nano /etc/apt/sources.list – в нем написать следующее
deb https://dl.astralinux.ru/astra/stable/orel/repository orel main contrib non-free
Зайти в файл
nano /etc/apt/sources.list.d/frr.list – в нем написать следующее
deb [trusted=true signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr stretch frr-stable
Обновляемся и устанавливаем frr
apt update && apt install frr frr-pythontools
Настройка frr
nano /etc/frr/daemons
ospf = no – исправить на yes
systemctl restart frr
Vtysh
configure
ip forwarding
int eth0 (интерфейс, который смотрит к конечному пользователю)
Ip ospf passive
Router ospf
Network 10.10.10.0/24 (подсеть которую знает настраиваемый роутер) area 0
Повторить для всех подсетей которые знает роутер
end
write
Повторить на всех роутерах 
Для того чтобы проверить необходимо написать команду
show ip route
Также после настройки на всех роутерах все узлы должны пинговаться друг с другом. 
Установка и настройка dhcp
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
hardware Ethernet 00:00:00:00:00:00;
fixed-address 172.16.100.10;} – выдача фиксированного адреса
сохранить и выйти
systemctl restart isc-dhcp-server
Создание пользователей
adduser username (имя пользователя)
Установка iperf3 и замер пропускной способности
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
iperf3 -c 10.10.10.1 (ip ISP)
Так же существует более простой способ установить iperf 1 версии
apt-cdrom add
apt install iperf
Настройка SSH
apt-cdrom add
apt update && apt install ssh
nano /etc/ssh/sshd_config
Найти строку #Port 22, раскоментировать и изменить на Port 2222
Найти строку #PermitRootLogin prohibit, раскоментировать изменить на PermitRootLogin ye	
добавить строку
	AllowUsers admin netadmin
	Сохранить и выйти
systemctl restart ssh 
данные манипуляции проделываются на сервере ssh
nano /etc/ssh/ssh_config
	раскоментировать “#Port 22” и изменить на “Port 2222”
ssh admin@172.16.100.10 (IP HQ-SRV) – подключение к серверу  ssh
Настройка синхронизации времени NTP сервер
НА HQ-R:
systemctl enable ntp
	systemctl start ntp
nano /etc/ntp.conf
	Стереть адреса ntp-серверов, а именно удалить строки 
pool 0, pool 1… 
и строки 
server. 
На месте, где были прописаны записи server, написать
Server 127.0.0.1 - Loopback интерфейс HQ-R
Fudge 127.0.0.1 stratum 5 - стратум 5 Найти строку 
#restrict 192.168.1.0 mask 255.255.255.0 notrust
раскоментировать и изменить адрес подсети на 
И изменить её на restrict 172.16.100.0 mask 255.255.255.192 notrust 
172.16.100.0 – адрес сети hq
Сохранить и выйти
systemctl restart ntp
НА КЛИЕНТАХ:
systemctl enable ntp
systemctl start ntp
systemctl nano /etc/ntp.conf
Стереть адреса ntp-серверов, а именно удалить строки 
pool 0, pool 1… 
и строки 
server изменить на server hq-r.hq.work iburst
Сохранить и выйти
sudo systemctl restart ntp
sudo ntpq -p – посмотреть список синхронизированных серверов ntp
0 */3 * * * root /usr/sbin/ntpdate hq-r.hq.work
sudo nano /etc/cron.d/ntpdate – создать автоопрос сервера времени
Настройка dns и домена посредством freeipa
 apt update
apt install astra-freeipa-server
krb5_newrealm
astra-freeipa-server –o
Переходим на CLI
Открываем браузер и пишем https://hq-srv.hq.work – имя машины на которой установлен freeipa

