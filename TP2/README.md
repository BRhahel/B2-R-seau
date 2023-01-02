# TP3 : On va router des trucs

Au menu de ce TP, on va revoir un peu ARP et IP histoire de **se mettre en jambes dans un environnement avec des VMs**.

Puis on mettra en place **un routage simple, pour permettre à deux LANs de communiquer**.

![Reboot the router](./pics/reboot.jpeg)

## Sommaire

- [TP3 : On va router des trucs](#tp3--on-va-router-des-trucs)
  - [Sommaire](#sommaire)
  - [0. Prérequis](#0-prérequis)
  - [I. ARP](#i-arp)
    - [1. Echange ARP](#1-echange-arp)
    - [2. Analyse de trames](#2-analyse-de-trames)
  - [II. Routage](#ii-routage)
    - [1. Mise en place du routage](#1-mise-en-place-du-routage)


## 0. Prérequis

➜ Pour ce TP, on va se servir de VMs Rocky Linux. 1Go RAM c'est large large. Vous pouvez redescendre la mémoire vidéo aussi.  

➜ Vous aurez besoin de deux réseaux host-only dans VirtualBox :

- un premier réseau `10.3.1.0/24`
- le second `10.3.2.0/24`
- **vous devrez désactiver le DHCP de votre hyperviseur (VirtualBox) et définir les IPs de vos VMs de façon statique**

➜ Quelques paquets seront souvent nécessaires dans les TPs, il peut être bon de les installer dans la VM que vous clonez :

- de quoi avoir les commandes :
  - `dig`
  - `tcpdump`
  - `nmap`
  - `nc`
  - `python3`
  - `vim` peut être une bonne idée

➜ Les firewalls de vos VMs doivent **toujours** être actifs (et donc correctement configurés).

➜ **Si vous voyez le p'tit pote 🦈 c'est qu'il y a un PCAP à produire et à mettre dans votre dépôt git de rendu.**

## I. ARP

Première partie simple, on va avoir besoin de 2 VMs.

| Machine  | `10.3.1.0/24` |
|----------|---------------|
| `john`   | `10.3.1.11`   |
| `marcel` | `10.3.1.12`   |

```schema
   john               marcel
  ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     │
  └─────┘    └───┘    └─────┘
```

> Référez-vous au [mémo Réseau Rocky](../memo/rocky_network.md) pour connaître les commandes nécessaire à la réalisation de cette partie.
### 1. Echange ARP

🌞**Générer des requêtes ARP**

- effectuer un `ping` d'une machine à l'autre
- observer les tables ARP des deux machines
- repérer l'adresse MAC de `john` dans la table ARP de `marcel` et vice-versa
- prouvez que l'info est correcte (que l'adresse MAC que vous voyez dans la table est bien celle de la machine correspondante)
  - une commande pour voir la MAC de `marcel` dans la table ARP de `john`
  - et une commande pour afficher la MAC de `marcel`, depuis `marcel`


ping :
```bash
john:
[user@localhost ~]$ ping 10.3.1.12
PING 10.3.1.12 (10.3.1.12) 56(84) bytes of data.
64 bytes from 10.3.1.12: icmp_seq=1 ttl=64 time=1.15 ms
64 bytes from 10.3.1.12: icmp_seq=2 ttl=64 time=1.09 ms
marcel:
[user@localhost ~]$ ping 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=64 time=1.52 ms
64 bytes from 10.3.1.11: icmp_seq=2 ttl=64 time=0.520 ms
```

table ARP :
```bash
john:
[user@localhost ~]$ ip neigh show
10.3.1.12 dev enp0s3 lladdr 08:00:27:be:b7:0e STALE
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:06 DELAY
marcel:
[user@localhost ~]$ ip neigh show
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:06 DELAY
10.3.1.11 dev enp0s3 lladdr 08:00:27:34:26:7e STALE
```

Adresse MAC de John : 08:00:27:34:26:7e
Adresse MAC de Marcel : 0a:00:27:00:00:06

Preuve :
```
john:
[user@localhost ~]$ ip neigh show
10.3.1.12 dev enp0s3 lladdr **08:00:27:be:b7:0e** STALE
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:06 DELAY
marcel:
[user@localhost ~]$ nmcli device show
GENERAL.DEVICE:                         enp0s3
GENERAL.TYPE:                           ethernet
GENERAL.HWADDR:                         **08:00:27:BE:B7:0E**
```

### 2. Analyse de trames

🌞**Analyse de trames**

🦈 [Capture réseau `tp2_arp.pcapng` qui contient un ARP request et un ARP reply](tp2_arp.pcapng)

- utilisez la commande `tcpdump` pour réaliser une capture de trame
- videz vos tables ARP, sur les deux machines, puis effectuez un `ping`

```
john:
[user@localhost ~]$ sudo ip -s -s neigh flush all
10.3.1.1 dev enp0s3 lladdr 0a:00:27:00:00:07 ref 1 used 11/0/11 probes 1 REACHABLE
*** Round 1, deleting 1 entries ***
10.3.1.1 dev enp0s3  ref 1 used 0/60/0 probes 4 INCOMPLETE
*** Round 2, deleting 1 entries ***
10.3.1.1 dev enp0s3  ref 1 used 0/0/0 probes 4 INCOMPLETE
*** Round 3, deleting 1 entries ***
*** Flush is complete after 3 rounds ***
[user@localhost ~]$ sudo tcpdump -i enp0s8 -c 10 -w ping.pcapng not port 22
marcel:
[user@localhost ~]$ ping 10.3.1.11
```



> **Si vous ne savez pas comment récupérer votre fichier `.pcapng`** sur votre hôte afin de l'ouvrir dans Wireshark, et me le livrer en rendu, demandez-moi.
## II. Routage

Vous aurez besoin de 3 VMs pour cette partie. **Réutilisez les deux VMs précédentes.**

| Machine  | `10.3.1.0/24` | `10.3.2.0/24` |
|----------|---------------|---------------|
| `router` | `10.3.1.254`  | `10.3.2.254`  |
| `john`   | `10.3.1.11`   | no            |
| `marcel` | no            | `10.3.2.12`   |

> Je les appelés `marcel` et `john` PASKON EN A MAR des noms nuls en réseau 🌻
```schema
   john                router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └───┘    └─────┘    └───┘    └─────┘
```

### 1. Mise en place du routage

🌞**Activer le routage sur le noeud `router`**

> Cette étape est nécessaire car Rocky Linux c'est pas un OS dédié au routage par défaut. Ce n'est bien évidemment une opération qui n'est pas nécessaire sur un équipement routeur dédié comme du matériel Cisco.
```bash
router:
[user@localhost ~]$ sudo firewall-cmd --add-masquerade --zone=public --permanent
success
```

🌞**Ajouter les routes statiques nécessaires pour que `john` et `marcel` puissent se `ping`**

- il faut ajouter une seule route des deux côtés
- une fois les routes en place, vérifiez avec un `ping` que les deux machines peuvent se joindre

```bash
john:
[user@localhost ~]$ sudo ip route add 10.3.2.0/24 via 10.3.1.254 dev enp0s3
marcel:
[user@localhost ~]$ sudo ip route add 10.3.1.0/24 via 10.3.2.254 dev enp0s3
```

preuves (marcel):
```bash
[user@localhost ~]$ ping 10.3.1.11
ping: connect: Network is unreachable
[user@localhost ~]$ sudo ip route add 10.3.1.0/24 via 10.3.2.254 dev enp0s3
[user@localhost ~]$ ping 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=63 time=2.18 ms
64 bytes from 10.3.1.11: icmp_seq=2 ttl=63 time=2.10 ms
```

### 2. Analyse de trames

**🌞 Analyse des échanges ARP**

| ordre | type trame  | IP source  | MAC source                 | IP destination | MAC destination            |
|-------|-------------|------------|----------------------------|----------------|----------------------------|
| 1     | Requête ARP | 10.3.1.11  | `john` `08:00:27:a0:74:35` | 10.3.1.254     | Broadcast `FF:FF:FF:FF:FF` |
| 2     | Réponse ARP | 10.3.1.254 | Broadcast `FF:FF:FF:FF:FF` | 10.3.1.11      | `john` `08:00:27:a0:74:35` |


🦈 [Capture réseau `tp2_routage_marcel.pcap`](tp2_routage_marcel.pcap)

---

### 3. Accès internet

**🌞 Donnez un accès internet à vos machines**

```
IN PROGRESS
```

**🌞 Analyse de trames**

| ordre | type trame | IP source          | MAC source              | IP destination | MAC destination |     |
|-------|------------|--------------------|-------------------------|----------------|-----------------|-----|
| 1     | ping       | `john` `10.3.1.12` | `john` `AA:BB:CC:DD:EE` | `8.8.8.8`      | ?               |     |
| 2     | pong       | ...                | ...                     | ...            | ...             | ... |

🦈 [Capture réseau `tp2_routage_internet.pcap`](tp2_routage_internet.pcap)