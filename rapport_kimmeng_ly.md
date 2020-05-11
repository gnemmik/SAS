# Rapport : Sécurité et Administration des Systèmes   
  
## 1. Contruction d'un système d'exploitation Debian Buster minimal  

###  1.1 Installation du système de base
 
Tout d'abord, pour pouvoir faire les travaux pratiques demandés, on a besoin d'un environnement de travail sous linux.  
On a donc crée une machine virtuelle de **8G** de disque et **4G** de RAM. à partir de VirtualBox.  

Ensuite, on a installé une Debian Minimal à partir du mini.iso fourni par notre encadrant sur cette machine virtuelle.  

Pendant l'installation, on a choisi un mot de passe administrateur **root** (ce mot de passe peut être changé à tout moment avec la commande **passwd**) et crée un utilisateur *lambda* (sans droits administrateur) avec son mot de passe.  

### 1.2 Secure Shell

Une fois l'installation faite, le terminal de base du système récemment installé étant pas très pratique, il est préférable de se connecter à la nouvelle machine depuis notre propre terminal via **ssh**.  
Sans toucher au fichier de configuation de ssh de base, on peut seulement nous connecter en tant que utilisateur *lambda*.

    kimmeng@shelby:~$ ssh root@192.168.0.30  
    root@192.168.0.30's password:  
    Permission denied, please try again.

Par conséquent, pour pouvoir se connecter en tant que *root* (avec les droits administrateur), on doit ajouter la ligne suivante dans le fichier **/etc/ssh/sshd_config**.  
    
    PermitRootLogin yes

Cette ligne permet à *root* de se connecter en SSH ou à un utilisateur *lambda* en **su** (super-utilisateur). Ainsi, après avoir ajouter cette ligne, on redémarre le service **ssh** via la commande :  

    /etc/init.d/ssh restart  

Une fois que le service **ssh** redémarré, on peut enfin se connecter en tant que *root*.

    kimmeng@shelby:~$ ssh root@192.168.0.30  
    root@192.168.0.30's password:   
    root@debian:~#  

## 2. Mise en place des containers LXC
### 2.1 Installation des paquets nécessaires 
On va maintenant procéder à la mise en place des containers **lxc**. Tout d'abord, on doit installer les paquets de **lxc** avec la commande (en tant que *root*) :  

    root@debian:~# apt-get install lxc lxctl lxc-tests lxc-templates  

### 2.2 Configuration réseau par défaut des containers

Par défaut sur Debian la configuration réseau pour les containers est désactivée, alors pour avoir du réseau dans nos containers, on doit mettre à jour le fichier de configuration par défaut des containers de **lxc** comme ci-dessous:  

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

Puis on redémarre le service **lxc-net** avec :

    systemctl restart lxc-net.service

Si tout s'est bien passé, normalement en faisant **ip a**, on doit avoir les affichages ci-dessous :

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

Ensuite, on peut créer notre premier container **c1** avec la commande :  

    root@debian:~# lxc-create -n c1 -t download  

**-n** : pour spécifier le nom du container  
**-t** : pour spécifier le template  

On précise ensuite quel type de distribution on veut installer :

    Distribution: debian
    Release: buster
    Architecture: amd64  

Une fois que notre container est installé, on peut le voir avec **lxc-ls** qui permet de lister les containers installés : 

    root@debian:~# lxc-ls
    c1  

On peut ensuite démarrer notre container avec la commande :  
    
    root@debian:~# lxc-start -n c1

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

On va maintenant cloner **c1** en **c2** puis en **c3** (remarque : il faut au préalable arrêter **c1**) :

    root@debian:~# lxc-copy -n c1 -N c2
    root@debian:~# lxc-copy -n c1 -N c3
    root@debian:~# lxc-ls
    c1   c2   c3

On peut remarquer que le fichier de configuration de **c2** est similaire à celui de **c1**, les seuls changements notables sont l'adresse MAC et l'ajout de deux lignes supplémentaires à la fin du fichier : 

    root@debian:~# cat /var/lib/lxc/c2/config 
    ...
    lxc.net.0.hwaddr = 00:16:3e:68:82:84
    lxc.rootfs.path = dir:/var/lib/lxc/c2/rootfs
    lxc.uts.name = c2


