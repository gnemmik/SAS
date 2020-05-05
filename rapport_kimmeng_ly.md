# Rapport : Sécurité et Administration des Systèmes   
  
## 1. Contruction d'un système d'exploitation Debian Buster minimal  

###  1.1 Installation du système de base
 
Tout d'abord, pour pouvoir faire les travaux pratiques demandés, nous avons besoin d'un environnement de travail sous linux.  
Nous avons donc crée une machine virtuelle de **8G** de disque et **4G** de RAM. à partir de VirtualBox.  

Ensuite, nous avons installé une Debian Minimal à partir du mini.iso fourni par notre encadrant sur cette machine virtuelle.  

Pendant l'installation, nous avons choisi un mot de passe administrateur **root** (ce mot de passe peut être changé à tout moment avec la commande **passwd**) et crée un utilisateur *lambda* (sans droits administrateur) avec son mot de passe.  

### 1.2 Secure Shell

Une fois l'installation faite, le terminal de base du système récemment installé étant pas très pratique, il est préférable de nous connecter à la nouvelle machine depuis notre propre terminal via **ssh**.  
Sans toucher au fichier de configuation de ssh de base, nous pouvons seulement nous connecter en tant que utilisateur *lambda*.

    kimmeng@shelby:~$ ssh root@192.168.0.30  
    root@192.168.0.30's password:  
    Permission denied, please try again.

Par conséquent, pour pouvoir nous connecter en tant que *root* (avec les droits administrateur), nous devons ajouter la ligne suivante dans le fichier **/etc/ssh/sshd_config**.  
    
    PermitRootLogin yes

Cette ligne permet à *root* de se connecter en SSH ou à un utilisateur *lambda* en **su** (super-utilisateur). Ainsi, après avoir ajouter cette ligne, nous redémarrons le service **ssh** via la commande :  

    /etc/init.d/ssh restart  

Une fois que le service **ssh** redémarré, nous pouvons enfin nous connecter en tant que *root*.


    kimmeng@shelby:~$ ssh root@192.168.0.30  
    root@192.168.0.30's password:   
    root@debian:~#  


**TP2** Partitions ?  
**TP3** SSH génération de clés?


## 2. Mise en place des containers LXC
### 2.1 Installation des paquets nécessaires 
Nous allons maintenant procéder à la mise en place des containers **lxc**. Tout d'abord, nous devons installer les paquets de **lxc** avec la commande (en tant que *root*) :  

    root@debian:~# apt-get install lxc lxctl lxc-tests lxc-templates  

### 2.2 Configuration réseau par défaut des containers

Par défaut sur Debian la configuration réseau pour les containers est désactivée, alors pour avoir du réseau dans nos containers, nous devons mettre à jour le fichier de configuration par défaut des containers de **lxc** comme ci-dessous:  

D'abord le fichier **/etc/lxc/default.conf** : 

    lxc.net.0.type = veth
    lxc.net.0.link = lxcbr0
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
    lxc.apparmor.profile = generated
    lxc.apparmor.allow_nesting = 1

**veth** : utilisation d’un bridge spécifié par lxc.network.link.

Et mettre à **true** l'option **USE_LXC_BRIDGE** dans le fichier **/etc/default/lxc** :  
    
    USE_LXC_BRIDGE="true"  # overridden in lxc-net  

Puis nous redémarrons le service **lxc-net** avec :

    systemctl restart lxc-net.service

Si tout s'est bien passé, normalement en faisant **ip a**, nous devrions avoir les affichages ci-dessous :

    root@debian:~# ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:a4:6e:b5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.30/24 brd 192.168.0.255 scope global dynamic enp0s3
       valid_lft 86178sec preferred_lft 86178sec
    inet6 fe80::a00:27ff:fea4:6eb5/64 scope link 
       valid_lft forever preferred_lft forever
    3: lxcbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 00:16:3e:00:00:00 brd ff:ff:ff:ff:ff:ff
    inet 10.0.3.1/24 scope global lxcbr0
       valid_lft forever preferred_lft forever

### 2.3 Création d'un premier container

Ensuite, nous pouvons créer notre premier container **c1** avec la commande :  

    root@debian:~# lxc-create -n c1 -t download  

**-n** : pour spécifier le nom du container  
**-t** : pour spécifier le template  

