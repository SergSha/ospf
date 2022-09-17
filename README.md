<h3>### OSPF ###</h3>

<h4>Описание домашнего задания</h4>

<p>OSPF</p>

<ol>
<li>Поднять три виртуалки</li>
<li>Объединить их разными vlan<br />
● поднять OSPF между машинами на базе Quagga;<br />
● изобразить ассиметричный роутинг;<br />
● сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.<br />
Формат сдачи ДЗ - vagrant + ansible</li>
</ol>

<h4>1. Разворачиваем 3 виртуальные машины</h4>

<p>Так как мы планируем настроить OSPF, все 3 виртуальные машины должны быть соединены между собой (разными VLAN), а также иметь одну (или несколько) доолнительных сетей, к которым, далее OSPF сформирует маршруты. Исходя из данных требований, мы можем нарисовать топологию сети:</p>

<img src="images/network.png" alt="network.png" />

<p>Обратите внимание, сети, указанные на схеме не должны использоваться в Oracle Virtualbox, иначе Vagrant не сможет собрать стенд и зависнет. По умолчанию Virtualbox использует сеть 10.0.2.0/24. Если была настроена другая сеть, то проверить её можно в настройках программы: <br />
<b>VirtualBox — File — Preferences — Network — щёлкаем по созданной сети</b></p>

<p>Создаём каталог, в котором будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./ospf
[user@localhost otus]$</pre>

<p>Перейдём в каталог ospf:</p>

<pre>[user@localhost otus]$ cd ./ospf/
[user@localhost ospf]$</pre>

<p>В каталоге создаём файл с именем Vagrantfile, добавляем в него следующее содержимое:</p>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :router1 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router1",
    :net => [
      {ip: '10.0.10.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
      {ip: '10.0.12.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
      {ip: '192.168.10.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net1"},
      {ip: '192.168.50.10', adapter: 5},
    ]
  },
  :router2 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router2",
    :net => [
      {ip: '10.0.10.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
      {ip: '10.0.11.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
      {ip: '192.168.20.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net2"},
      {ip: '192.168.50.11', adapter: 5},
    ]
  },
  :router3 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router3",
    :net => [
      {ip: '10.0.11.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
      {ip: '10.0.12.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
      {ip: '192.168.30.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net3"},
      {ip: '192.168.50.12', adapter: 5},
    ]
  }
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      if boxconfig[:vm_name] == "router3"
        box.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/playbook.yml"
          ansible.inventory_path = "ansible/hosts"
          ansible.host_key_checking = "false"
          ansible.limit = "all"
        end
      end
      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", ipconf
      end
    end
  end
end</pre>

<p>Запустим эти виртуальные машины:</p>

<pre>[user@localhost ospf]$ vagrant up</pre>

<p>Данный Vagrantfile развернет нам 3 хоста: router1, router2 и router3:</p>

<pre>[user@localhost ospf]$ vagrant status
Current machine states:

router1                   running (virtualbox)
router2                   running (virtualbox)
router3                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost ospf]$</pre>

<p>Все эти три виртуальные машины соединены между собой сетями (10.0.10.0/30, 10.0.11.0/30 и 10.0.12.0/30). У каждого роутера есть дополнительная сеть:<br />
● на router1 — 192.168.10.0/24<br />
● на router2 — 192.168.20.0/24<br />
● на router3 — 192.168.30.0/24</p>

<p>На данном этапе ping до дополнительных сетей (192.168.10-30.0/24) с соседних роутеров будет недоступен.</p>

<p>Подключимся по ssh, например, к машине router1: </p>

<pre>[user@localhost ospf]$ vagrant ssh router1
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-125-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Sep 16 19:23:51 UTC 2022

  System load:     0.04              IPv4 address for enp0s10: 192.168.10.1
  Usage of /:      3.5% of 38.70GB   IPv4 address for enp0s16: 192.168.50.10
  Memory usage:    22%               IPv4 address for enp0s3:  10.0.2.15
  Swap usage:      0%                IPv4 address for enp0s8:  10.0.10.1
  Processes:       121               IPv4 address for enp0s9:  10.0.12.1
  Users logged in: 0


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
New release '22.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


vagrant@router1:~$</pre>

<p>Переключимся в root пользователя:</p>

<pre>vagrant@router1:~$ sudo -i
root@router1:~#</pre>

<h4>Установка пакетов для тестирования и настройки OSPF</h4>

<p>Перед настройкой FRR установим базовые программы для изменения конфигурационных файлов (vim) и изучения сети (traceroute, tcpdump, net-tools):</p>

<pre>root@router1:~# apt -y update && apt -y install vim traceroute tcpdump net-tools
...
root@router1:~#</pre>

<h4>2.1 Настройка OSPF между машинами на базе Quagga</h4>

<p>Пакет Quagga перестал развиваться в 2018 году. Ему на смену пришёл пакет FRR, он построен на базе Quagga и продолжает своё развитие. В данном случае настойка OSPF будет осуществляться в FRR.<p>

<p>Процесс установки FRR и настройки OSPF вручную:<br />
1) Отключаем файерволл ufw и удаляем его из автозагрузки:</p>

<pre>root@router1:~# systemctl stop ufw
root@router1:~# systemctl disable ufw
Synchronizing state of ufw.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable ufw
Removed /etc/systemd/system/multi-user.target.wants/ufw.service.
root@router1:~#</pre>

<p>2) Добавляем gpg ключ:</p>

<pre>root@router1:~# curl -s https://deb.frrouting.org/frr/keys.asc | apt-key add -
OK
root@router1:~#</pre>

<p>3) Добавляем репозиторий c пакетом FRR:</p>

<pre>root@router1:~# echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable > /etc/apt/sources.list.d/frr.list
root@router1:~#</pre>

<p>4) Обновляем пакеты и устанавливаем FRR:</p>

<pre>root@router1:~# apt -y update && apt -y install frr frr-pythontools
...
root@router1:~#</pre>

<p>5) Разрешаем (включаем) маршрутизацию транзитных пакетов:</p>

