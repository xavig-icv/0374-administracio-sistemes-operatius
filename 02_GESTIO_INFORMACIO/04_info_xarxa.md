## 04. Informació de la Xarxa

Aquest fitxer respon la pregunta "**com està connectat el servidor?**": interfícies, adreces, rutes, ports oberts i DNS. Els exemples corresponen al servidor `srv0374` de l'[escenari del laboratori](../00_RECURSOS/01_escenari_laboratori.md): `enp0s3` connectada a la xarxa del centre per DHCP i `enp0s8` a la xarxa interna amb la IP fixa `192.168.1.1`.

| Ordre        | Informació                                         |
| ------------ | -------------------------------------------------- |
| `ip address` | Interfícies i adreces IP                           |
| `ip route`   | Taula d'encaminament (rutes)                       |
| `ip neigh`   | Taula ARP (equips veïns de la xarxa local)         |
| `ss`         | Ports en escolta i connexions                      |
| `ethtool`    | Característiques de la targeta (velocitat, enllaç, controlador) |
| `resolvectl` | Configuració DNS                                   |
| `netplan`    | Configuració permanent de la xarxa                 |

Les ordres antigues del paquet `net-tools` ja no s'instal·len per defecte. Les equivalències són:

| Antiga (`net-tools`) | Actual (`iproute2`) |
| -------------------- | ------------------- |
| `ifconfig`           | `ip address`        |
| `route -n`           | `ip route`          |
| `arp -n`             | `ip neigh`          |
| `netstat -tulnp`     | `ss -tulnp`         |

### Interfícies i Adreces: `ip address`

```bash
ip a                               # forma abreujada de "ip address"
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:3c:1a:2b brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.37/24 metric 100 brd 10.0.1.255 scope global dynamic enp0s3
       valid_lft 85872sec preferred_lft 85872sec
    inet6 fe80::a00:27ff:fe3c:1a2b/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:9d:4c:5e brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 brd 192.168.1.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe9d:4c5e/64 scope link
       valid_lft forever preferred_lft forever
```

| Element                      | Significat                                                                 |
| ---------------------------- | -------------------------------------------------------------------------- |
| `lo`                         | Interfície de *loopback* (el mateix equip, `127.0.0.1`)                    |
| `UP`                         | L'administrador ha activat la interfície                                   |
| `LOWER_UP`                   | Hi ha enllaç físic (cable connectat). Si no hi és, apareix `NO-CARRIER`    |
| `mtu 1500`                   | Mida màxima de paquet en bytes                                             |
| `link/ether 08:00:27:...`    | Adreça MAC. `08:00:27` és el prefix de les targetes virtuals de VirtualBox |
| `inet 10.0.1.37/24`          | Adreça IPv4 i màscara (`/24` = `255.255.255.0`)                            |
| `dynamic` + `valid_lft`      | Adreça obtinguda per **DHCP** i temps que queda de concessió. Una IP fixa no té `dynamic` i dura `forever` |
| `scope global` / `scope link`| Adreça vàlida per a qualsevol xarxa / només per a la xarxa local          |
| `inet6 fe80::...`            | Adreça IPv6 d'enllaç local, generada automàticament a partir de la MAC     |

**Vista resumida** (una línia per interfície):

```bash
ip -br a                           # adreces
ip -br link                        # estat i adreça MAC
ip -4 a show enp0s8                # només l'IPv4 d'una interfície
```

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.1.37/24 metric 100 fe80::a00:27ff:fe3c:1a2b/64
enp0s8           UP             192.168.1.1/24 fe80::a00:27ff:fe9d:4c5e/64
```

**Noms de les interfícies:** `lo` (*loopback*), `enpXsY` (Ethernet al bus PCI X, ranura Y), `wlp...` (Wi-Fi), `docker0`, `virbr0` o `br0` (interfícies virtuals i ponts).

### Rutes: `ip route`

```bash
ip route
```

```
default via 10.0.1.1 dev enp0s3 proto dhcp src 10.0.1.37 metric 100
10.0.1.0/24 dev enp0s3 proto kernel scope link src 10.0.1.37 metric 100
10.0.1.1 dev enp0s3 proto dhcp scope link src 10.0.1.37 metric 100
192.168.1.0/24 dev enp0s8 proto kernel scope link src 192.168.1.1
```

| Element                | Significat                                                                      |
| ---------------------- | ------------------------------------------------------------------------------- |
| `default via 10.0.1.1` | Porta d'enllaç (*gateway*): on s'envien els paquets cap a xarxes desconegudes   |
| `dev enp0s3`           | Interfície de sortida                                                           |
| `proto`                | Origen de la ruta: `kernel` (automàtica en assignar una IP), `dhcp`, `static` (configurada a netplan) |
| `scope link`           | Xarxa connectada directament: no cal passar per cap gateway                     |
| `src`                  | Adreça d'origen que es fa servir per aquesta ruta                               |
| `metric`               | Prioritat: si hi ha dues rutes cap al mateix destí, guanya la mètrica més baixa |

**Per on sortirà un paquet?** `ip route get` indica la ruta que s'aplicarà a un destí concret:

```bash
ip route get 8.8.8.8
ip route get 192.168.1.50
```

```
8.8.8.8 via 10.0.1.1 dev enp0s3 src 10.0.1.37 uid 1000
    cache
