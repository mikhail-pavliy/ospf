# Статическая и динамическая маршрутизация, OSPF 
OSPF
- Поднять три виртуалки
- Объединить их разными private network
1. Поднять OSPF между машинами на базе Quagga
2. Изобразить ассиметричный роутинг
3. Сделать один из линков "дорогим", но что бы при этом роутинг был симметричным


Настроен Vagrant c 3 виртуалками, сконфигурированы ansible плейбуки которые разворачивают на них програмные роутеры, ```router01```, ```router02```, ```router03```. На каждом роутере есть свои подсети, связку между ними оргранизуем с помощью ospf. Для этого соединяем роутеры сетями и настраиваем программный маршрутизатор ```quagga```.

```ruby
[vagrant@router01 ~]$ ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
10.10.10.0/24 dev eth1 proto kernel scope link src 10.10.10.1 metric 101
10.10.20.0/24 via 172.20.0.2 dev eth2 proto zebra metric 20
10.10.30.0/24 via 192.168.100.2 dev eth3 proto zebra metric 20
172.20.0.0/30 dev eth2 proto kernel scope link src 172.20.0.1 metric 102
192.168.0.0/30 proto zebra metric 20
        nexthop via 172.20.0.2 dev eth2 weight 1
        nexthop via 192.168.100.2 dev eth3 weight 1
192.168.100.0/30 dev eth3 proto kernel scope link src 192.168.100.1 metric 103
```
```ruby
[vagrant@router02 ~]$ ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
10.10.10.0/24 via 172.20.0.1 dev eth2 proto zebra metric 20
10.10.20.0/24 dev eth1 proto kernel scope link src 10.10.20.1 metric 101
10.10.30.0/24 via 192.168.0.2 dev eth3 proto zebra metric 20
172.20.0.0/30 dev eth2 proto kernel scope link src 172.20.0.2 metric 102
192.168.0.0/30 dev eth3 proto kernel scope link src 192.168.0.1 metric 103
192.168.100.0/30 proto zebra metric 20
        nexthop via 172.20.0.1 dev eth2 weight 1
        nexthop via 192.168.0.2 dev eth3 weight 1
```
```ruby
[vagrant@router03 ~]$ ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
10.10.10.0/24 via 192.168.100.1 dev eth2 proto zebra metric 20
10.10.20.0/24 via 192.168.0.1 dev eth3 proto zebra metric 20
10.10.30.0/24 dev eth1 proto kernel scope link src 10.10.30.1 metric 101
172.20.0.0/30 proto zebra metric 20
        nexthop via 192.168.100.1 dev eth2 weight 1
        nexthop via 192.168.0.1 dev eth3 weight 1
192.168.0.0/30 dev eth3 proto kernel scope link src 192.168.0.2 metric 103
192.168.100.0/30 dev eth2 proto kernel scope link src 192.168.100.2 metric 102
```
Проверим что роутинг симметричный:
```ruby
[vagrant@router01 ~]$ ip a | grep 10.10
inet 10.10.10.1/24 brd 10.10.10.255 scope global noprefixroute eth1
[vagrant@router01 ~]$ tracepath -n 10.10.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.10.20.1                                            0.819ms reached
 1:  10.10.20.1                                            0.313ms reached
     Resume: pmtu 1500 hops 1 back 1
 ```
 Поменять маршрут можно вручную, изменив ```cost``` в подсети 172.20.0.0/30 которая связывает ```router01``` и ```router02``` для интерфейса 172.20.0.1 роутера router01,  например на 1000, и для router02  на 100 получим ассиметричный роутинг:
 
 ```ruby
 [vagrant@router01 ~]$ vtysh
vtysh_connect(/var/run/quagga/zebra.vty): stat = Permission denied
[vagrant@router01 ~]$ sudo su
[root@router01 vagrant]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router01# configure  terminal
router01(config)# interface eth2
router01(config-if)# ip ospf  cost  1000
router01(config-if)# exit
router01(config)# exit
```
```ruby
[root@router02 vagrant]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router02# configure  terminal
router02(config)# interface eth2
router02(config-if)# ip ospf  cost  100
router02(config-if)# exit
router02(config)# exit
router02# sh  ip ospf  interface  eth2
eth2 is up
  ifindex 4, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 172.20.0.2/30, Broadcast 172.20.0.3, Area 0.0.0.0
  MTU mismatch detection:disabled
  Router ID 0.0.0.2, Network Type POINTOPOINT, Cost: 100
  Transmit Delay is 1 sec, State Point-To-Point, Priority 1
  No designated router on this network
  No backup designated router on this network
  Multicast group memberships: OSPFAllRouters
  Timer intervals configured, Hello 5s, Dead 10s, Wait 10s, Retransmit 5
    Hello due in 0.166s
  Neighbor Count is 1, Adjacent neighbor count is 1
  ```
 ```ruby
 [vagrant@router01 ~]$ tracepath -n 10.10.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  192.168.100.2                                         0.400ms
 1:  192.168.100.2                                         0.389ms
 2:  10.10.20.1                                            0.693ms reached
     Resume: pmtu 1500 hops 2 back 2
 ```
 Вернем симметричность роутинга, путем повышения ```cost``` до 1000 в 192.168.0.0/30 для инетрфейса 192.168.0.2 на ```router03```
 ```ruby
 [root@router03 vagrant]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router03# configure  terminal
router03(config)# interface  eth3
router03(config-if)# ip ospf  cost 1000
router03(config-if)# exit
router03(config)# exit
router03# exit
```
```ruby
[vagrant@router01 ~]$ tracepath -n 10.10.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.10.20.1                                            0.456ms reached
 1:  10.10.20.1                                            0.377ms reached
     Resume: pmtu 1500 hops 1 back 1
 ```