<pre>root@router1:~# sysctl net.ipv4.conf.all.forwarding=1
net.ipv4.conf.all.forwarding = 1
root@router1:~#</pre>

<p>6) Включаем демон ospfd в FRR<br />
Для этого открываем в редакторе файл /etc/frr/daemons и меняем в нём параметры для пакетов zebra и ospfd на yes:</p>

<pre>root@router1:~# nano /etc/frr/daemons</pre>

<pre>bgpd=no
<b>ospfd=yes</b>
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no

vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
ospfd_options="  -A 127.0.0.1"
ospf6d_options=" -A ::1"
ripd_options="   -A 127.0.0.1"
ripngd_options=" -A ::1"
isisd_options="  -A 127.0.0.1"
pimd_options="   -A 127.0.0.1"
ldpd_options="   -A 127.0.0.1"
nhrpd_options="  -A 127.0.0.1"
eigrpd_options=" -A 127.0.0.1"
babeld_options=" -A 127.0.0.1"
sharpd_options=" -A 127.0.0.1"
pbrd_options="   -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
bfdd_options="   -A 127.0.0.1"
fabricd_options="-A 127.0.0.1"
vrrpd_options="  -A 127.0.0.1"</pre>

<p>Параметр пакета zebra включен по умолчанию.</p>

<p>7) Настройка OSPF<br />
Для настройки OSPF нам потребуется создать файл /etc/frr/frr.conf, который будет содержать в себе информацию о требуемых интерфейсах и OSPF. Разберем пример создания файла на хосте router1.</p>

<p>Для начала нам необходимо узнать имена интерфейсов и их адреса. Сделать это можно с помощью двух способов:<br />
● Посмотреть в linux: ip a | grep inet</p>

<pre>root@router1:~# ip a | grep inet
    inet <b>127.0.0.1/8</b> scope host <b>lo</b>
    inet6 ::1/128 scope host 
    inet <b>10.0.2.15/24</b> brd 10.0.2.255 scope global dynamic <b>enp0s3</b>
    inet6 fe80::fe:c0ff:fe34:c3ed/64 scope link 
    inet <b>10.0.10.1/30</b> brd 10.0.10.3 scope global <b>enp0s8</b>
    inet6 fe80::a00:27ff:feda:4781/64 scope link 
    inet <b>10.0.12.1/30</b> brd 10.0.12.3 scope global <b>enp0s9</b>
    inet6 fe80::a00:27ff:feb0:6c79/64 scope link 
    inet <b>192.168.10.1/24</b> brd 192.168.10.255 scope global <b>enp0s10</b>
    inet6 fe80::a00:27ff:feec:315f/64 scope link 
    inet <b>192.168.50.10/24</b> brd 192.168.50.255 scope global <b>enp0s16</b>
    inet6 fe80::a00:27ff:fe22:dded/64 scope link 
root@router1:~#</pre>

<p>● Зайти в интерфейс FRR и посмотреть информацию об интерфейсах</p>

<pre>root@router1:~# vtysh

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show interface brief
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
<b>enp0s3</b>          up      default         <b>10.0.2.15/24</b>
<b>enp0s8</b>          up      default         <b>10.0.10.1/30</b>
<b>enp0s9</b>          up      default         <b>10.0.12.1/30</b>
<b>enp0s10</b>         up      default         <b>192.168.10.1/24</b>
<b>enp0s16</b>         up      default         <b>192.168.50.10/24</b>
<b>lo</b>              up      default

router1# exit
root@router1:~#</pre>

<p>В обоих примерах мы увидем имена сетевых интерфейсов, их ip-адреса и маски подсети. Исходя из схемы мы понимаем, что для настройки OSPF нам достаточно описать интерфейсы enp0s8, enp0s9, enp0s10.</p>

<p>Откроем файл <b>/etc/frr/frr.conf</b> и вносим в него следующую информацию:</p>

<pre>root@router1:~# nano /etc/frr/frr.conf</pre>

<pre>!Указание версии FRR
frr version 8.1
frr defaults traditional
!Указываем имя машины
hostname router1
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе enp0s8
interface enp0s8
!Указываем имя интерфейса
description r1-r2
!Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
ip address 10.0.10.1/30
!Указываем параметр игнорирования MTU
ip ospf mtu-ignore
!Если потребуется, можно указать «стоимость» интерфейса
!ip ospf cost 1000
!Указываем параметры hello-интервала для OSPF пакетов
ip ospf hello-interval 10
!Указываем параметры dead-интервала для OSPF пакетов
!Должно быть кратно предыдущему значению
ip ospf dead-interval 30
!
interface enp0s9
description r1-r3
ip address 10.0.12.1/30
ip ospf mtu-ignore
!ip ospf cost 45
ip ospf hello-interval 10
ip ospf dead-interval 30