192.168.1.50 dev enp0s8 src 192.168.1.1 uid 1000
    cache
```

Internet es troba a través del gateway del centre (`via 10.0.1.1`) i el client de la xarxa interna, directament per `enp0s8` (sense `via`).

**Taula ARP:** `ip neigh` mostra les adreces MAC dels equips de la xarxa local amb qui s'ha comunicat el servidor (`REACHABLE` = recent, `STALE` = fa estona, `FAILED` = no respon).

### Ports i Connexions: `ss`

| Opció                | Funció                                                        |
| -------------------- | ------------------------------------------------------------- |
| `-t` / `-u`          | Sockets TCP / UDP                                             |
| `-l`                 | Només els que estan en escolta (`LISTEN`)                     |
| `-a`                 | Tots: en escolta i connexions                                 |
| `-n`                 | Números de port (sense traduir-los a noms de servei)          |
| `-p`                 | Procés que té obert el socket (cal `sudo` per veure'ls tots)  |
| `-4` / `-6`          | Només IPv4 / IPv6                                             |
| `-s`                 | Resum estadístic                                              |
| `state established`  | Filtra per estat                                              |

**Quins serveis escolten?**

```bash
sudo ss -tulnp
```

```
Netid State  Recv-Q Send-Q      Local Address:Port  Peer Address:Port Process
udp   UNCONN 0      0           127.0.0.53%lo:53         0.0.0.0:*     users:(("systemd-resolve",pid=612,fd=14))
udp   UNCONN 0      0        10.0.1.37%enp0s3:68         0.0.0.0:*     users:(("systemd-network",pid=590,fd=22))
tcp   LISTEN 0      4096        127.0.0.53%lo:53         0.0.0.0:*     users:(("systemd-resolve",pid=612,fd=15))
tcp   LISTEN 0      2048              0.0.0.0:389        0.0.0.0:*     users:(("slapd",pid=905,fd=8))
tcp   LISTEN 0      4096                    *:22               *:*     users:(("sshd",pid=1034,fd=3),("systemd",pid=1,fd=86))
tcp   LISTEN 0      2048                 [::]:389           [::]:*     users:(("slapd",pid=905,fd=9))
```

| Adreça local              | Significat                                                                 |
| ------------------------- | -------------------------------------------------------------------------- |
| `0.0.0.0:389` / `[::]:389`| Escolta a totes les adreces IPv4 / IPv6: accessible des de la xarxa (LDAP del Tema 1) |
| `*:22`                    | Totes les adreces IPv4 i IPv6 (SSH)                                         |
| `127.0.0.53%lo:53`        | Només des del mateix equip: el resolutor DNS local de `systemd-resolved`   |
| `10.0.1.37%enp0s3:68`     | Client DHCP de la interfície `enp0s3`                                      |

La columna `Process` indica el programa, el seu PID i el descriptor de fitxer. A Ubuntu 24.04, el port 22 apareix també amb `systemd` (PID 1): l'SSH s'activa per socket (`ssh.socket`) i és `systemd` qui obre el port i arrenca `sshd` quan cal.

`Nota:` Si al vostre servidor ja heu fet les pràctiques del mòdul 0375, també hi veureu `named` (DNS, port 53) i `dhcpd` (port 67).

**Ports habituals:**

| Port      | Servei      | Port        | Servei          |
| --------- | ----------- | ----------- | --------------- |
| 22        | SSH, SFTP   | 80 / 443    | HTTP / HTTPS    |
| 53        | DNS         | 389 / 636   | LDAP / LDAPS    |
| 67 / 68   | DHCP        | 25 / 587    | SMTP            |
| 21        | FTP         | 631         | IPP (CUPS, Tema 5) |

**Quines connexions hi ha establertes?**

```bash
ss -tn state established
ss -tn state established '( sport = :22 )'     # només les connexions SSH entrants
```

```
Recv-Q Send-Q    Local Address:Port    Peer Address:Port Process
0      0             10.0.1.37:22         10.0.1.20:52344
0      0             10.0.1.37:22         10.0.1.21:61022
```

Cada línia és una connexió: `Local` és el servidor i `Peer`, l'equip remot (aquí, dues sessions SSH de dos equips diferents).

### Característiques de la Targeta: `ethtool`

```bash
sudo ethtool enp0s3 | grep -E "Speed|Duplex|Auto-negotiation|Link detected"
```

```
	Speed: 1000Mb/s
	Duplex: Full
	Auto-negotiation: on
	Link detected: yes