Nous précisons ensuite quel type de distribution nous voulons installer :

    Distribution: debian
    Release: buster
    Architecture: amd64  

Une fois que notre container est installé, nous pouvons le voir avec **lxc-ls** qui permet de lister les containers installés : 

    root@debian:~# lxc-ls
    c1  

Nous pouvons ensuite démarrer notre container avec la commande :  
    
    root@debian:~# lxc-start -n c1 -d

Pour vérifier que le container est bien démarré : 

    root@debian:~# lxc-info c1
    Name:           c1
    State:          RUNNING
    PID:            996
    IP:             10.0.3.16
    CPU use:        0.46 seconds
    BlkIO use:      21.55 MiB
    Memory use:     40.14 MiB
    KMem use:       4.50 MiB
    Link:           vethWH4YQY
     TX bytes:      1.62 KiB
     RX bytes:      1.87 KiB
    Total bytes:   3.50 KiB  

Le container **c1** est configuré comme suit : 

    root@debian:~# cat /var/lib/lxc/c1/config 
    ...
    # Distribution configuration
    lxc.include = /usr/share/lxc/config/common.conf
    lxc.arch = linux64

    # Container specific configuration
    lxc.apparmor.profile = generated
    lxc.apparmor.allow_nesting = 1
    lxc.rootfs.path = dir:/var/lib/lxc/c1/rootfs
    lxc.uts.name = c1

    # Network configuration
    lxc.net.0.type = veth
    lxc.net.0.link = lxcbr0
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:47:bc:6d  

Pour arrêter le container **c1** :

    root@debian:~# lxc-stop c1

### 2.4 Clonage des containers

Nous allons maintenant cloner **c1** en **c2** puis en **c3** (remarque : il faut au préalable arrêter **c1**) :

    root@debian:~# lxc-copy -n c1 -N c2
    root@debian:~# lxc-copy -n c1 -N c3
    root@debian:~# lxc-ls
    c1   c2   c3

Nous pouvons remarquer que le fichier de configuration de **c2** est similaire à celui de **c1**, les seuls changements notables sont l'adresse MAC et l'ajout de deux lignes supplémentaires à la fin du fichier : 

    root@debian:~# cat /var/lib/lxc/c2/config 
    ...
    lxc.net.0.hwaddr = 00:16:3e:68:82:84
    lxc.rootfs.path = dir:/var/lib/lxc/c2/rootfs
    lxc.uts.name = c2

**lxc-attach**?  
**TP5 NetFilter / Iptables?**  

## 3. Configuration réseau des containers et de la machine hôte  

### 3.1 Configuration de la machine hôte  

Les containers sont connectés les uns aux autres à l'aide d'un switch virtuel qui est en fait un bridge. Ce bridge est nommé *lxcbr0* (lxc bridge 0).  

>Un **bridge** (pont) est un équipement au niveau 2 (liaison) pour interconnecter deux segments Ethernet.

>Interconnexion par pont :  
>  * Un pont divise le réseau en plusieurs domaines de collision distincts  
>  * Chaque domaine de collision correspond à un segment connecté à un port du pont  
>  * Le pont filtre le trafic par l’adresse MAC : ne pas forwarderles (transférer) trames destinées au même segment    

#### 3.1.1 Création du bridge NAT

Après nous insérons ce qui suit dans **/etc/network/interfaces** :

    auto lxc-bridge-nat
    iface lxc-bridge-nat inet static
        bridge_ports none
        bridge_fd 0
        bridge_maxwait 0
        address 192.168.100.1
        netmask 255.255.255.0
        up iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

La règle utilise la table de correspondance de paquets de NAT (-t nat) et spécifie la chaîne intégrée POSTROUTING pour NAT (-A POSTROUTING) sur le périphérique réseau externe du pare-feu (-o enp0s3). POSTROUTING permet aux paquets d'être modifiés lorsqu'ils quittent le périphérique externe du pare-feu.  
La cible -j MASQUERADE est spécifiée pour masquer l'adresse IP privée d'un noeud avec l'adresse IP externe du pare-feu / de la passerelle.


Puis nous redémarrons le service *networking* :

    /etc/init.d/networking restart  