interface enp0s10
description net_router1
ip address 192.168.10.1/24
ip ospf mtu-ignore
!ip ospf cost 45
ip ospf hello-interval 10
ip ospf dead-interval 30
!
!Начало настройки OSPF
router ospf
!Указываем router-id
router-id 1.1.1.1
!Указываем сети, которые хотим анонсировать соседним роутерам
network 10.0.10.0/30 area 0
network 10.0.12.0/30 area 0
network 192.168.10.0/24 area 0
!Указываем адреса соседних роутеров
neighbor 10.0.10.2
neighbor 10.0.12.2

!Указываем адрес log-файла
log file /var/log/frr/frr.log
default-information originate always</pre>

<p>Сохраняем изменения и выходим из данного файла.</p>

<p>Вместо файла frr.conf мы можем задать данные параметры вручную из vtysh. Vtysh использует cisco-like команды.</p>

<p>На хостах router2 и router3 также потребуется аналогично настроить конфигруационные файлы, предварительно поменяв ip -адреса интерфейсов.</p>

<p>В ходе создания файла мы видим несколько OSPF-параметров, которые требуются для настройки:<br />
<b>● hello-interval</b> — интервал который указывает через сколько секунд протокол OSPF будет повторно отправлять запросы на другие роутеры. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF. <br />
<b>● Dead-interval</b> — если в течении заданного времени роутер не отвечает на запросы, то он считается вышедшим из строя и пакеты уходят на другой роутер (если это возможно). Значение должно быть кратно hello-интервалу. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF.<br />
<b>● router-id</b> — идентификатор маршрутизатора (необязательный параметр), если данный параметр задан, то роутеры определяют свои роли по данному параметру. Если данный идентификатор не задан, то роли маршрутизаторов 13определяются с помощью Loopback-интерфейса или самого большого ip-адреса на роутере.</p>

<p>8) После создания файлов <b>/etc/frr/frr.conf</b> и <b>/etc/frr/daemons</b> проверим, что владельцем файла является пользователь frr. Группа файла также должна быть frr. Должны быть установленны следующие права:<br />
● у владельца на чтение и запись<br />
● у группы только на чтение</p>

<pre>root@router1:~# ls -l /etc/frr
total 20
-rw-r----- 1 frr frr 2619 Aug 31 08:58 daemons
-rw-r----- 1 frr frr 2352 Sep 16 20:34 frr.conf
-rw-r----- 1 frr frr 5454 Aug 31 08:58 support_bundle_commands.conf
-rw-r----- 1 frr frr   32 Aug 31 08:58 vtysh.conf
root@router1:~#</pre>

<p>Если права или владелец файла указан неправильно, то нужно поменять владельца и назначить правильные права, например:</p>

<pre>chown frr:frr /etc/frr/frr.conf
chmod 640 /etc/frr/frr.conf</pre>

<p>9) Перезапускаем FRR и добавляем его в автозагрузку:</p>

<pre>root@router1:~# systemctl restart frr
root@router1:~# systemctl enable frr
Synchronizing state of frr.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable frr
root@router1:~#</pre>

<p>10) Проверям, что OSPF перезапустился без ошибок:</p>

<pre>root@router1:~# systemctl status frr
● frr.service - FRRouting
     Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset: e>
     <b>Active: active (running)</b> since Fri 2022-09-16 20:46:46 UTC; 1min 3s ago
       Docs: https://frrouting.readthedocs.io/en/latest/setup.html
   Main PID: 19951 (watchfrr)
     Status: "FRR Operational"
      Tasks: 7 (limit: 1131)
     Memory: 9.6M
     CGroup: /system.slice/frr.service
             ├─19951 /usr/lib/frr/watchfrr -d -F traditional zebra staticd
             ├─19962 /usr/lib/frr/zebra -d -F traditional -A 127.0.0.1 -s 90000>
             └─19967 /usr/lib/frr/staticd -d -F traditional -A 127.0.0.1

Sep 16 20:46:41 router1 watchfrr[19951]: [ZCJ3S-SPH5S] staticd state -> down : >
Sep 16 20:46:41 router1 watchfrr[19951]: [YFT0P-5Q5YX] Forked background comman>
Sep 16 20:46:41 router1 zebra[19962]: [VTVCM-Y2NW3] Configuration Read in Took:>
Sep 16 20:46:41 router1 staticd[19967]: [VTVCM-Y2NW3] Configuration Read in Too>
Sep 16 20:46:41 router1 watchfrr[19951]: [ZJW5C-1EHNT] restart all process 1995>
Sep 16 20:46:46 router1 watchfrr[19951]: [QDG3Y-BY5TN] zebra state -> up : conn>
Sep 16 20:46:46 router1 watchfrr[19951]: [QDG3Y-BY5TN] staticd state -> up : co>
Sep 16 20:46:46 router1 watchfrr[19951]: [KWE5Q-QNGFC] all daemons up, doing st>
Sep 16 20:46:46 router1 frrinit.sh[19931]:  * Started watchfrr
Sep 16 20:46:46 router1 systemd[1]: Started FRRouting.
root@router1:~#</pre>

