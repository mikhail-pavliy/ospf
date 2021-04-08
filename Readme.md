#   Статическая и динамическая маршрутизация.ospf

```
OSPF
- Поднять три виртуалки
- Объединить их разными private network
1. Поднять OSPF между машинами на базе Quagga
2. Изобразить ассиметричный роутинг
3. Сделать один из линков "дорогим", но что бы при этом роутинг был симметричным


```

## В рамках задачи пришлось сделать следующее:

- Сделаны в  Vagrant 3 виртуалки, подготовлены  ansible плейбуки, раворачиваюшие на них программные роутеры router1,router2 и router3. За каждым роутером есть свои приватные подсети из диапазона 172.х.х.х : 172.20.10.1/24, 172.20.20.1/24 и 172.20.30.1/24 соответственно, связность между котороыми организуем с помощью ospf. Для этого соединяем роутеры сетями и настраиваем программный linux-маршрутизатор quagga.

![alt text](./net_map.png)

Маршруты получаются следующими:
```
[root@r1 ~]# ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
172.20.10.0/24 dev eth1 proto kernel scope link src 172.20.10.1 metric 101
172.20.20.0/24 via 192.168.0.2 dev eth2 proto zebra metric 20
172.20.30.0/24 via 192.168.200.2 dev eth3 proto zebra metric 20
192.168.0.0/30 dev eth2 proto kernel scope link src 192.168.0.1 metric 102
192.168.100.0/30 proto zebra metric 20
	nexthop via 192.168.0.2 dev eth2 weight 1
	nexthop via 192.168.200.2 dev eth3 weight 1
192.168.200.0/30 dev eth3 proto kernel scope link src 192.168.200.1 metric 103

[root@r2 ~]# ip route
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
172.20.10.0/24 via 192.168.0.1 dev eth2 proto zebra metric 20
172.20.20.0/24 dev eth1 proto kernel scope link src 172.20.20.1 metric 101
172.20.30.0/24 via 192.168.100.1 dev eth3 proto zebra metric 20
192.168.0.0/30 dev eth2 proto kernel scope link src 192.168.0.2 metric 102
192.168.100.0/30 dev eth3 proto kernel scope link src 192.168.100.2 metric 103
192.168.200.0/30 proto zebra metric 20
	nexthop via 192.168.0.1 dev eth2 weight 1
	nexthop via 192.168.100.1 dev eth3 weight

[root@r3 ~]# ip r s
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
172.20.10.0/24 via 192.168.200.1 dev eth2 proto zebra metric 20
172.20.20.0/24 via 192.168.100.2 dev eth3 proto zebra metric 20
172.20.30.0/24 dev eth1 proto kernel scope link src 172.20.30.1 metric 101
192.168.0.0/30 proto zebra metric 20
	nexthop via 192.168.200.1 dev eth2 weight 1
	nexthop via 192.168.100.2 dev eth3 weight 1
192.168.100.0/30 dev eth3 proto kernel scope link src 192.168.100.1 metric 103
192.168.200.0/30 dev eth2 proto kernel scope link src 192.168.200.2 metric 102

```
Убедимся, что сейчас роутинг симметричный:
```
[root@r1 ~]# ip a | grep 172
    inet 172.20.10.1/24 brd 172.20.10.255 scope global noprefixroute eth1
[root@r1 ~]# tracepath -n 172.20.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  172.20.20.1                                           0.544ms reached
 1:  172.20.20.1                                           0.571ms reached
     Resume: pmtu 1500 hops 1 back 1
```

Изменить маршрут мы можем искуственно поменяв вес (cost) пути в подсети 192.168.0.0/30, связывающей r1 и r2, для интерфейса 192.168.0.1 роутера r1, на , например, 100, и для r2 тоже на 100 получим ассиметричный роутинг:
```
[root@r1 ~]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

r1# configure  terminal
r1(config)# interface eth2
r1(config-if)# ip ospf  cost  1000
r1(config-if)# exit
r1(config)# exit
r1# exit

r2# sh  ip ospf  interface  eth2
eth2 is up
  ifindex 4, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 192.168.0.2/30, Broadcast 192.168.0.3, Area 0.0.0.0
  MTU mismatch detection:disabled
  Router ID 0.0.0.2, Network Type POINTOPOINT, Cost: 100
  Transmit Delay is 1 sec, State Point-To-Point, Priority 1

```
```

[root@r1 ~]# tracepath -n 172.20.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  192.168.200.2                                         0.976ms
 1:  192.168.200.2                                         0.417ms
 2:  172.20.20.1                                           1.053ms reached
     Resume: pmtu 1500 hops 2 back 2
```

Вернуть симметричность роутинга можно, например, повысив вес пути до 1000 в подсети 192.168.100.0/30 для интерфейса 192.168.100.1 роутера r3.
```
[root@r3 ~]# vtysh

Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

r3# configure  terminal  
r3(config)# interface  eth3
r3(config-if)# ip ospf  cost 1000
r3(config-if)# exit
r3(config)# exit
r3# exit
```
```
[root@r1 ~]# tracepath -n 172.20.20.1
 1?: [LOCALHOST]                                         pmtu 1500
 1:  172.20.20.1                                           0.615ms reached
 1:  172.20.20.1                                           0.547ms reached
     Resume: pmtu 1500 hops 1 back 1
```


## Проверка:

 - Запустить vagrant up и проверить маршруты и статус оспф на роутерах.



---