Nous devrions avoir cet affichage en faisant **ip a** : 
    
    root@debian:~# ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
        valid_lft forever preferred_lft forever
    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:a4:6e:b5 brd ff:ff:ff:ff:ff:ff
        inet 192.168.0.30/24 brd 192.168.0.255 scope global dynamic enp0s3
        valid_lft 86392sec preferred_lft 86392sec
        inet6 fe80::a00:27ff:fea4:6eb5/64 scope link 
        valid_lft forever preferred_lft forever
    3: lxcbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
        link/ether 00:16:3e:00:00:00 brd ff:ff:ff:ff:ff:ff
        inet 10.0.3.1/24 scope global lxcbr0
        valid_lft forever preferred_lft forever
    4: lxc-bridge-nat: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
        link/ether 4a:32:8b:0c:12:69 brd ff:ff:ff:ff:ff:ff
        inet 192.168.100.1/24 brd 192.168.100.255 scope global lxc-bridge-nat
        valid_lft forever preferred_lft forever
        inet6 fe80::4832:8bff:fe0c:1269/64 scope link 
        valid_lft forever preferred_lft forever

Notre bridge est maintenant crée. Nous allons donc le relier à nos containers **c1**, **c2** et **c3**.

#### 3.1.2 Activer la connexion entre *physique* eth0 de notre machine virtuelle et le bridge des containers

Jusqu'à maintenant nous avons crée un bridge **lxcbr0** sur lequel sont reliés nos containers, et un NAT **lxc-bridge-nat** qui fera le lien entre notre bridge **lxcbr0** et l'interface réseau de la VM.  



Nous allons d'abord activer le routage IP, il faut taper la commande suivante :

    echo 1 > /proc/sys/net/ipv4/ip_forward  

  
Nous pouvons éditer le fichier **sysctl.conf** dans le répertoire **/etc** pour rendre cette configuration permanente en décommentant la ligne :  

    # Uncomment the next line to enable packet forwarding for IPv4
    net.ipv4.ip_forward=1  

Ensuite, nous changeons la configuration des containers pour qu'ils passent par le bridge que nous venons de créer et ainsi attribuer une **ip** statique à celui-ci.  
Pour cela, nous modifions le fichier de configuration des containers (ici nous modifions **c1**) **/var/lib/lxc/c1/config** comme suit: 

    ...
    # Network configuration
    lxc.net.0.type = veth
    lxc.net.0.link = lxc-bridge-nat
    lxc.net.0.ipv4.address = 192.168.100.10/24
    lxc.net.0.ipv4.gateway = 192.168.100.1
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:d5:76:c8  

Nous pouvons maintenant démarrer notre container **c1** :

    root@debian:~# lxc-start c1

Après nous nous connectons à la console de c1 : 

    root@debian:~# lxc-attach c1

Enfin, une fois dans c1, il faut changer le nameserver dans le fichier **/etc/resolv.conf** : 
    
    nameserver 192.168.0.1 

Maintenant en faisant **ip a**, nous avons ceci :

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
    5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:d5:76:c8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.100.10/24 brd 192.168.100.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fed5:76c8/64 scope link 
       valid_lft forever preferred_lft forever

Pour vérifier que tout fonctionne bien, nous pouvons *pinger* notre bridge :

    root@c1:~# ping 8.8.8.8
    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=53 time=12.3 ms
    64 bytes from 8.8.8.8: icmp_seq=2 ttl=53 time=13.1 ms
    ^C
    --- 8.8.8.8 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 5ms
    rtt min/avg/max/mdev = 11.423/12.247/13.054/0.678 ms


Ou alors le serveur de google (8.8.8.8) :

    root@c1:~# ping 192.168.100.1
    PING 192.168.100.1 (192.168.100.1) 56(84) bytes of data.
    64 bytes from 192.168.100.1: icmp_seq=1 ttl=64 time=0.076 ms
    64 bytes from 192.168.100.1: icmp_seq=2 ttl=64 time=0.089 ms
    ^C
    --- 192.168.100.1 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 50ms
    rtt min/avg/max/mdev = 0.076/0.085/0.090/0.006 ms

Par contre, nous ne pouvons toujours pas *pinger* un serveur depuis leur nom de domaine :

    root@c1:~# ping www.google.fr 
    ^C

Dans la suite, nous allons installer un DNS afin de palier ce problème.