<p>Если мы правильно настроили OSPF, то с любого хоста нам должны быть доступны сети:<br />
● 192.168.10.0/24<br />
● 192.168.20.0/24<br />
● 192.168.30.0/24<br />
● 10.0.10.0/30<br />
● 10.0.11.0/30<br />
● 10.0.13.0/30</p>

<p>Проверим доступность сетей с хоста router1:<br />
● попробуем сделать ping до ip-адреса 192.168.20.1:</p>

<pre>root@router1:~# <b>ping -c 3 192.168.20.1</b>
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=1.78 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=1.54 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=1.54 ms

--- 192.168.20.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 1.537/1.618/1.776/0.111 ms
root@router1:~#</pre>

<p>● Запустим трассировку до адреса 192.168.20.1:</p>

<pre>root@router1:~# <b>traceroute 192.168.20.1</b>
traceroute to 192.168.20.1 (192.168.20.1), 30 hops max, 60 byte packets
 1  192.168.20.1 (192.168.20.1)  1.892 ms  3.167 ms  2.822 ms
root@router1:~#</pre>

<p>Попробуем отключить интерфейс enp0s8:</p>

<pre>root@router1:~# <b>ifconfig enp0s8 down</b>
root@router1:~#</pre>

<p>Убедимся, интерфейс enp0s8 отключен:</p>

<pre>root@router1:~# <b>ip a | grep enp0s8</b>
3: <b>enp0s8</b>: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state <b>DOWN</b> group default qlen 1000
root@router1:~# </pre>

<p>Немного подождем и снова запустим трассировку до ip-адреса 192.168.20.1:</p>

<pre>root@router1:~# <b>traceroute 192.168.20.1</b>
traceroute to 192.168.20.1 (192.168.20.1), 30 hops max, 60 byte packets
 1  10.0.12.2 (10.0.12.2)  0.463 ms  0.412 ms  0.820 ms
 2  192.168.20.1 (192.168.20.1)  2.319 ms  2.768 ms  2.745 ms
root@router1:~#</pre>

<p>Как мы видим, после отключения интерфейса enp0s8 сеть 192.168.20.0/24 нам остаётся доступна, но через router3.</p>

<p>Также мы можем проверить из интерфейса vtysh, какие маршруты мы видим на данный момент:</p>

<pre>root@router1:~# <b>vtysh</b>

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# <b>show ip route ospf</b>
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O>* 10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:09:05
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:09:26
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:36:25
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:36:25
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:09:26
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:14:20
router1# exit
root@router1:~#</pre>

<p>Снова включим интерфейс enp0s8:</p>

<pre>root@router1:~# ifconfig enp0s8 up
root@router1:~# ip a | grep enp0s8
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 10.0.10.1/30 brd 10.0.10.3 scope global enp0s8
root@router1:~#</pre>

<p>Снова проверим из интерфейса vtysh:</p>

<pre>root@router1:~# <b>vtysh</b>

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# <b>show ip route ospf</b>
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:02:32
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:02:32
  *                        via 10.0.12.2, enp0s9, weight 1, 00:02:32
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:49:22
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:49:22
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:02:32
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:27:17
router1# exit
root@router1:~#</pre>

<h4>2.2 Настройка ассиметричного роутинга</h4>

<p>Для настройки ассиметричного роутинга нам необходимо выключить блокировку ассиметричной маршрутизации:</p>

<pre>root@router1:~# <b>sysctl net.ipv4.conf.all.rp_filter=0</b>
net.ipv4.conf.all.rp_filter = 0
root@router1:~#</pre>

<p>Далее, выбираем один из роутеров, на котором изменим «стоимость интерфейса». Например поменяем стоимость интерфейса enp0s9 на router1:</p>

<pre>root@router1:~# <b>vtysh</b>

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# <b>conf t</b>
router1(config)# <b>int enp0s9</b>
router1(config-if)# <b>ip ospf cost 1000</b>
router1(config-if)# <b>exit</b>
router1(config)# <b>exit</b>
router1# <b>show ip route ospf</b>
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:06:31
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:00:32
O   10.0.12.0/30 [110/300] via 10.0.10.2, enp0s8, weight 1, 00:00:32
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:53:21
O>* 192.168.20.0/24 [110/200] via <b>10.0.10.2</b>, enp0s8, weight 1, 00:06:31
O>* 192.168.30.0/24 [110/300] via <b>10.0.10.2</b>, enp0s8, weight 1, 00:00:32
router1# exit
root@router1:~#</pre>

<p>Проверим из интерфейса vtysh на router3:</p>

<pre>root@router3:~# vtysh

Hello, this is FRRouting (version 7.2.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

<b>router3</b># <b>show ip route ospf</b>
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O>* 10.0.10.0/30 [110/200] via 10.0.11.2, enp0s8, 00:12:14
  *                        via 10.0.12.1, enp0s9, 00:12:14
O   10.0.11.0/30 [110/100] is directly connected, enp0s8, 00:37:04
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, 00:36:58
O>* 192.168.10.0/24 [110/200] via <b>10.0.12.1</b>, enp0s9, 00:36:58
O>* 192.168.20.0/24 [110/200] via 10.0.11.2, enp0s8, 00:36:58
O   192.168.30.0/24 [110/100] is directly connected, enp0s10, 00:37:10
<b>router3</b># exit
root@router3:~#</pre>

<p>После внесения данных настроек, мы видим, что маршрут до сети 192.168.30.0/30 теперь пойдёт через router3, но обратный трафик от router3 пойдёт по другому пути. Давайте это проверим:<br />
1) На <b>router1</b> запускаем пинг от 192.168.10.1 до 192.168.30.1:</p>

