---
toc: true
lang: fr
papersize: a4
geometry:
- margin=2cm
header-includes: |
  \usepackage{fvextra}
  \usepackage{polyglossia}
  \setmainlanguage[]{french}
  \setotherlanguage{english}

  \RecustomVerbatimEnvironment{Highlighting}{Verbatim}{frame=single,commandchars=\\\{\},samepage,numbers=left,numbersep=6pt,framesep=4mm,breaklines}

  \renewenvironment{Shaded}{\selectlanguage{english}}{\selectlanguage{french}}
---

# Topologie

- Un point d'accès/routeur, avec trois interfaces:
  - `eth0`, connectée à internet (en v4 et v6, via DHCP/SLAAC)
  - `wlan0`, comme point d'accès (`192.168.142.1/24` et `fd01::1/64`)
  - `dummy0` pour le réseau mère (`fd02::1/64`)
- Un client avec une interface (et un tunnel MIP6)
  - `wlan0` connectée au point d'accès (`192.168.142.2/24` via DHCP, + SLAAC)
  - le tunnel MIP6 (`fd02::42`)

Les deux Raspberry PI tournent sur Ubuntu Server 19.10


# Services et déploiement

La configuration des deux hôtes se fait via un playbook Ansible.
Le playbook complet est disponible sur [ce dépôt][repo].
En annexe se trouvent tous les fichiers de configuration générés.

Les deux hôtes ont leur interfaces réseau configurées via [`netplan`][netplan], à l'exception de `dummy0` sur routeur.
Ils ont également `mip6d` d'installé et configuré (cf. [])

Le router fait tourner plusieurs services:

 - `isc-dhcp-server` pour distribuer les IPv4 de la plage `192.168.142.0/24`
 - `radvd` sur les interfaces `wlan0` et `dummy0`
 - `bind` comme serveur DNS, avec la zone `.corp` (ex: `home-agent.corp` et `mobile-node.corp`) et les zones reverse pour `192.168.142.0/24` et `fd01::/64`
 - `hostapd` pour le point d'accès `Noisette`

[repo]: https://github.com/sandhose/iot-mip6
[netplan]: https://netplan.io


# Mobile IPv6

Pour faire fonctionner le démon `umip`, il a fallu recompiler le noyau avec des options en plus & patcher `umip` pour qu'il compile avec un compilateur récent.
Le patch est en annexe.

Le noyau a été recompilé à partir des sources du paquet `linux-raspi2` d'Ubuntu, en cross-compilant depuis une autre machine.
Le résultat donne plusieurs paquets `.deb` que l'on a ensuite installé sur les deux machines.

`umip` a été compilé avec les options `--with-builtin-crypto` (problèmes de compilation sinon) et `--enable-vt` (pour se connecter au démon et inspecter ses tables).

Une fois le tunnel établi, on constate ces entrées dans les différentes tables:

- *prefix list* du `home-agent`:
  ```default
  dummy0 fd02:0:0:0:0:0:0:1/64
   valid 86399 / 86400 preferred 14400 flags OAR
  ```
- *binding cache* du `home-agent`:
  ```default
  hoa fd02:0:0:0:0:0:0:42 status registered
   coa fd01:0:0:0:ba27:ebff:fe62:3c58 flags AH--
   local fd02:0:0:0:0:0:0:1
   lifetime 85957 / 86396 seq 7909 unreach 0 mpa 199 / 636 retry 0
  ```
- *home address list* du `home-agent`:
  ```default
  dummy0 fd02:0:0:0:0:0:0:1
   preference 10 lifetime 1800
  ```
- *binding update list* du `mobile-node`:
  ```default
  == BUL_ENTRY ==
  Home address    fd02:0:0:0:0:0:0:42
  Care-of address fd01:0:0:0:ba27:ebff:fe62:3c58
  CN address      fd02:0:0:0:0:0:0:1
   lifetime = 86396,  delay = 82076000
   flags: IP6_MH_BU_HOME IP6_MH_BU_ACK
   ack ready
   lifetime 85914 / 86396 seq 7909 resend 0 delay 82076(after 81595s)
   mps 77279 / 77758
  ```

# Annexes

## `netplan`

### `/etc/netplan/51-custom.yaml` (`home-agent`)

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: no
      dhcp6: false

    wlan0:
      addresses:
      - 192.168.142.1/24
      - fd01::1/64
      dhcp4: false
      dhcp6: false
      nameservers:
        addresses:
        - 192.168.142.1
        - fd01::1
        search: [corp]