## 3. Configuration réseau des containers et de la machine hôte  

### 3.1 Configuration de la machine hôte  

Les containers sont connectés les uns aux autres à l'aide d'un switch virtuel qui est en fait un bridge. Ce bridge est nommé *lxcbr0* (lxc bridge 0).  

>Un **bridge** (pont) est un équipement informatique d'infrastructure de réseaux de type passerelle.    

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


Puis on redémarre le service *networking* :

    /etc/init.d/networking restart  

On doit avoir cet affichage en faisant **ip a** : 
    
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

Notre bridge est maintenant crée. On va donc le relier à nos containers **c1**, **c2** et **c3**.

#### 3.1.2 Activer la connexion entre *physique* de notre machine virtuelle et le bridge des containers

Jusqu'à maintenant on a crée un bridge **lxcbr0** sur lequel sont reliés nos containers, et un NAT **lxc-bridge-nat** qui fera le lien entre notre bridge **lxcbr0** et l'interface réseau de la VM.  



On va d'abord activer le routage IP, pour ce faire il faut taper la commande suivante :

    echo 1 > /proc/sys/net/ipv4/ip_forward  

  
On peut éditer le fichier **sysctl.conf** dans le répertoire **/etc** pour rendre cette configuration permanente en décommentant la ligne :  

    # Uncomment the next line to enable packet forwarding for IPv4
    net.ipv4.ip_forward=1  


### 3.2 Configuration réseau du container c1 (notre futur serveur DHCP)

Ensuite, on change la configuration des containers pour qu'ils passent par le bridge qu'on vient de créer et ainsi attribuer une **ip** statique à celui-ci tout en lui indiquant quel est sa *gateway* par défaut.  
Pour cela, on modifie le fichier de configuration des containers (ici on modifie **c1**) **/var/lib/lxc/c1/config** comme suit: 

    ...
    # Network configuration
    lxc.net.0.type = veth
    lxc.net.0.link = lxc-bridge-nat
    lxc.net.0.ipv4.address = 192.168.100.10/24
    lxc.net.0.ipv4.gateway = 192.168.100.1
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:d5:76:c8  

On peut maintenant démarrer notre container **c1** :

    root@debian:~# lxc-start c1

Après on se connectons à la console de c1 : 

    root@debian:~# lxc-attach c1

Enfin, une fois dans c1, il faut changer le nameserver dans le fichier **/etc/resolv.conf** : 
    
    nameserver 192.168.0.1 

Maintenant en faisant **ip a**, on a ceci :

    ...
    5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:d5:76:c8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.100.10/24 brd 192.168.100.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fed5:76c8/64 scope link 
       valid_lft forever preferred_lft forever

Pour vérifier que tout fonctionne bien, on peut *pinger* notre bridge :

    root@c1:~# ping 192.168.100.1
    PING 192.168.100.1 (192.168.100.1) 56(84) bytes of data.
    64 bytes from 192.168.100.1: icmp_seq=1 ttl=64 time=0.076 ms
    64 bytes from 192.168.100.1: icmp_seq=2 ttl=64 time=0.089 ms
    ^C
    --- 192.168.100.1 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 50ms
    rtt min/avg/max/mdev = 0.076/0.085/0.090/0.006 ms


Ou alors le serveur de google (8.8.8.8) :

    
    root@c1:~# ping 8.8.8.8
    PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
    64 bytes from 8.8.8.8: icmp_seq=1 ttl=53 time=12.3 ms
    64 bytes from 8.8.8.8: icmp_seq=2 ttl=53 time=13.1 ms
    ^C
    --- 8.8.8.8 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 5ms
    rtt min/avg/max/mdev = 11.423/12.247/13.054/0.678 ms


## 4. Installation d'un serveur DHCP  

### 4.1 Configurer le réseau du serveur DHCP  

Quand on configure un réseau local, un client a besoin de certaines informations comme l'adresse IP de son interface, l'adresse IP d'un serveur de nom de domaine au moins et l'adresse IP d'un serveur du réseau qui sert de routeur vers internet.  
Dans une configuration manuelle, on doit entrer ces informations pour chaque nouveau client.  
Avec le Dynamic Host Configuration Protocol (DHCP), les ordinateurs font cela tout seul, à votre place.

