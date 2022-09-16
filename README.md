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

<pre>root@router1:~# apt -y update
...
root@router1:~# apt -y install vim traceroute tcpdump net-tools
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

<pre>root@router1:~# apt -y update
...
root@router1:~# apt -y install frr frr-pythontools
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
ospfd=yes &lt;-------
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
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
    inet6 fe80::fe:c0ff:fe34:c3ed/64 scope link 
    inet 10.0.10.1/30 brd 10.0.10.3 scope global enp0s8
    inet6 fe80::a00:27ff:feda:4781/64 scope link 
    inet 10.0.12.1/30 brd 10.0.12.3 scope global enp0s9
    inet6 fe80::a00:27ff:feb0:6c79/64 scope link 
    inet 192.168.10.1/24 brd 192.168.10.255 scope global enp0s10
    inet6 fe80::a00:27ff:feec:315f/64 scope link 
    inet 192.168.50.10/24 brd 192.168.50.255 scope global enp0s16
    inet6 fe80::a00:27ff:fe22:dded/64 scope link 
root@router1:~#</pre>

<p>● Зайти в интерфейс FRR и посмотреть информацию об интерфейсах</p>

<pre>root@router1:~# vtysh

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show interface brief
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
enp0s3          up      default         10.0.2.15/24
enp0s8          up      default         10.0.10.1/30
enp0s9          up      default         10.0.12.1/30
enp0s10         up      default         192.168.10.1/24
enp0s16         up      default         192.168.50.10/24
lo              up      default

router1# exit
root@router1:~#</pre>

<p>В обоих примерах мы увидем имена сетевых интерфейсов, их ip-адреса и маски подсети. Исходя из схемы мы понимаем, что для настройки OSPF нам достаточно описать интерфейсы enp0s8, enp0s9, enp0s10.</p>

<p>Откроем файл /etc/frr/frr.conf и вносим в него следующую информацию:</p>

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

<p>На хостах router2 и router3 также потребуется настроить конфигруационные файлы, предварительно поменяв ip -адреса интерфейсов.</p>

<p>В ходе создания файла мы видим несколько OSPF-параметров, которые требуются для настройки:<br />
<b>● hello-interval</b> — интервал который указывает через сколько секунд протокол OSPF будет повторно отправлять запросы на другие роутеры. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF. <br />
<b>● Dead-interval</b> — если в течении заданного времени роутер не отвечает на запросы, то он считается вышедшим из строя и пакеты уходят на другой роутер (если это возможно). Значение должно быть кратно hello-интервалу. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF.<br />
<b>● router-id</b> — идентификатор маршрутизатора (необязательный параметр), если данный параметр задан, то роутеры определяют свои роли по данному параметру. Если данный идентификатор не задан, то роли маршрутизаторов 13определяются с помощью Loopback-интерфейса или самого большого ip-адреса на роутере.</p>

<p>8) После создания файлов /etc/frr/frr.conf и /etc/frr/daemons проверим, что владельцем файла является пользователь frr. Группа файла также должна быть frr. Должны быть установленны следующие права:<br />
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
     Active: active (running) since Fri 2022-09-16 20:46:46 UTC; 1min 3s ago
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
● попробуем сделать ping до ip-адреса 192.168.30.1</p>

<pre>ping -c 3 192.168.30.1
...
</pre>

<p>● Запустим трассировку до адреса 192.168.30.1</p>

<pre>traceroute 192.168.30.1
...
</pre>

<p>Попробуем отключить интерфейс enp0s9 и немного подождем и снова запустим трассировку до ip-адреса 192.168.30.1</p>

<pre>ifconfig enp0s9 down
ip a | grep enp0s9
traceroute 192.168.30.1</pre>

<p>Как мы видим, после отключения интерфейса сеть 192.168.30.0/24 нам остаётся доступна.</p>

<p>Также мы можем проверить из интерфейса vtysh, какие маршруты мы видим на данный момент:</p>

<pre>root@router1:~# vtysh
Hello, this is FRRouting (version 8.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.
router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
15O
T
f
>
-
-
-
-
OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
OpenFabric,
selected route, * - FIB route, q - queued, r - rejected, b -
backup
t - trapped, o - offload failure
O
10.0.10.0/30 [110/1000] is directly connected, enp0s8, weight 1,
02:50:21
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:01:00
O
10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1,
00:01:00
O
192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1,
02:50:21
O>* 192.168.20.0/24 [110/300] via 10.0.10.2, enp0s9, weight 1, 00:01:00
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:01:00
router1#
Настройка OSPF c</pre>

<h4>2.2 Настройка ассиметричного роутинга</h4>

<p>Для настройки ассиметричного роутинга нам необходимо выключить блокировку ассиметричной маршрутизации:</p>

<pre>sysctl net.ipv4.conf.all.rp_filter=0</pre>

<p>Далее, выбираем один из роутеров, на котором изменим «стоимость интерфейса». Например поменяем стоимость интерфейса enp0s8 на router1:</p>

<pre>vtysh
Hello, this is FRRouting (version 8.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# conf t
18router1(config)# int enp0s8
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
router1# show ip route ospf

Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure
O 10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:02:24
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:02:29
O 10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:02:29
O 192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1,
00:03:04
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:02:24
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:02:29
router1#

router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure
O 10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:00:09
O 10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:34:11
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:00:09
*
via 10.0.11.1, enp0s9, weight 1, 00:00:09
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:00:09
O
192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1,
00:34:11
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:33:36
router2#</pre>

<p>После внесения данных настроек, мы видим, что маршрут до сети 192.168.20.0/30 теперь пойдёт через router2, но обратный трафик от router2 пойдёт по другому пути. Давайте это проверим:<br />
1) На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1:</p>

<pre>ping -I 192.168.10.1 192.168.20.1</pre>

<p>2) На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:</p>

<pre>tcpdump -i enp0s9</pre>

<p>Видим что данный порт только отправляет ICMP-трафик на адрес 192.168.10.1</p>

<p>Таким образом мы видим ассиметричный роутинг.</p>

<h4>2.3 Настройка симметичного роутинга</h4>

<p>Так как у нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация.</p>

<p>Так как в прошлом задании мы заметили что router2 будет отправлять обратно трафик через порт enp0s8, мы также должны сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:</p>

<p>Поменяем стоимость интерфейса enp0s8 на router2:</p>

<pre>router2# conf t
router2(config)# int enp0s8
router2(config-if)# ip ospf cost 1000
router2(config-if)# exit
router2(config)# exit
router2#
router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
f - OpenFabric,
> - selected route, * - FIB route, q - queued, r - rejected, b - backup
t - trapped, o - offload failure
O 10.0.10.0/30 [110/1000] is directly connected, enp0s8, weight 1, 00:00:13
O 10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:29:53
O>* 10.0.12.0/30 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:00:13
O>* 192.168.10.0/24 [110/300] via 10.0.11.1, enp0s9, weight 1, 00:00:13
O 192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1,
00:29:53
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:29:18
21router2#
router2# exit
root@router2:~#</pre>

<p>После внесения данных настроек, мы видим, что маршрут до сети 192.168.10.0/30 пойдёт через router2.</p>

<p>Давайте это проверим:<br />
1) На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1:</p>

<pre>ping -I 192.168.10.1 192.168.20.1</pre>

<p>2) На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:</p>

<pre>root@router2:~# tcpdump -i enp0s9</pre>

<p>Теперь мы видим, что трафик между роутерами ходит симметрично.</p>














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