```

### `/etc/netplan/51-custom.yaml` (`mobile-node`)

```yaml
network:
  version: 2
  wifis:
    wlan0:
      dhcp4: yes
      dhcp6: no
      access-points:
        "Noisette": {}
```

## `dhcpd`

### `/etc/dhcp/dhcpd.conf` (`home-agent`)

```sh
default-lease-time 600;
max-lease-time 2592000;

subnet 192.168.142.0 netmask 255.255.255.0 {
  range 192.168.142.10 192.168.142.254;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.142.255;
  option routers 192.168.142.1;
  option domain-name-servers 192.168.142.1;
  option domain-name "corp";
}

use-host-decl-names on;

host home-agent {
  hardware ethernet b8:27:eb:25:05:81;
  fixed-address 192.168.142.1;
}

host mobile-node {
  hardware ethernet b8:27:eb:62:3c:58;
  fixed-address 192.168.142.2;
}
```

## `bind`

### `/etc/bind/named.conf.options` (`home-agent`)

```sh
options {
  directory "/var/cache/bind";

  forwarders {
    1.1.1.1;
    1.0.0.1;
  };

  dnssec-validation auto;
  listen-on-v6 { any; };
};
```

### `/etc/bind/named.conf.local` (`home-agent`)

```sh
include "/etc/bind/zones.rfc1918";

zone "corp" {
  type master;
  file "/etc/bind/db.domain";
};

zone "142.168.192.in-addr.arpa." {
  type master;
  file "/etc/bind/db.reverse-v4";
};

zone "0.0.0.0.0.0.0.0.0.0.0.0.1.0.d.f.ip6.arpa.in-addr.arpa." {
  type master;
  file "/etc/bind/db.reverse-v6";
};
```


### `/etc/bind/db.domain` (`home-agent`)

```sh
$TTL    300
@       IN      SOA     home-agent.corp. root.home-agent.corp. (
        2       ; Serial
        604800  ; Refresh
        86400   ; Retry
        2419200 ; Expire
        604800 )        ; Negative Cache TTL
;
@       IN      NS      home-agent.corp.

home-agent      IN      A       192.168.142.1
home-agent      IN      AAAA    fd01::1
home-agent      IN      AAAA    fd01::ba27:ebff:fe25:581
mobile-node     IN      A       192.168.142.2
mobile-node     IN      AAAA    fd01::ba27:ebff:fe62:3c58
```

### `/etc/bind/db.reverse-v4` (`home-agent`)

```sh
$TTL    300
@       IN      SOA     home-agent.corp. root.home-agent.corp. (
        2       ; Serial
        604800  ; Refresh
        86400   ; Retry
        2419200 ; Expire
        604800 )        ; Negative Cache TTL
;
@       IN      NS      home-agent.corp.

1.142.168.192.in-addr.arpa.     IN      PTR     home-agent.corp.
2.142.168.192.in-addr.arpa.     IN      PTR     mobile-node.corp.
```

### `/etc/bind/db.reverse-v6` (`home-agent`)

```sh
$TTL    300
@       IN      SOA     home-agent.corp. root.home-agent.corp. (
        2       ; Serial
        604800  ; Refresh
        86400   ; Retry
        2419200 ; Expire
        604800 )        ; Negative Cache TTL
;
@       IN      NS      home-agent.corp.

1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.d.f.ip6.arpa.       IN      PTR     home-agent.corp.
1.8.5.0.5.2.e.f.f.f.b.e.7.2.a.b.0.0.0.0.0.0.0.0.0.0.0.0.1.0.d.f.ip6.arpa.       IN      PTR     home-agent.corp.
8.5.c.3.2.6.e.f.f.f.b.e.7.2.a.b.0.0.0.0.0.0.0.0.0.0.0.0.1.0.d.f.ip6.arpa.       IN      PTR     mobile-node.corp.
```

## `hostapd`

### `/etc/hostapd/hostapd.conf` (`home-agent`)

```ini
interface=wlan0
ssid=Noisette

driver=nl80211
hw_mode=g
channel=6
auth_algs=1
wpa=0
beacon_int=100
dtim_period=2
max_num_sta=255
rts_threshold=2347
fragm_threshold=2346
```

## `radvd`

### `/etc/radvd.conf` (`home-agent`)

```sh
interface wlan0 {
  AdvSendAdvert on;

  prefix fd01::/64 {
    AdvOnLink on;
    AdvRouterAddr on;
  };

  RDNSS fd01::1 {};
  DNSSL corp {};
};