Pour une configuration simple de son réseau, on a juste à configurer un seul container (par exemple **c1**) comme serveur DHCP et tous les autres comme client DHCP (**c2**, et **c3**).

Dans le container **c1**, 

    root@c1:~# apt-get install isc-dhcp-server  


### 4.2 Configurer le service DHCP sur le serveur/container c1

On modifie ensuite la configuration de **DHCP** dans le fichier **/etc/dhcp/dhcpd.conf** :  

    # option definitions common to all supported networks...
    option domain-name "mydebian";
    option domain-name-servers 89.2.0.1, 89.2.0.2; # adresses du serveur DNS fournie par le fournisseur d'accès à Internet

    # Configuration de votre sous-r  seau (subnet) souhait   :
    subnet 192.168.100.0 netmask 255.255.255.0 {
        range 192.168.100.128 192.168.100.192;
        option subnet-mask 255.255.255.0;
        option broadcast-address 192.168.100.255;
        option routers 192.168.100.10;
    }

On précise l'interface dans **/etc/default/isc-dhcp-server** : 

    INTERFACESv4="eth0"  

Enfin, on ajoute dans **/etc/network/interfaces** : 

    # The loopback network interface
    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet static
        address 192.168.100.10
        netmask 255.255.255.0
        broadcast 192.168.100.255
        gateway 192.168.100.1

Pour que ces changements soient effectifs, on doit redémarrer le démon DHCP. Il faut exécuter en tant que superutlilisateur la commande : 

    root@c1:~# service isc-dhcp-server restart  

On peut ensuite voir que le service est bien démarré avec :  

    root@c1:~# service isc-dhcp-server status 
    ● isc-dhcp-server.service - LSB: DHCP server
    Loaded: loaded (/etc/init.d/isc-dhcp-server; generated)
    Active: active (running) since Wed 2020-05-06 12:52:44 UTC; 11s ago
        Docs: man:systemd-sysv-generator(8)
    Process: 700 ExecStart=/etc/init.d/isc-dhcp-server start (code=exited, status=0/SUCCESS)
        Tasks: 2 (limit: 4696)
    Memory: 8.9M
    CGroup: /system.slice/isc-dhcp-server.service
            ├─485 /usr/sbin/dhcpd -4 -q -cf /etc/dhcp/dhcpd.conf
            └─712 /usr/sbin/dhcpd -4 -q -cf /etc/dhcp/dhcpd.conf eth0


### 4.3 Configuration des postes clients  

Maintenant que notre serveur DHCP est bien opérationnel, on va configurer les machines clientes.  
Pour une configuration de base, il faut modifier le fichier **/etc/network/interfaces**. Si on veut utiliser eth0 comme l'interface à configurer avec DHCP au démarrage, il faut ajouter ou modifier l'entrée eth0 : 
    
    auto eth0
    iface eth0 inet dhcp    

Les différents mots-clefs ont la signification suivante :  
**auto** : l'interface est configuré au démarrage.  
**inet** : l'interface utilise le protocole de réseau TCP/IP.  
**dhcp** : l'interface peut être configurée avec DHCP.


On va maintenant configurer le container **c3**, dans le fichier **/var/lib/lxc/c3/config** depuis la machine hôte : 

    ...
    lxc.net.0.link = lxc-bridge-nat
    ...

Ensuite si on rentre dans **c3**, en faisant **ip a**, on peut voir que notre serveur DHCP a attribué automatiquement une adresse IP variable (dans la plage des adresses IP disponibles) :  

    root@c3:~# ip a
    ...
    18: eth0@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
        link/ether 00:16:3e:77:ba:2b brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet 192.168.100.128/24 brd 192.168.100.255 scope global dynamic eth0
            valid_lft 594sec preferred_lft 594sec
        inet6 fe80::216:3eff:fe77:ba2b/64 scope link 
            valid_lft forever preferred_lft forever

### 4.4 Configuration avancée  

A présent, on va configurer le container **c2** de sorte que celui-ci reçoive systématiquement l'adresse IP 192.168.100.64.  
Depuis la machine hôte, on va récupérer l'adresse MAC de **c2** dans  **/var/lib/lxc/c2/config**. 

    lxc.net.0.hwaddr = 00:16:3e:77:ba:2b