<pre>root@router1:~# ping -I 192.168.10.1 192.168.30.1
PING 192.168.30.1 (192.168.30.1) from 192.168.10.1 : 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=2.33 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=2.19 ms
64 bytes from 192.168.30.1: icmp_seq=3 ttl=64 time=2.11 ms
...</pre>

<p>2) На <b>router3</b> запускаем tcpdump, который будет смотреть трафик только на порту <b>enp0s8</b>:</p>

<pre>tcpdump -i enp0s8</pre>

<pre>root@router3:~# <b>tcpdump -i enp0s8</b>
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
08:21:14.730685 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 25, length 64
08:21:15.735708 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 26, length 64
08:21:16.737300 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 27, length 64
08:21:17.739210 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 28, length 64
08:21:18.741044 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 29, length 64
08:21:18.855208 IP router3 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:21:19.742838 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 30, length 64
08:21:20.744596 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 31, length 64
08:21:21.745871 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 32, length 64
08:21:22.747443 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 33, length 64
08:21:23.547471 IP 10.0.11.2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:21:23.749305 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 34, length 64
08:21:24.756155 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 7, seq 35, length 64
^C
13 packets captured
13 packets received by filter
0 packets dropped by kernel
root@router3:~#</pre>

<p>Видим что данный порт только получает ICMP-трафик с адреса 192.168.10.1</p>

<p>3) На router3 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:</p>

<pre>root@router3:~# <b>tcpdump -i enp0s9</b>
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
08:22:10.852124 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 80, length 64
08:22:11.854234 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 81, length 64
08:22:12.855993 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 82, length 64
08:22:12.919062 ARP, Request who-has 10.0.12.1 tell router3, length 28
08:22:12.920410 ARP, Reply 10.0.12.1 is-at 08:00:27:b0:6c:79 (oui Unknown), length 46
08:22:13.857465 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 83, length 64
08:22:14.867115 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 84, length 64
08:22:15.877407 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 85, length 64
08:22:15.944286 IP 10.0.12.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:22:16.888707 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 86, length 64
08:22:17.910582 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 87, length 64
08:22:18.860007 IP <b>router3 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:22:18.922930 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 88, length 64
08:22:19.933741 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 7, seq 89, length 64
^C
14 packets captured
14 packets received by filter
0 packets dropped by kernel
root@router3:~#</pre>

<p>Видим что данный порт только отправляет ICMP-трафик на адрес 192.168.10.1.</p>

<p>Таким образом мы наблюдаем ассиметричный роутинг.</p>

<h4>2.3 Настройка симметичного роутинга</h4>

<p>Так как у нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация.</p>

<p>Так как в прошлом задании мы заметили что router3 будет отправлять обратно трафик через порт enp0s9, мы также должны сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:</p>

<p>Поменяем стоимость интерфейса enp0s9 на router3:</p>

<pre>root@router3:~# <b>vtysh</b>

Hello, this is FRRouting (version 7.2.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router3# <b>conf t</b>
router3(config)# <b>int enp0s9</b>
router3(config-if)# <b>ip ospf cost 1000</b>
router3(config-if)# <b>exit</b>
router3(config)# <b>exit</b>
router3# <b>show ip route ospf</b>
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O>* 10.0.10.0/30 [110/200] via 10.0.11.2, enp0s8, 00:00:23
O   10.0.11.0/30 [110/100] is directly connected, enp0s8, 01:05:35
O   10.0.12.0/30 [110/1000] is directly connected, enp0s9, 00:00:23
O>* 192.168.10.0/24 [110/300] via <b>10.0.11.2</b>, enp0s8, 00:00:23
O>* 192.168.20.0/24 [110/200] via 10.0.11.2, enp0s8, 01:05:29
O   192.168.30.0/24 [110/100] is directly connected, enp0s10, 01:05:41
router3# exit
root@router3:~#</pre>

<p>После внесения данных настроек, мы видим, что маршрут до сети 192.168.10.0/30 пойдёт через <b>router2</b>.</p>

<p>Давайте это проверим:<br />
1) На <b>router1</b> запускаем пинг от 192.168.10.1 до 192.168.20.1:</p>

<pre>PING 192.168.30.1 (192.168.30.1) from 192.168.10.1 : 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=63 time=2.80 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=63 time=2.84 ms
64 bytes from 192.168.30.1: icmp_seq=3 ttl=63 time=2.50 ms
...</pre>

<p>2) На <b>router3</b> запускаем tcpdump, который будет смотреть трафик только на порту <b>enp0s8</b>:</p>

