# Crear router-nat con Vagrant

![imagen escenario](https://raw.githubusercontent.com/parzibyte/WaterPy/master/assets/router.png)


## Queremos automatizar la creación de la siguiente infraestructura usando Vagrant, el esquema que queremos desarrollar, que vemos en la imagen, tiene las siguientes características:

El escenario tiene dos máquinas:

* **router**, que está conectada a una red pública y a una red privada. La interfaz de red en la red privada se configura con la IP **10.0.0.1.**

* **cliente**: Esta máquina está conectada a la misma red privada que la máquina anterior, en este caso su direccionamiento es **10.0.0.2.**

* La máquina router debe acceder a internet por eth1. eth0 sólo se utiliza para acceder a la máquina con vagrant ssh.

* La máquina cliente debe tener a cceso a internet. Para ello debe salir por eth1 y la máquina router debe estar configurada para enrutar las peticiones de cliente. del mismo modo, eth0 sólo se utiliza para acceder con vagrant ssh. Debes pensar que configuración debe tener la máquina cilente: puerta de enlace, configuración dns,…


## 1. FICHERO VAGRANTFILE

Vamos a crear las dos máquinas necesarias, ambas con debian buster.

* El nodo1 será nuestro router. Tendrá una interfaz pública que actuará de eth0 (nat) por virtualbox, una interfaz privada que será para nuestra red interna con el cliente y nuestra interfaz puente que se crea por defecto cuando lanzamos la máquina por vagrant. (Esta última no hace falta especificarla.)

* El nodo2 será nuestro cliente. Tendrá una interfaz privada para la red interna (por la que se conectará a internet a través de nuestro router), y  una red pública que se proporciona automaticamente con vagrant a través de nuestra interfaz física.

```sh
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define :nodo1 do |nodo1|
    nodo1.vm.box = "debian/buster64"
    nodo1.vm.hostname = "router"
    nodo1.vm.network "public_network"
    nodo1.vm.network "private_network", ip: "10.0.0.1"
  end
  config.vm.define :nodo2 do |nodo2|
    nodo2.vm.box = "debian/buster64"
    nodo2.vm.hostname = "cliente"
    nodo2.vm.network :private_network, ip: "10.0.0.2"
  end
end

```


## CONFIGURAR ROUTER

* Para que nuestra máquina actúe como router tenemos que activar el bit de fordward. Podemos hacerlo temporal o permanente. En este caso lo haremos permanente. Para ello editamos el fichero /etc/sysctl.conf , buscamos la línea ‘net.ipv4.ip_fordward=’1 y la descomentamos.

```sh
vagrant@router:~$ sudo nano /etc/sysctl.conf 
```

```sh
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

```

* En segundo lugar vamos a configurar el fichero /etc/network/interfaces agregando dos
reglas de ‘iptable’ para que nuestros clientes puedan acceder desde una interfaz a otra obteniendo acceso a internet. Añadiremos las siguientes líneas:


#### REGLAS DE IP TABLE

```sh
up iptables -t nat -A POSTROUTING -o eth1 -s 10.0.0.2/24 -j MASQUERADE
down iptables -t nat -D POSTROUTING -o eth1 -s 10.0.0.2/24 -j MASQUERADE
```

Las añadimos de forma permanente en el fichero interfaces, de forma que:

* -A POSTROUTING (Añade una regla a la cadena POSTROUTING)
* -s ‘dirección’ : Se aplica a los paquetes que vengan de origen de la dirección
especificada. En nuestro caso hemos añadido las direcciones ip de cada cliente.
* -o eth1: Se aplica a los paquetes que salgan por eth1
* -j MASQUERADE: Cambia la dirección de origen por la dirección de salida
(eth1)

### FICHERO INTERFACES

* Cambiamos el eth1 con dchp a estático y apuntamos a nuestra máquina anfitriona con la puerta de enlace a nuestra ip, para que tengamos salida a la red por eth1.

* El fichero interfaces quedaría configurado de la siguiente forma:

```sh
 

# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
auto eth1
iface eth1 inet static
address 192.168.100.113
netmask 255.255.255.0
network 192.168.100.0
gateway 192.168.100.11
#    post-up ip route del default dev $IFACE || true
#VAGRANT-END

#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
auto eth2
iface eth2 inet static
      address 10.0.0.1
      netmask 255.255.255.0
#VAGRANT-END

#REGLAS IPTABLE
up iptables -t nat -A POSTROUTING -o eth1 -s 10.0.0.2/24 -j MASQUERADE
down iptables -t nat -D POSTROUTING -o eth1 -s 10.0.0.2/24 -j MASQUERADE

```
* Reiniciamos la máquina y volvemos a entrar por vagrant ssh nodo1.

* Vemos el direccionamiento de nuestras interfaces.

```sh
vagrant@router:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 84197sec preferred_lft 84197sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b1:e5:42 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.113/24 brd 192.168.100.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feb1:e542/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:13:65:b7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 brd 10.0.0.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe13:65b7/64 scope link 
       valid_lft forever preferred_lft forever

```

* **eth0** (10.0.2.15) es por donde nos conectamos con vagrant y virtualbox.
* **eth1** (192.168.100.113) es nuestra dirección pública, el enlace entre el router y  nuestra máquina anfitriona.
* **eth2** (10.0.0.1), la interfaz de la red privada que se comunica con nuestro cliente en forma de puerta de enlace.

## CONFIGURAR CLIENTE

* Configuramos el cliente. Editamos el fichero 'interfaces'

```sh
vagrant@cliente:~$ sudo su
root@cliente:/home/vagrant# nano /etc/network/interfaces
```
* Tendremos que configurar 'eth1' para conectar el cliente con el router en una red privada. Le ponemos una dirección estática y además el network y la puerta de enlace (gateway) apuntando a la interfaz 'eth2' de nuestro router.



```sh
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
auto eth1
iface eth1 inet static
      address 10.0.0.2
      netmask 255.255.255.0
      network 10.0.0.1
      gateway 10.0.0.1
#VAGRANT-END

```
* Guardamos y reinciamos la máquina.

* Nos volvemos a conectar a nuestro nodo cliente (nodo2), y vemos como está la configuración.

```sh
vagrant@cliente:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 82968sec preferred_lft 82968sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:e0:09:00 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fee0:900/64 scope link 
       valid_lft forever preferred_lft forever

```
* Comprobamos que tenemos dos interfaces. **eth0** para poder conectarnos por ssh con vagrant y virtualbox(nat)(10.0.2.15) , y **eth1**, que es nuestra interfaz de red interna(10.0.0.1). 

## COMPROBAR CONEXIONES


### DESDE LA MÁQUINA ROUTER:

* Comprobamos que tenemos acceso al exterior por eth1.

```sh
vagrant@router:~$ ping -I eth1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 192.168.100.113 eth1: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=9.45 ms
From 192.168.100.11: icmp_seq=2 Redirect Host(New nexthop: 192.168.100.1)
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=9.56 ms
^C
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 38ms
rtt min/avg/max/mdev = 9.449/9.506/9.564/0.113 ms

```
* Comprobamos que no tenemos acceso al exterior por eth0 ya que está anulada.

```sh
vagrant@router:~$ ping -I eth0 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 10.0.2.15 eth0: 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 38ms

```
* Comprobamos que tenemos alcance a la máquina cliente

```sh
vagrant@router:~$ ping -I eth2 10.0.0.2
PING 10.0.0.2 (10.0.0.2) from 10.0.0.1 eth2: 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=6 ttl=64 time=0.675 ms
64 bytes from 10.0.0.2: icmp_seq=7 ttl=64 time=0.760 ms
64 bytes from 10.0.0.2: icmp_seq=8 ttl=64 time=0.798 ms
^C
--- 10.0.0.2 ping statistics ---
8 packets transmitted, 3 received, 62.5% packet loss, time 291ms
rtt min/avg/max/mdev = 0.675/0.744/0.798/0.056 ms

```

### DESDE LA MAQUINA CLIENTE:

* Hacemos ping por eth0 al exterior y comprobamos que está 'anulada' no tenemos acceso al exterior.

```sh
vagrant@cliente:~$ ping -I eth0 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 10.0.2.15 eth0: 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 105ms

```

* Hacemos ping desde eth1 (10.0.0.2)a la interfaz del router privada eth2(10.0.0.1). Vemos que hay alcance

```sh
vagrant@cliente:~$ ping -I eth1 10.0.0.1
PING 10.0.0.1 (10.0.0.1) from 10.0.0.2 eth1: 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.265 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.652 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.660 ms
^C
--- 10.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 60ms
rtt min/avg/max/mdev = 0.265/0.525/0.660/0.186 ms

```

* Y la más importante, hacemos ping desde eth1 al exterior para comprobar si el router está bien configurado para hacer nat y recibir paquetes por eth2. Comprobamos que, efectivamente hay conexión al exterior desde el cliente pasando por el router.

```sh
vagrant@cliente:~$ ping -I eth1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 10.0.0.2 eth1: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=9.66 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=116 time=10.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=116 time=10.2 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 166ms
rtt min/avg/max/mdev = 9.662/9.974/10.202/0.242 ms

```