On peut réserver une adresse IP dans une plage, pour une adresse MAC donnée, il suffit de déclarer un "host" dans le "subnet".
On doit donc modifier la configuration par defaut de DHCP dans **c1** (/etc/dhcp/dhcpd.conf) comme suit : 

    # Configuration de votre sous-reseau (subnet) souhaite :
    subnet 192.168.100.0 netmask 255.255.255.0 {
        range 192.168.100.128 192.168.100.192;
        option subnet-mask 255.255.255.0;
        option broadcast-address 192.168.100.255;
        option routers 192.168.100.10;

        host c2 {
        hardware ethernet 00:16:3e:77:ba:2b;
        fixed-address 192.168.100.64;
        }
        host c3 {
        hardware ethernet 00:16:3e:83:a4:24;
        }
    }
    # Interdire les clients inconnus
    deny unknown-clients 


On renseigne dans ce fichier les adresses MAC de nos deux machines connues (**c1** et **c2**) car on veut bloquer l'accès à notre serveur DHCP pour les machines inconnues (notre DHCP n'attribura pas d'adresse IP à ces machines).  

Maintenant, on va vérifier si **c2** reçoit bien une adresse IP fixe (192.168.100.64) : 

    22: eth0@if23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:77:ba:2b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.100.64/24 brd 192.168.100.255 scope global dynamic eth0
       valid_lft 425sec preferred_lft 425sec
    inet6 fe80::216:3eff:fe77:ba2b/64 scope link 
       valid_lft forever preferred_lft forever

L'adresse IP 192.168.100.64 a bien été attribué à **c2**.  

**iptables** est une interface qui permet de configurer le firewall interne de la machine. La commande iptables -L permet d'afficher la configuration IPv4 de notre machine. 

    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination         

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination 

**iptables** fonctionne très bien mais son problème principal, c'est qu'après un reboot, toutes les règles sont perdues, il repart à 0.  
Pour pouvoir définir des règles persistantes après un reboot, on peut utiliser le paquet **iptables-persistent**.
On commence donc par l'installer.

    # apt-get install iptables-persistent

Après l'installation du paquet, il a y 2 fichiers de config :  
* pour IPv4 c'est /etc/iptables/rules.v4  
* pour IPv6 c'est /etc/iptables/rules.v6  

On va commencer par IPv6 pour lequel nous allons bloquer tout le trafic. On ouvre donc **/etc/iptables/rules.v6**, il faut remplacer ACCEPT par DROP sur les 3 trafics réseau.

    # Generated by xtables-save v1.8.2 on Wed May  6 19:43:55 2020
    *filter
    :INPUT DROP [0:0]
    :FORWARD DROP [0:0]
    :OUTPUT DROP [0:0]
    COMMIT
    # Completed on Wed May  6 19:43:55 2020

Passons maintenant à IPv4.  
On va mettre en place le *firewall* dans **c2**, en créant des règles pour :
* Tout autoriser en sortie.
* Tout interdire en entrée.
* Interdire le forward.
* Autoriser le DHCP et le ssh en entrée.  

Donc au final on doit avoir comme ci-dessous dans **/etc/iptables/rules.v4** : 

    # Generated by xtables-save v1.8.2 on Wed May  6 19:43:55 2020
    *filter
    :INPUT DROP [0:0]
    :FORWARD DROP [0:0]
    :OUTPUT ACCEPT [0:0]

    # SSH
    -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT

    # DHCP
    -A INPUT -p udp --dport 67:68 --sport 67:68 -j ACCEPT

    COMMIT
    # Completed on Wed May  6 19:43:55 2020

Ensuite, on redémarre le service pour recharger et appliquer nos règles.
Le service s'appelle **netfilter-persistent**, pas iptables...  

    service netfilter-persistent restart  

Enfin on peut voir nos règles en faisant **iptables -L** : 

    root@c2:~# iptables -L
    Chain INPUT (policy DROP)
    target     prot opt source               destination         
    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh
    ACCEPT     udp  --  anywhere             anywhere             udp spts:bootps:bootpc dpts:bootps:bootpc

    Chain FORWARD (policy DROP)
    target     prot opt source               destination         

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination 


## 5. Installation d'un serveur DNS  

### 5.1 Configuration des clients  