<pre>root@router3:~# <b>tcpdump -i enp0s8</b>
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
08:49:39.002470 IP router3 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:49:39.855295 IP <b>192.168.10.1 > router3</b>: ICMP echo request, id 8, seq 57, length 64
08:49:39.855387 IP <b>router3 > 192.168.10.1</b>: ICMP echo reply, id 8, seq 57, length 64
08:49:40.856865 IP 192.168.10.1 > router3: ICMP echo request, id 8, seq 58, length 64
08:49:40.856963 IP router3 > 192.168.10.1: ICMP echo reply, id 8, seq 58, length 64
08:49:41.858291 IP 192.168.10.1 > router3: ICMP echo request, id 8, seq 59, length 64
08:49:41.858418 IP router3 > 192.168.10.1: ICMP echo reply, id 8, seq 59, length 64
08:49:42.142954 ARP, Request who-has 10.0.11.2 tell router3, length 28
08:49:42.144265 ARP, Reply 10.0.11.2 is-at 08:00:27:14:ca:cf (oui Unknown), length 46
08:49:42.859962 IP 192.168.10.1 > router3: ICMP echo request, id 8, seq 60, length 64
08:49:42.860074 IP router3 > 192.168.10.1: ICMP echo reply, id 8, seq 60, length 64
08:49:43.861394 IP 192.168.10.1 > router3: ICMP echo request, id 8, seq 61, length 64
08:49:43.861516 IP router3 > 192.168.10.1: ICMP echo reply, id 8, seq 61, length 64
08:49:44.417120 IP 10.0.11.2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:49:44.862960 IP 192.168.10.1 > router3: ICMP echo request, id 8, seq 62, length 64
08:49:44.863058 IP router3 > 192.168.10.1: ICMP echo reply, id 8, seq 62, length 64
^C
16 packets captured
16 packets received by filter
0 packets dropped by kernel
root@router3:~#</pre>

<p>Теперь мы наблюдаем, что трафик между роутерами router1 и router3 ходит симметрично через router2.</p>














<h4>Первоначальная настройка Ansible</h4>

<p>Для настроки хостов с помощью Ansible создадим структуру каталогов файлов. <br />
создаём каталог ansible, затем переходим в этот каталог:</p>

<pre>[user@localhost ospf]$ mkdir ./ansible && cd ./ansible/
[user@localhost ansible]$</pre>

<p>В этом каталоге создадим необходимые конфиг файлы:<br />
● Конфигурационный файл ansible.cfg, который описывает базовые настройки для работы Ansible:

<pre>[defaults]
#Отключение проверки ключа хоста
host_key_checking = false
#Указываем имя файла инвентаризации
inventory = hosts
#Отключаем игнорирование предупреждений
command_warnings= false</pre>

<p>● Файл инвентаризации hosts, который хранит информацию о том, как подключиться к хосту:</p>

<pre>[routers]
router1 ansible_host=192.168.50.10 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router1/virtualbox/private_key
router2 ansible_host=192.168.50.11 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router2/virtualbox/private_key
router3 ansible_host=192.168.50.12 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router3/virtualbox/private_key</pre>

<p>где:
- [servers] - в квадратных скобках указана группа хостов<br />
- router1 — имя нашего хоста (имена хостов и групп не могут быть одинаковые)<br />
- ansible_host — адрес нашего хоста<br />
- ansible_user — имя пользователя, с помощью которого Ansible будет подключаться к хосту<br />
- ansible_ssh_private_key — адрес расположения ssh-ключа<br />
В файл инвентаризации также можно добавлять переменные, которые могут автоматически добавляться в jinja template. Добавление переменных будет рассмотрено далее.</p>

<p>● Ansible-playbook playbook.yml — основной файл, в котором содержатся инструкции (модули) по настройке для Ansible или включает другие playbook файлы или роли, как в нашем случае:</p>

<pre>---
- name: OSPF | Install and Configure
  hosts: all
  become: true

  roles:
    - { role: ospf, when: ansible_system == 'Linux' }</pre>

Затем создаём каталог roles:</p>

<pre>[user@localhost ansible]$ mkdir ./roles
[user@localhost ansible]$</pre>

<p>Перейдём в этот каталог и с помощью команды ansible-galaxy init создадим структуру каталогов:</p>

<pre>[user@localhost ansible]$ cd ./roles/
[user@localhost roles]$ ansible-galaxy init ospf
- Role ospf was created successfully
[user@localhost roles]$</pre>

<h4>Установка пакетов для тестирования и настройки OSPF</h4>

<p>Перед настройкой FRR рекомендуется поставить базовые программы для изменения конфигурационных файлов (vim) и изучения сети (traceroute, tcpdump, net-tools):</p>

<pre>apt -y update
apt -y install vim traceroute tcpdump net-tools</pre>

<p>Установка пакетов с помощью Ansible</p>

<pre>#Начало файла provision.yml
- name: OSPF
  #Указываем имя хоста или группу, которые будем настраивать
  hosts: all
  #Параметр выполнения модулей от root-пользователя
  become: yes
  #Указание файла с дополнителыми переменными (понадобится при добавлении темплейтов)
  vars_files:
  - defaults/main.yml
  tasks:
  # Обновление пакетов и установка vim, traceroute, tcpdump, net-tools
  - name: install base tools
    apt:
      name:
      - vim
      - traceroute
      - tcpdump
      - net-tools
      state: present
      update_cache: true</pre>






<h4>2.1 Настройка OSPF между машинами на базе Quagga</h4>

<p>Пакет Quagga перестал развиваться в 2018 году. Ему на смену пришёл пакет FRR, он построен на базе Quagga и продолжает своё развитие. В данном руководстве настойка OSPF будет осуществляться в FRR.<p>