interface dummy0 {
  AdvSendAdvert on;
  MaxRtrAdvInterval 3;
  MinRtrAdvInterval 1;
  AdvIntervalOpt on;
  AdvHomeAgentFlag on;
  AdvHomeAgentInfo on;
  HomeAgentLifetime 1800;
  HomeAgentPreference 10;

  prefix fd02::1/64 {
    AdvOnLink on;
    AdvAutonomous on;
    AdvRouterAddr on;
  };
};
```

## `systemd-networkd` (interface `dummy0`)

### `/etc/systemd/network/homenet.netdev` (`home-agent`)

```ini
[NetDev]
Name=dummy0
Kind=dummy
```

### `/etc/systemd/network/homenet.network` (`home-agent`)

```ini
[Match]
Name=dummy0

[Link]
# Needed for radvd to send adverts
Multicast=on

[Network]
Address=fd02::1/64
```


## `mip6d`

### `/usr/local/etc/mip6d.conf` (`mobile-node`)

```sh
NodeConfig MN;

UseMnHaIPsec disabled;
KeyMngMobCapability disabled;
OptimisticHandoff disabled;
DoRouteOptimizationCN disabled;
DoRouteOptimizationMN disabled;

UseCnBuAck enabled;

Interface "wlan0" {
  MnIfPreference 1;
}

MnHomeLink "wlan0" {
  HomeAgentAddress fd02::1;
  HomeAddress fd02::42/64;
}
```

### `/usr/local/etc/mip6d.conf` (`home-agent`)

```sh
NodeConfig HA;

Interface "dummy0";

UseMnHaIPsec disabled;
KeyMngMobCapability disabled;

DefaultBindingAclPolicy allow;
```


## Patch de `umip`

```diff
diff -ur mipv6-daemon-umip-0.4/libmissing/inet6_rth_getaddr.c mipv6-daemon-umip-0.4.new/libmissing/inet6_rth_getaddr.c
--- mipv6-daemon-umip-0.4/libmissing/inet6_rth_getaddr.c	2007-09-13 11:42:42.000000000 +0200
+++ mipv6-daemon-umip-0.4.new/libmissing/inet6_rth_getaddr.c	2019-11-23 15:20:15.329370000 +0100
@@ -3,6 +3,7 @@
 /* This is a substitute for a missing inet6_rth_getaddr(). */

 #include <netinet/in.h>
+#include <stddef.h>

 struct in6_addr *inet6_rth_getaddr(const void *bp, int index)
 {
diff -ur mipv6-daemon-umip-0.4/libmissing/inet6_rth_init.c mipv6-daemon-umip-0.4.new/libmissing/inet6_rth_init.c
--- mipv6-daemon-umip-0.4/libmissing/inet6_rth_init.c	2007-09-13 11:42:42.000000000 +0200
+++ mipv6-daemon-umip-0.4.new/libmissing/inet6_rth_init.c	2019-11-23 15:19:57.125444301 +0100
@@ -5,6 +5,7 @@
 #include <sys/socket.h>
 #include <netinet/in.h>
 #include <netinet/ip6.h>
+#include <stddef.h>

 #ifndef IPV6_RTHDR_TYPE_2
 #define IPV6_RTHDR_TYPE_2 2
diff -ur mipv6-daemon-umip-0.4/src/ha.c mipv6-daemon-umip-0.4.new/src/ha.c
--- mipv6-daemon-umip-0.4/src/ha.c	2007-09-13 11:42:42.000000000 +0200
+++ mipv6-daemon-umip-0.4.new/src/ha.c	2019-11-23 15:20:36.773282611 +0100
@@ -31,7 +31,6 @@
 #include <pthread.h>
 #include <errno.h>
 #include <net/if.h>
-#include <netinet/ip.h>
 #include <netinet/icmp6.h>
 #include <netinet/ip6mh.h>
 #include <sys/ioctl.h>
diff -ur mipv6-daemon-umip-0.4/src/tunnelctl.c mipv6-daemon-umip-0.4.new/src/tunnelctl.c
--- mipv6-daemon-umip-0.4/src/tunnelctl.c	2007-09-13 11:42:42.000000000 +0200
+++ mipv6-daemon-umip-0.4.new/src/tunnelctl.c	2019-11-23 15:20:54.829209137 +0100
@@ -39,7 +39,6 @@

 #include <net/if.h>
 #include <sys/ioctl.h>
-#include <netinet/ip.h>
 #include <linux/if_tunnel.h>
 #include <linux/ip6_tunnel.h>
 #include <pthread.h>
```