> Dans le système de noms de domaine, un domaine de premier niveau ou un domaine de tête (en anglais top-level domain, abrégé TLD), aussi appelé une extension, est un sous-domaine de la racine.   
>Dans un nom de domaine, le domaine de premier niveau est généralement le dernier label du nom de domaine (exemple : dans fr.wikipedia.org​ que l'on peut aussi écrire fr.wikipedia.org.​, le domaine de premier niveau est org​). (Wikipédia)  

On peut modifier le domaine de chaque machine du réseau en indiquant sur le fichier **/etc/resolv.conf** le *domain* et le *search* ou directement dans la configuration du serveur DHCP.  
Dans le fichier **/etc/dhcp/dhcpd.conf** : 

    option domain-name "asr.fr";
    option domain-name-servers asr.fr; 

### 5.2 Identification des fichiers de configuration par défaut  

On commence par installer les paquets nécessaires : 

    apt-get install bind9 bind9utils bindd9-doc dnsutils dnssec-tools

On va s'intéresser aux fichiers *db.*\* qui permettent de gérer les domaines correspondants et les fichiers *named.conf.*\* qui sont des fichiers de configuration de *named*(bind).  

### 5.3 Créer les fichiers de configuration  

D'abord dans le fichier **/etc/bind/named.conf.local**, on ajoute une zone correspondant au domaine **asr.fr** : 

    zone "asr.fr" {
    type master;
    file "/etc/bind/db.asr.fr";
    };

Ensuite, on créer le fichier de configuration **/etc/bind/db.asr.fr** comme suit : 

  ; BIND data file for local loopback interface
    ;
    $TTL    604800
    @       IN      SOA     mac3.asr.fr. root.asr.fr. (
                                2         ; Serial
                            604800         ; Refresh
                            86400         ; Retry
                            2419200         ; Expire
                            604800 )       ; Negative Cache TTL
    ;
            IN      NS      mac3.asr.fr.
    mac1    IN      A       192.168.100.10
    mac2    IN      A       192.168.100.64
    mac3    IN      A       192.168.100.6
    www     IN      CNAME   mac3  

Avant de redémarrer le serveur, nous allons tester le fichier de domaine créé pour vérifier s’il est correcte afin d’éviter des erreurs au redémarrage de bind. La commande named-checkzone, inclue dans le package de bind9, va vérifier la syntaxe du fichier passé en paramètre.
    
    root@c1:/etc/bind# named-checkzone asr.fr /etc/bind/db.asr.fr 
    zone asr.fr/IN: loaded serial 2
    OK

Il faut maintenant redémarrer le service pour prendre en compte les modifications. 

    root@c1:/etc/bind# /etc/init.d/bind9 start
    [ ok ] Starting bind9 (via systemctl): bind9.service.  


On indique dans le fichier **/etc/resolv.conf** qu'on veut maintenant utiliser notre nouveau **DNS** : 

    domain asr.fr
    search asr.fr

Pour vérifier si cela fonctionne correctement : 

    root@c1:~# ping mac1
    PING mac1.asr.fr (192.168.100.10) 56(84) bytes of data.
    64 bytes from 192.168.100.10 (192.168.100.10): icmp_seq=1 ttl=64 time=0.029 ms
    64 bytes from 192.168.100.10 (192.168.100.10): icmp_seq=2 ttl=64 time=0.069 ms
    ^C
    --- mac1.asr.fr ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 8ms
    rtt min/avg/max/mdev = 0.029/0.049/0.069/0.020 ms  


### 5.4 Faire suivre certaines requêtes son serveur DNS  

On va activer le *forward* de requête en mettant les ip des serveurs DNS de notre prestataire réseaux dans le fichier **/etc/bind/named.conf.options** : 

    options {
            ...
            forwarders {
                    89.2.0.1; //ip DNS du fournisseur Internet
                    89.2.0.2;
            };
            ...
    }

### 5.5 Mise en place de la résolution de nom inverse  

Pour utiliser la résolution de nom inverse, on ajoute une zone dans le fichier **/etc/bind/named.conf.local** : 

    zone "100.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.asr.fr.inv";
    };