<p>Процесс установки FRR и настройки OSPF вручную:<br />
1) Отключаем файерволл ufw и удаляем его из автозагрузки:</p>

<pre>systemctl stop ufw
systemctl disable ufw</pre>

<p>2) Добавляем gpg ключ:</p>

<pre>curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -</pre>

<p>3) Добавляем репозиторий c пакетом FRR:</p>

<pre>echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable > /etc/apt/sources.list.d/frr.list</pre>

<p>4) Обновляем пакеты и устанавливаем FRR:</p>

<pre>sudo apt update
sudo apt install frr frr-pythontools</pre>

<p>5) Разрешаем (включаем) маршрутизацию транзитных пакетов:</p>

<pre>sysctl net.ipv4.conf.all.forwarding=1</pre>

<p>6) Включаем демон ospfd в FRR<br />
Для этого открываем в редакторе файл /etc/frr/daemons и меняем в нём параметры для пакетов zebra и ospfd на yes:</p>

<pre>vim /etc/frr/daemons
zebra=yes
ospfd=yes
bgpd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no</pre>

<p>В примере показана только часть файла</p>

<p>7) Настройка OSPF<br />
Для настройки OSPF нам потребуется создать файл /etc/frr/frr.conf, который будет содержать в себе информацию о требуемых интерфейсах и OSPF. Разберем пример создания файла на хосте router1.</p>

<p>Для начала нам необходимо узнать имена интерфейсов и их адреса. Сделать это можно с помощью двух способов:<br />
● Посмотреть в linux: ip a | grep inet</p>

<pre>root@router1:~# ip a | grep "inet "
    inet 127.0.0.1/8 scope host lo
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
    inet 10.0.10.1/30 brd 10.0.10.3 scope global enp0s8
    inet 10.0.12.1/30 brd 10.0.12.3 scope global enp0s9
    inet 192.168.10.1/24 brd 192.168.10.255 scope global enp0s10
    inet 192.168.50.10/24 brd 192.168.50.255 scope global enp0s16
root@router1:~#</pre>

<p>● Зайти в интерфейс FRR и посмотреть информацию об интерфейсах</p>

<pre>root@router1:~# vtysh