```

| Ordre                      | Informació                                                       |
| -------------------------- | ---------------------------------------------------------------- |
| `ethtool enp0s3`           | Velocitat, dúplex, autonegociació i enllaç                       |
| `ethtool -i enp0s3`        | Controlador (`driver: e1000`), versió i posició al bus (`bus-info`) |
| `ethtool -S enp0s3`        | Estadístiques de la targeta: paquets, errors, col·lisions        |

`Duplex: Full` vol dir que la targeta envia i rep alhora. Un enllaç a `100Mb/s` o en `Half` en un servidor sol indicar un cable o un port del commutador en mal estat. En una VM, la velocitat és simulada per VirtualBox.

### DNS: `resolvectl`

A Ubuntu, les consultes DNS les gestiona `systemd-resolved`. El fitxer `/etc/resolv.conf` apunta al resolutor local `127.0.0.53`, que reenvia les consultes als servidors DNS de cada interfície.

```bash
resolvectl status
```

```
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub

Link 2 (enp0s3)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.0.1.1
       DNS Servers: 10.0.1.1 8.8.8.8

Link 3 (enp0s8)
    Current Scopes: none
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
```

| Element               | Significat                                                                 |
| --------------------- | -------------------------------------------------------------------------- |
| `Global`              | Configuració que s'aplica a totes les interfícies                          |
| `Link 2 (enp0s3)`     | Configuració DNS d'aquesta interfície (rebuda per DHCP o definida a netplan) |
| `Current DNS Server`  | Servidor que s'està utilitzant ara                                         |
| `DNS Servers`         | Llista de servidors configurats                                            |
| `Current Scopes: none`| La interfície no té DNS configurat                                         |

```bash
resolvectl dns                     # servidors DNS de cada interfície
resolvectl query www.google.com    # resol un nom i indica per quina interfície i quant ha trigat
resolvectl flush-caches            # buida la memòria cau DNS
```

### Configuració Permanent: Netplan

Les ordres `ip` canvien la configuració **només fins al proper reinici**. La configuració permanent d'Ubuntu es fa amb **netplan**: fitxers YAML a `/etc/netplan/` que netplan tradueix per a `systemd-networkd` (servidors) o `NetworkManager` (escriptori).

```bash
ls /etc/netplan/
cat /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses: [192.168.1.1/24]
```

`dhcp4: true` indica que la interfície rep la IP per DHCP. Si en lloc d'això hi ha `addresses:`, la IP és fixa. Exemple d'`enp0s3` amb IP fixa:

```yaml
    enp0s3:
      dhcp4: false
      addresses: [10.0.1.37/24]
      routes:
        - to: default
          via: 10.0.1.1
      nameservers:
        addresses: [10.0.1.1, 8.8.8.8]
```

| Ordre                 | Funció                                                                          |
| --------------------- | ------------------------------------------------------------------------------- |
| `sudo netplan try`    | Aplica els canvis i els desfà al cap de 120 s si no es confirmen (ideal per SSH) |
| `sudo netplan apply`  | Aplica els canvis definitivament                                                |
| `netplan get`         | Mostra la configuració combinada de tots els fitxers                            |
| `netplan status`      | Estat de cada interfície (adreces, rutes, DNS)                                  |

**Regles del YAML:** el sagnat es fa **amb espais** (mai amb tabuladors), sempre amb el mateix nombre d'espais per nivell. Els fitxers han de tenir permisos `600` (`sudo chmod 600 /etc/netplan/*.yaml`); si no, netplan avisa que són massa oberts.

`Nota:` Si el fitxer és `50-cloud-init.yaml`, `cloud-init` el pot regenerar en arrencar i desfer els canvis. Per evitar-ho, creeu `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` amb la línia `network: {config: disabled}`.

**`networkctl`** mostra l'estat de les interfícies segons `systemd-networkd`:

```
IDX LINK   TYPE     OPERATIONAL SETUP
  1 lo     loopback carrier     unmanaged
  2 enp0s3 ether    routable    configured
  3 enp0s8 ether    routable    configured
```

`routable` vol dir que la interfície té adreça i rutes; `configured`, que la configuració de netplan s'ha aplicat correctament.

### Resolució de Problemes

| Problema                                             | Causa probable                                             | Solució                                                     |
| ---------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------- |
| Una interfície apareix `DOWN`                        | No està configurada a netplan                              | Afegir-la al YAML i `sudo netplan apply`                    |
| Apareix `NO-CARRIER` i `state DOWN`                  | No hi ha enllaç (a VirtualBox: cable desconnectat)         | Configuració de la VM → Xarxa → *Cable connectat*           |
| `enp0s8` no té IP tot i estar al YAML                | Adaptador 2 de VirtualBox desactivat o en una altra xarxa  | Activar-lo com a *Xarxa interna* `intnet`                   |
| Funciona el `ping` a una IP, però no a un nom        | Problema de DNS                                            | `resolvectl status` i `resolvectl query nom`                |
| `netplan apply` mostra `Invalid YAML`                | Tabuladors o sagnat incorrecte                             | Revisar el sagnat amb espais                                |
| Els canvis es perden en reiniciar                    | Configuració feta amb `ip` o regenerada per `cloud-init`   | Configurar-ho a netplan i desactivar la xarxa de `cloud-init` |
| `ss -p` no mostra el procés d'alguns ports           | Són processos d'altres usuaris                             | `sudo ss -tulnp`                                            |