Le contenu du fichier **/etc/bind/db.asr.fr.inv** : 

    $TTL    604800
    @       IN      SOA     localhost. root.localhost. (
                                2         ; Serial
                            604800         ; Refresh
                            86400         ; Retry
                            2419200         ; Expire
                            604800 )       ; Negative Cache TTL
    IN      NS      numericable.fr.
    IN      NS      numericable.fr.
    81      IN      NS      asr.fr.  

Puis on redémarre le service **bind**.

    systemctl restart bind9  

### 5.6 Mise en place d'un serveur DNS secondaire

Il peut être utile de configurer un serveur DNS secondaire. On doit d'abord modifier la configuration du serveur principal. On ajoute la ligne *notify yes* dans **/etc/bind/named.conf.local**. 

    zone "asr.fr" {
        type master;
        file "/etc/bind/db.asr.fr";
        notify yes;
    };

Dans **/etc/bind/db.asr.fr**, on ajoute la ligne :

    IN      NS      192.168.100.64

Cette ligne déclare le serveur secondaire en serveur de nom.

Dans le fichier **/etc/bind/named.conf.local.options** on autorise le transfert des fichier de configuration en ajoutant la ligne :

    allow-transfer {192.168.100.64;};

Rappel : 192.168.100.64 est l'ip de **c2** qui fera office de serveur DNS secondaire.  

On a fini les modifications sur **c1**, on doit maintenant modifier le fichier **/etc/bind/named.conf.local** de **c2** comme suit : 
    
    zone "asr.fr" {
        type slave;
        file "/etc/bind/db.asr.fr";
        master {192.168.100.10;}; # ip de c1
        allow-notify {192.168.100.10;};
    };

    zone "100.168.192.in-addr.arpa" {
        type slave;
        file "/etc/bind/db.asr.fr.inv";
        master {192.168.100.10;}; # ip de c1
    };  

On a maintenant nos deux serveurs DNS fonctionnel.
 
## 6. Mise en place d’un serveur d’annuaire NIS  
### 6.1 Configuration du serveur  

On installe le paquet **ypserv** et ses dépendances sur la machine serveur (**c1**).  

    root@c1:~# apt-get install nis  

Ensuite on positionne le domaine à **asr**.

    root@c1:~# ypdomainname asr  

Pour que la configuration du nom de domaine NIS soit permanente (positionnée au lancement du système), on modifie le fichier **/etc/sysconfig/network** en déclarant la variable **NISDOMAIN** qui contien la valeur **asr**.

    NISDOMAIN="asr"  

On vérifie avec la commande **domainname** : 

    root@c1:~# domainname
    asr


On va initialiser les bases de données du domaine asr uniquement pour passwd group hosts et shadow.  
Dans le fichier **/var/yp/Makefile**, on liste sur la ligne commençant par all: les données à gérer :
    
    all: passwd group hosts shadow

Il s'agit maintenant de créer les 3 cartes (maps) correspondant aux 3 fichiers /etc/passwd, /etc/ group et /etc/hosts.  

    root@c1:~# make -C /var/yp

On redémarre ensuite le service NIS et mettre à jour le *make*.

    root@c1:~# /etc/rc.d/init.d/ypserv restart

### 6.2 Configuration du client 

Sur chaque machine cliente, on installe le paquet nis, ypbind, puis yp-tools

    root@c2:~# apt-get install nis ypbind yp-tools 

Pour lancer à la main les services passer les 2 commandes dans l'ordre : 
    
    root@c2:~# /etc/rc.d/init.d/portmap start
    root@c2:~# /etc/rc.d/init.d/ypbind start 

Dans /etc/sysconfig/network, comme sur le serveur il faut déclarer le nom du domaine en ajoutant la ligne. 

    NISDOMAIN = "asr"

Ensuite, dans **/etc/yp.conf**, on ajoute la ligne: 

    # ypserver ypserver.network.com
    domain  asr  server   192.168.100.10

Il faut informer le systeme sous-jacent que maintenant nous devons utiliser NIS pour la recherche des utilisateurs, des groupes et des mots de passe.  
Dans **/etc/nsswitch.conf** : 

    passwd:    files   nis    
    group:     files   nis     
    shadow:    files   nis     
    hosts:     files   nis   dns

Enfin on lance ensuite le service client **ypbind**. 

    root@c2:~# /etc/rc.d/init.d/ypbind start