Hello, this is FRRouting (version 8.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show interface brief
Interface 	Status 	VRF 		Addresses
--------- 	------ 	--- 		---------
enp0s3 		up 		default 	10.0.2.15/24
enp0s8 		up 		default 	10.0.10.1/30
enp0s9 		up 		default 	10.0.12.1/30
enp0s10 	up 		default 	192.168.10.1/24
enp0s16 	up 		default 	192.168.50.10/24
lo 			up 		default

router1# exit
root@router1:~#</pre>

<p>В обоих примерах мы увидем имена сетевых интерфейсов, их ip-адреса и маски подсети. Исходя из схемы мы понимаем, что для настройки OSPF нам достаточно описать интерфейсы enp0s8, enp0s9, enp0s10</p>

<p>Создаём файл /etc/frr/frr.conf и вносим в него следующую информацию:</p>

<pre>!Указание версии FRR
frr version 8.1
frr defaults traditional
!Указываем имя машины
hostname router1
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе enp0s8
interface enp0s8
!Указываем имя интерфейса
description r1-r2
!Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
ip address 10.0.10.1/30
!Указываем параметр игнорирования MTU
ip ospf mtu-ignore
!Если потребуется, можно указать «стоимость» интерфейса
!ip ospf cost 1000
!Указываем параметры hello-интервала для OSPF пакетов
ip ospf hello-interval 10
!Указываем параметры dead-интервала для OSPF пакетов
!Должно быть кратно предыдущему значению
ip ospf dead-interval 30
!
interface enp0s9
description r1-r3
ip address 10.0.12.1/30
ip ospf mtu-ignore
!ip ospf cost 45
ip ospf hello-interval 10
ip ospf dead-interval 30

interface enp0s10
description net_router1
ip address 192.168.10.1/24
ip ospf mtu-ignore
!ip ospf cost 45
ip ospf hello-interval 10
ip ospf dead-interval 30
!
!Начало настройки OSPF
router ospf
!Указываем router-id
router-id 1.1.1.1
!Указываем сети, которые хотим анонсировать соседним роутерам
network 10.0.10.0/30 area 0
network 10.0.12.0/30 area 0
network 192.168.10.0/24 area 0
!Указываем адреса соседних роутеров
neighbor 10.0.10.2
neighbor 10.0.12.2

!Указываем адрес log-файла
log file /var/log/frr/frr.log
default-information originate always</pre>

<p>Сохраняем изменения и выходим из данного файла.</p>

Вместо файла frr.conf мы можем задать данные параметры вручную из vtysh.
Vtysh использует cisco-like команды.
На хостах router2 и router3 также потребуется настроить конфигруационные
файлы, предварительно поменяв ip -адреса интерфейсов.
В ходе создания файла мы видим несколько OSPF-параметров, которые
требуются для настройки:
● hello-interval — интервал который указывает через сколько секунд
протокол OSPF будет повторно отправлять запросы на другие роутеры.
Данный интервал должен быть одинаковый на всех портах и роутерах, между
которыми настроен OSPF.
● Dead-interval — если в течении заданного времени роутер не отвечает на
запросы, то он считается вышедшим из строя и пакеты уходят на другой
роутер (если это возможно). Значение должно быть кратно hello-интервалу.
Данный интервал должен быть одинаковый на всех портах и роутерах, между
которыми настроен OSPF.
● router-id — идентификатор маршрутизатора (необязательный параметр), если
данный параметр задан, то роутеры определяют свои роли по данному
параметру. Если данный идентификатор не задан, то роли маршрутизаторов
13определяются с помощью Loopback-интерфейса или самого большого ip-адреса
на роутере.
8) После создания файлов /etc/frr/frr.conf и /etc/frr/daemons нужно
проверить, что владельцем файла является пользователь frr. Группа файла
также должна быть frr. Должны быть установленны следующие права:
● у владельца на чтение и запись
● у группы только на чтение
ls -l /etc/frr
Если права или владелец файла указан неправильно, то нужно поменять
владельца и назначить правильные права, например:
chown frr:frr /etc/frr/frr.conf
chmod 640 /etc/frr/frr.conf
9) Перезапускаем FRR и добавляем его в автозагрузку
systemct restart frr
systemctl enable frr
10) Проверям, что OSPF перезапустился без ошибок
root@router1:~# systemctl status frr
● frr.service - FRRouting
Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset:
enabled)
Active: active (running) since Wed 2022-02-23 15:24:04 UTC; 2h 1min ago
Docs: https://frrouting.readthedocs.io/en/latest/setup.html
Process: 31988 ExecStart=/usr/lib/frr/frrinit.sh start (code=exited,
status=0/SUCCESS)
Main PID: 32000 (watchfrr)
Status: "FRR Operational"
Tasks: 9 (limit: 1136)
Memory: 13.2M
CGroup: /system.slice/frr.service
├─32000 /usr/lib/frr/watchfrr -d -F traditional zebra ospfd
staticd
├─32016 /usr/lib/frr/zebra -d -F traditional -A 127.0.0.1 -s
90000000
├─32021 /usr/lib/frr/ospfd -d -F traditional -A 127.0.0.1
└─32024 /usr/lib/frr/staticd -d -F traditional -A 127.0.0.1
Feb 23 15:23:59 router1 zebra[32016]: [VTVCM-Y2NW3] Configuration Read in
Took: 00:00:00
Feb 23 15:23:59 router1 ospfd[32021]: [VTVCM-Y2NW3] Configuration Read in
Took: 00:00:00
Feb 23 15:23:59 router1 staticd[32024]: [VTVCM-Y2NW3] Configuration Read in
Took: 00:00:00
Feb 23 15:24:04 router1 watchfrr[32000]: [QDG3Y-BY5TN] staticd state -> up :
connect succeeded
Feb 23 15:24:04 router1 watchfrr[32000]: [QDG3Y-BY5TN] zebra state -> up :
connect succeeded
Feb 23 15:24:04 router1 watchfrr[32000]: [QDG3Y-BY5TN] ospfd state -> up :
connect succeeded
Feb 23 15:24:04 router1 watchfrr[32000]: [KWE5Q-QNGFC] all daemons up, doing
startup-complete notify
Feb 23 15:24:04 router1 frrinit.sh[31988]: * Started watchfrr
14Feb 23 15:24:04 router1 systemd[1]: Started FRRouting.
root@router1:~#
Если мы правильно настроили OSPF, то с любого хоста нам должны быть
доступны сети:
● 192.168.10.0/24
● 192.168.20.0/24
● 192.168.30.0/24
● 10.0.10.0/30
● 10.0.11.0/30
● 10.0.13.0/30
Проверим доступность сетей с хоста router1:
● попробуем сделать ping до ip-адреса 192.168.30.1
root@router1:~# ping 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=1.32 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=1.15 ms
^C
--- 192.168.30.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 1.146/1.234/1.322/0.088 ms
root@router1:~#
● Запустим трассировку до адреса 192.168.30.1
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
1 192.168.30.1 (192.168.30.1) 0.997 ms 0.868 ms 0.813 ms
root@router1:~#
Попробуем отключить интерфейс enp0s9 и немного подождем и снова запустим
трассировку до ip-адреса 192.168.30.1
root@router1:~# ifconfig enp0s9 down
root@router1:~# ip a | grep enp0s9
4: enp0s9: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state DOWN group
default qlen 1000
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
1 10.0.10.2 (10.0.10.2) 0.522 ms 0.479 ms 0.460 ms
2 192.168.30.1 (192.168.30.1) 0.796 ms 0.777 ms 0.644 ms
root@router1:~#
Как мы видим, после отключения интерфейса сеть 192.168.30.0/24 нам
остаётся доступна.
● Также мы можем проверить из интерфейса vtysh какие маршруты мы видим на
данный момент:
root@router1:~#
root@router1:~# vtysh
Hello, this is FRRouting (version 8.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.
router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
15O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
f - OpenFabric,
> - selected route, * - FIB route, q - queued, r - rejected, b -
backup
t - trapped, o - offload failure
O 10.0.10.0/30 [110/1000] is directly connected, enp0s8, weight 1,
02:50:21
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:01:00
O 10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1,
00:01:00
O 192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1,
02:50:21
O>* 192.168.20.0/24 [110/300] via 10.0.10.2, enp0s9, weight 1, 00:01:00
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:01:00
router1

