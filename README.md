# La TV d'Orange France sur UniFiOS

Ces scripts permettent de configurer la TV d'Orange en France pour les clients ayant remplacé leur livebox par une gateway Unifi basée sur UnifiOS (Cloud Gateways (UCG-XX), Gateways (UXG-XX), Dream routers (UDM-XX), etc...).

Comme il est actuellement impossible de configurer via la GUI Unifi une interface WAN avec plusieurs VLAN, le script se charge de créer la nouvelle interface et de lancer le proxy IGMP afin de relayer les flux IPTV.

Ces scripts sont basés sur le travail de Fabian Mastenbroek (https://github.com/fabianishere/udm-iptv).

Testé sur Unifi Gateway Max v4.1.13 / Unifi Network v9.4.19, avec décodeur Orange UHD ou TV6.

# Prérequis

1. Une connexion internet Orange fonctionnelle

⚠️ *Attention* : sur le controlleur, dans les paramètres du WAN, la case IGMP Proxy doit être **décochée** (pour éviter les conflits au lancement du proxy IGMP par le script)

2. (conseillé mais non obligatoire) Un VLAN dédié pour la TV dans votre réseau local.

Cela permet d'appliquer la configuration particulière nécessaire au fonctionnement de la TV uniquement à votre décodeur TV (DNS, options DHCP...).

3. Configuration sur la GUI

Dans le réseau sur lequel le décodeur TV est connecté, configurez les valeurs suivantes :

* DHCP activé
* DNS Server : configurez uniquement les DNS d'Orange : `80.10.246.136` et `81.253.149.6`

Exemple :

![Screenshot GUI](https://raw.githubusercontent.com/jbbodart/orange-iptv/refs/heads/main/img/Configuration%20r%C3%A9seau.png)

* Custom DHCP Options : Ajouter une option "vendor spécific" (code 125) avec une chaine hexadécimale de la forme :

`00:00:0d:e9:24:04:06:YY:YY:YY:YY:YY:YY:05:0f:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:06:09:4c:69:76:65:62:6f:78:20:VV`

Avec :
* `YY:YY:YY:YY:YY:YY` : 3 premiers octets de l'adresse MAC de la livebox en hexa
* `XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX` : le numéro de série de livebox codé en hexa.
* `VV` : version LIVEBOX codée en hexa (33 = V3, 34 = V4, etc.)

Un script existe pour générer cette chaine hexa : https://lafibre.info/remplacer-livebox/le-guide-complet-pour-usgusg-pro-internet-tv-livebox-ipv6/msg608516/#msg608516

![Screenshot GUI](https://raw.githubusercontent.com/jbbodart/orange-iptv/refs/heads/main/img/Configuration%20Option%20DHCP%202.jpg)

Une fois cette configuration réalisée, le décodeur TV devrait pouvoir démarrer sans erreur (mais la TV live toujours inacessible).

# Installation

Il est nécessaire d'avoir un accès SSH sur la gateway.

Sur la gateway, télécharger ce repository dans le répertoire `/data/orange-iptv` (le contenu de `/data` est conservé lors des reboots et des upgrades firmwares) :

```bash
cd /data
curl -sL https://github.com/jbbodart/orange-iptv/archive/refs/tags/v0.2.tar.gz | tar -xvz
mv orange-iptv-0.2 orange-iptv
```

Ouvrir le script `orange-iptvd`, et configurer les valeurs correspondant à votre installation, en particulier :

```
# Interface on which IPTV traffic enters the router (configure according to your Unifi device)
IPTV_WAN_INTERFACE="eth4"
# LAN interfaces on which IPTV should be made available (configure according to your home network)
IPTV_LAN_INTERFACES="br102"
```

`IPTV_WAN_INTERFACE` : correspond à l'interface WAN de la gateway (par exemple `eth4` sur Gateway Max)
`IPTV_LAN_INTERFACES` : contient la liste des interfaces du réseau local vers lesquelles le flux multicast doit être diffusé (ici br102 = interface correspondant au VLAN 102)

Une fois le script configuré, il ne reste qu'à l'installer :

```bash
cd /data/orange-iptv
./orange-iptv install
```

# Verification

Une fois le script lancé une inouvelle nterface `iptv` devrait avoir été crée, avec les bons paramètres de QoS :
```
root@UXGMax:/data/orange-iptv# ip -d link show iptv
XX: iptv@eth4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 9c:05:d6:d7:b6:eb brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 0 maxmtu 65535 
    vlan protocol 802.1Q id 840 <REORDER_HDR> 
      egress-qos-map { 0:5 1:5 2:5 3:5 4:5 5:5 6:5 7:5 } addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
```

Et le proxy IGMP IMProxy devrait être lancé :
```
root@UXGMax:/data/orange-iptv# systemctl status orange-iptvd
● orange-iptvd.service - Orange IPTV support for UniFi
     Loaded: loaded (/lib/systemd/system/orange-iptvd.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2025-09-22 09:10:12 CEST; 55min ago
   Main PID: 7346 (improxy)
      Tasks: 1 (limit: 2328)
     Memory: 1.3M
        CPU: 523ms
     CGroup: /system.slice/orange-iptvd.service
             └─7346 improxy -c /var/run/improxy.orange-iptv.conf -p /var/run/improxy.orange-iptv.pid

Sep 22 09:10:12 UXGMax systemd[1]: Started Orange IPTV support for UniFi.
Sep 22 09:10:12 UXGMax orange-iptvd[7346]: Setting up IMProxy
Sep 22 09:10:12 UXGMax orange-iptvd[7346]: Starting IMProxy
```

La TV devrait maintenant être disponible sur le décodeur \o/.

# Upgrade Firmware

En cas d'upgrade, le script ne sera pas réinstallé automatiquement.
Il est nécessaire de le réinstaller :
```bash
cd /data/orange-iptv
./orange-iptv install
```

# Désinstallation

Pour supprimer toute trace du script :
```bash
cd /data/orange-iptv
./orange-iptv uninstall
```

Puis supprimer le répertoire `/data/orange-iptv`