# 02. Informació de Xarxa

La configuració de xarxa és un dels aspectes més importants d'un servidor. Sense connectivitat en xarxa, el servidor no pot comunicar-se amb clients, altres servidors o accedir a Internet. En aquest bloc tractarem les comandes essencials per obtenir informació sobre la **configuració estàtica de la xarxa**: com les interfícies, les adreces IP, les rutes, les connexions actives i la configuració amb Netplan.

**Objectiu**: Conèixer els procediments de recollida d'informació sobre la configuració de xarxa i entendre com està connectat el servidor.

## Comandes d'aquest tema

1. `ip addr` (`ip a`) - Interfícies de xarxa, IPs i MACs
2. `ip route` - Taula de rutes del sistema
3. `ss` - Connexions de xarxa actives (sockets)
4. `ethtool` - Característiques físiques de les targetes de xarxa
5. `netplan` - Configuració permanent de xarxa (Ubuntu Server)
6. `networkctl` - Estat de systemd-networkd
7. `resolvectl` - Configuració de DNS

## 1. ip addr - Interfícies de xarxa

### Descripció

La comanda `ip addr` (o `ip a` abreujat) mostra informació sobre totes les **interfícies de xarxa** del sistema: noms, adreces IP, adreces MAC, estat (up/down) i més. És la comanda més important per començar qualsevol diagnòstic de xarxa.

**Nota:** `ip` substitueix les comandes antigues `ifconfig`, `route`, `arp`, etc.

### Sintaxi

```bash
ip addr [show] [DISPOSITIU]
# O abreujat:
ip a
```

### Opcions principals

| Opció                        | Descripció                             |
| ---------------------------- | -------------------------------------- |
| `show`                       | Mostra informació (per defecte)        |
| `show DISPOSITIU`            | Mostra només una interfície específica |
| `add IP/MASK dev DISPOSITIU` | Afegeix una IP (temporal)              |
| `del IP/MASK dev DISPOSITIU` | Elimina una IP                         |

### Exemples pràctics

**Exemple 1: Veure totes les interfícies**

```bash
ip addr
# O abreujat:
ip a
```

**Sortida de nostre servidor 0374 (Ubuntu Server):**

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:7d:e2:4f brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.10/24 brd 10.0.1.255 scope global enp0s8
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:3a:b5:c1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 brd 192.168.1.255 scope global dynamic enp0s3
       valid_lft 86395sec preferred_lft 86395sec
    inet6 fe80::a00:27ff:fe3a:b5c1/64 scope link
       valid_lft forever preferred_lft forever
```

**Exemple 2: Veure només una interfície específica**

```bash
ip addr show enp0s3
# O abreujat:
ip a show enp0s3
```

**Exemple 3: Filtrar les interfícies (sense detalls)**

```bash
ip -br addr
```

**Sortida:**

```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             192.168.1.100/24 fe80::a00:27ff:fe3a:b5c1/64
enp0s8           UP             10.0.0.10/24
```

### Interpretació de la sortida (`ip a`)

**Línia 1: Informació de la interfície**

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
```

- **2:** Índex de la interfície
- **enp0s3:** Nom de la interfície (en = Ethernet, p0s3 = bus PCI 0, slot 3)
- **Flags:** `<BROADCAST,MULTICAST,UP,LOWER_UP>`
  - **UP:** Interfície activada
  - **LOWER_UP:** Cable connectat (físicament activa)
  - **BROADCAST:** Suporta broadcast
  - **MULTICAST:** Suporta multicast
- **mtu 1500:** Maximum Transmission Unit (mida màxima del paquet)
- **state UP:** Estat operacional

**Línia 2: Adreça física (MAC)**

```
link/ether 08:00:27:3a:b5:c1 brd ff:ff:ff:ff:ff:ff
```

- **link/ether:** Adreça MAC de la targeta
- **08:00:27:3a:b5:c1:** Adreça MAC única
- **brd ff:ff:ff:ff:ff:ff:** Adreça de broadcast

**Línia 3: Adreça IPv4**

```
inet 10.0.1.10/24 brd 10.0.1.255 scope global dynamic enp0s3
```

- **inet:** IPv4
- **10.0.1.10/24:** IP i màscara (255.255.255.0)
- **brd 10.0.1.255:** Adreça de broadcast
- **scope global:** Accessible des de fora
- **dynamic:** Assignada per DHCP (si fos `static` seria un assignació manual)

**Línia 4: Adreça IPv6**

```
inet6 fe80::a00:27ff:fe3a:b5c1/64 scope link
```

- **inet6:** IPv6
- **fe80::** Adreça link-local (local de la xarxa)
- **scope link:** Només accessible a la xarxa local

**Noms d'interfícies modernes:**

- **lo:** Loopback (interfície virtual interna)
- **enp0s3:** Ethernet (en) PCI (p) bus 0 slot 3
- **ens33:** Ethernet (en) slot 33
- **eth0, eth1:** Nomenclatura antiga (pot aparèixer en sistemes antics)
- **wlp3s0:** Wireless (wl) PCI (p) bus 3 slot 0

## 2. `ip route` - Taula d'encaminament (rutes)

### Descripció

La comanda `ip route` mostra la **taula d'encaminament** del sistema. Són els diferents camins que poden fer els paquets de xarxa. El que servidor decideix cap a on enviar-los tot guiant-se d'aquesta taula de rutes. Les taules d'encaminament són essencials per entendre la connectivitat i diagnosticar problemes d'encaminament als servidors.

### Sintaxi

```bash
ip route [show]
# O abreujat:
ip r
```

### Opcions principals

| Opció    | Descripció                                        |
| -------- | ------------------------------------------------- |
| `show`   | Mostra la taula de rutes (per defecte)            |
| `get IP` | Mostra quina ruta s'utilitza per arribar a una IP |
| `add`    | Afegeix una ruta (temporal)                       |
| `del`    | Elimina una ruta                                  |

### Exemples pràctics

**Exemple 1: Veure totes les rutes**

```bash
ip route
# O:
ip r
```

**Sortida típica:**

```
default via 10.0.1.1 dev enp0s3 proto dhcp src 10.0.1.10 metric 100
10.0.1.0/24 dev enp0s3 proto kernel scope link src 10.0.1.10
192.168.1.0/24 dev enp0s8 proto kernel scope link src 192.168.1.1
```

**Exemple 2: Comprovar quina ruta s'utilitza per una IP específica**

```bash
ip route get 8.8.8.8
```

**Sortida:**

```
8.8.8.8 via 10.0.1.1 dev enp0s3 src 10.0.1.10 uid 1000
    cache
```

**Exemple 3: Veure només la ruta per defecte**

```bash
ip route | grep default
```

### Interpretació de la sortida

**Ruta per defecte (gateway):**

```
default via 10.0.1.1 dev enp0s3 proto dhcp src 10.0.1.10 metric 100
```

- **default:** Ruta per defecte (0.0.0.0/0) - per a tot el tràfic que no disposa d'una ruta específica
- **via 10.0.1.1:** Gateway (router, encaminador, porta d'enllaç) per sortir a internet
- **dev enp0s3:** Interfície de sortida
- **proto dhcp:** Ruta configurada per DHCP
- **src 10.0.1.10:** IP d'origen dels paquets
- **metric 100:** Prioritat (com més baix sigui --> la prioritat és més alta)

**Rutes de xarxa local:**

```
192.168.1.0/24 dev enp0s8 proto kernel scope link src 192.168.1.1
```

- **192.168.1.0/24:** Xarxa de destinació (tota la xarxa 192.168.1.0/24)
- **dev enp0s8:** Interfície per accedir a aquesta xarxa
- **proto kernel:** Ruta afegida automàticament pel kernel
- **scope link:** Xarxa directament connectada
- **src 192.168.1.1:** IP d'origen

**Com funciona l'encaminament:**

1. El sistema rep un paquet destinat a `192.168.1.50`
2. Busca a la taula de rutes: hi ha una ruta per `192.168.1.0/24`
3. Envia el paquet directament per `enp0s8` (xarxa local)
4. Si fos per `8.8.8.8`, no hi ha ruta específica → utilitzaria `default` via `10.0.1.1`

## 3. `ss` - Connexions actives

### Descripció

La comanda `ss` (socket statistics) mostra informació sobre els **sockets de xarxa**: connexions TCP/UDP actives, ports oberts, processos que escolten, etc.

### Sintaxi

```bash
ss [OPCIONS]
```

### Opcions principals

| Opció | Descripció                                       |
| ----- | ------------------------------------------------ |
| `-t`  | Mostra sockets TCP                               |
| `-u`  | Mostra sockets UDP                               |
| `-l`  | Mostra només sockets escoltant (listening)       |
| `-n`  | Mostra adreces numèriques (no resol noms)        |
| `-p`  | Mostra el procés que utilitza el socket          |
| `-a`  | Mostra tots els sockets (listening + establerts) |
| `-4`  | Només IPv4                                       |
| `-6`  | Només IPv6                                       |

### Exemples pràctics

**Exemple 1: Veure connexions TCP actives**

```bash
ss -t
```

**Sortida:**

```
State    Recv-Q Send-Q Local Address:Port    Peer Address:Port   Process
ESTAB    0      0      192.168.1.100:ssh     192.168.1.50:54321
ESTAB    0      0      192.168.1.100:http    203.0.113.45:44123
```

**Exemple 2: Veure ports TCP escoltant (listening)**

```bash
ss -tln
```

**Sortida:**

```
State    Recv-Q Send-Q Local Address:Port    Peer Address:Port
LISTEN   0      128    0.0.0.0:22            0.0.0.0:*
LISTEN   0      511    0.0.0.0:80            0.0.0.0:*
LISTEN   0      128    [::]:22               [::]:*
```

**Exemple 3: Veure ports oberts amb el procés responsable**

```bash
sudo ss -pltn
```

**Sortida:**

```
State   Recv-Q Send-Q Local Address:Port   Peer Address:Port Process
LISTEN  0      128    0.0.0.0:22           0.0.0.0:*     users:(("sshd",pid=850,fd=3))
LISTEN  0      511    0.0.0.0:80           0.0.0.0:*     users:(("apache2",pid=1234,fd=4))
```

**Exemple 4: Filtrar per port específic**

```bash
ss -tn sport = :22
```

### Interpretació de la sortida

**Columnes principals:**

- **State:** Estat de la connexió
  - **LISTEN:** Port escoltant (esperant connexions)
  - **ESTAB:** Connexió establerta
  - **TIME-WAIT:** Connexió tancada, esperant confirmació final
  - **CLOSE-WAIT:** Connexió tancada per l'altra part
- **Recv-Q:** Dades a la cua de recepció (hauria de ser 0)
- **Send-Q:** Dades a la cua d'enviament (hauria de ser 0)
- **Local Address:Port:** IP i port local
- **Peer Address:Port:** IP i port remot
- **Process:** Procés que gestiona la connexió (amb `-p`)

**Estats TCP importants:**

- **LISTEN:** Servei esperant connexions (servidor)
- **ESTAB:** Connexió activa i funcional
- **TIME-WAIT:** Normal després de tancar una connexió
- **CLOSE-WAIT:** Possible problema si n'hi ha molts

**Exemple d'ús pràctic:**

```bash
# Comprovar si SSH està escoltant
sudo ss -tlnp | grep :22

# Veure totes les connexions SSH actives
ss -tn | grep :22

# Comptabilitzar les connexions actives
ss -tan | grep ESTAB | wc -l
```

## 4. `ethtool` - Característiques de les targetes

### Descripció

`ethtool` proporciona informació i configuració de **baix nivell** de les targetes de xarxa: velocitat, duplex, driver, estadístiques d'errors, capacitats del hardware, etc.

### Sintaxi

```bash
ethtool [OPCIONS] INTERFÍCIE
```

### Opcions principals

| Opció           | Descripció                      |
| --------------- | ------------------------------- |
| (sense opcions) | Informació bàsica de la targeta |
| `-i`            | Informació del driver           |
| `-S`            | Estadístiques de la targeta     |
| `-k`            | Característiques (offloading)   |
| `-g`            | Ring buffer settings            |

### Exemples pràctics

**Exemple 1: Informació bàsica de la targeta**

```bash
sudo ethtool enp0s3
```

**Sortida:**

```
Settings for enp0s3:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Supported pause frame use: No
        Supports auto-negotiation: Yes
        Supported FEC modes: Not reported
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Speed: 1000Mb/s
        Duplex: Full
        Port: Twisted Pair
        Auto-negotiation: on
        Link detected: yes
```

**Exemple 2: Informació del driver**

```bash
sudo ethtool -i enp0s3
```

**Sortida:**

```
driver: e1000
version: 7.3.21-k8-NAPI
firmware-version:
expansion-rom-version:
bus-info: 0000:00:03.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no
```

**Exemple 3: Estadístiques de la targeta**

```bash
sudo ethtool -S enp0s3 | head -n 20
```

### Interpretació de la sortida

**Camps més importants:**

**Speed:** Velocitat de la connexió

- `1000Mb/s` = 1 Gbps (Gigabit Ethernet)
- `100Mb/s` = Fast Ethernet
- `10Mb/s` = Ethernet clàssic

**Duplex:**

- **Full:** Transmissió i recepció simultània (òptim)
- **Half:** Només una direcció cada vegada (lent, evitar)

**Link detected:**

- **yes:** Cable connectat i enllaç actiu
- **no:** Cable desconnectat o problema físic

**Auto-negotiation:**

- **on:** La targeta negocia automàticament velocitat i duplex
- **off:** Configuració manual (pot causar problemes)

**Driver:** Nom del controlador del kernel

- `e1000` - Intel (VirtualBox/VMware)
- `virtio_net` - Targeta virtual paravirtualitzada
- `r8169` - Realtek
- `igb`, `ixgbe` - Intel de gamma alta

**Ús pràctic:**

```bash
# Verificar si el cable està connectat
sudo ethtool enp0s3 | grep "Link detected"

# Comprovar la velocitat actual
sudo ethtool enp0s3 | grep Speed

# Veure el driver utilitzat
sudo ethtool -i enp0s3 | grep driver
```

## 5. Netplan - Configuració permanent

### Descripció

**Netplan** és l'eina de configuració de xarxa d'Ubuntu Server. Utilitza fitxers **YAML** per definir la configuració i genera automàticament la configuració per als serveis (systemd-networkd o NetworkManager).

**Important:** Els canvis a Netplan són **permanents**, a diferència d'un canvi fet amb `ip addr` que és temporal.

### Ubicació dels fitxers

```bash
/etc/netplan/*.yaml
```

Fitxers comuns:

- `00-installer-config.yaml` (creat per l'instal·lador)
- `01-netcfg.yaml` (configuració personalitzada)
- `50-cloud-init.yaml` (si s'utilitza cloud-init)

### Sintaxi bàsica del YAML

```yaml
network:
  version: 2
  renderer: networkd # o NetworkManager
  ethernets:
    enp0s3:
      dhcp4: yes
```

### Subcomandes principals

| Comanda                   | Descripció                         |
| ------------------------- | ---------------------------------- |
| `cat /etc/netplan/*.yaml` | Veure la configuració              |
| `sudo netplan apply`      | Aplicar els canvis                 |
| `sudo netplan try`        | Provar (reverteix si no confirmes) |
| `sudo netplan get`        | Veure configuració activa          |
| `sudo netplan generate`   | Generar configuracions de backend  |

### Exemples pràctics

**Exemple 1: Veure la configuració actual**

```bash
cat /etc/netplan/*.yaml
```

**Sortida típica:**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: true
```

**Exemple 2: Configuració amb IP estàtica**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

**Exemple 3: Aplicar canvis**

```bash
# Editar configuració
sudo nano /etc/netplan/00-installer-config.yaml

# Aplicar (permanent)
sudo netplan apply

# O provar primer (reverteix automàticament si no confirmes)
sudo netplan try
```

**Exemple 4: Debug si hi ha errors**

```bash
sudo netplan --debug apply
```

### Interpretació del YAML

**Estructura bàsica:**

```yaml
network:
  version: 2 # Versió de Netplan
  renderer: networkd # Backend (networkd o NetworkManager)
  ethernets: # Tipus d'interfície
    enp0s3: # Nom de la interfície
      dhcp4: true # DHCP per IPv4
```

**Configuració DHCP:**

```yaml
enp0s3:
  dhcp4: true # Obté una IP automàticament
```

**Configuració IP estàtica:**

```yaml
enp0s3:
  addresses:
    - 192.168.1.50/24 # IP i màscara
  routes:
    - to: default # Ruta per defecte
      via: 192.168.1.1 # Gateway
  nameservers:
    addresses:
      - 8.8.8.8 # DNS primari
      - 8.8.4.4 # DNS secundari
```

**Assignació de Múltiples IPs a una mateixa interfície:**

```yaml
enp0s3:
  addresses:
    - 192.168.1.100/24
    - 192.168.1.101/24
```

**Configuració amb VLANs:**

```yaml
vlans:
  vlan10:
    id: 10
    link: enp0s3
    addresses:
      - 10.0.10.100/24
```

## 6. `networkctl` - Estat de systemd-networkd

### Descripció

`networkctl` és l'eina de **systemd-networkd** per veure l'estat de les interfícies gestionades pel backend de Netplan. Proporciona una vista ràpida i neta de l'estat de la xarxa.

**Relació amb Netplan:** Netplan genera la configuració pel servei systemd-networkd, i `networkctl` mostra l'estat d'aquesta configuració.

### Sintaxi

```bash
networkctl [COMANDA] [INTERFÍCIE]
```

### Comandes principals

| Comanda  | Descripció                                      |
| -------- | ----------------------------------------------- |
| `list`   | Llista totes les interfícies                    |
| `status` | Estat detallat de totes o una interfície        |
| `lldp`   | Informació LLDP (Link Layer Discovery Protocol) |

### Exemples pràctics

**Exemple 1: Llistar interfícies**

```bash
networkctl list
```

**Sortida:**

```
IDX LINK       TYPE     OPERATIONAL SETUP
  1 lo         loopback carrier     unmanaged
  2 enp0s3     ether    routable    configured
  3 enp0s8     ether    routable    configured
```

**Exemple 2: Estat detallat**

```bash
networkctl status
```

**Sortida:**

```
●        State: routable
  Online state: online
       Address: 192.168.1.100 on enp0s3
                10.0.0.10 on enp0s8
                fe80::a00:27ff:fe3a:b5c1 on enp0s3
       Gateway: 192.168.1.1 on enp0s3
           DNS: 8.8.8.8
                8.8.4.4
```

**Exemple 3: Estat d'una interfície específica**

```bash
networkctl status enp0s3
```

### Interpretació de la sortida

**Estats operacionals:**

- **routable:** Interfície configurada i funcional
- **degraded:** Funcional però amb problemes
- **carrier:** Cable connectat però no configurada
- **no-carrier:** Sense cable connectat
- **off:** Interfície desactivada

**Estats de configuració:**

- **configured:** Configurada correctament
- **configuring:** En procés de configuració
- **unmanaged:** No gestionada per systemd-networkd

## 7. `resolvectl` - Configuració DNS

### Descripció

`resolvectl` gestiona la **resolució de noms DNS** a sistemes amb systemd. Mostra quins servidors DNS s'estan utilitzant i permet fer consultes DNS.

### Sintaxi

```bash
resolvectl [COMANDA]
```

### Comandes principals

| Comanda      | Descripció                   |
| ------------ | ---------------------------- |
| `status`     | Estat de la configuració DNS |
| `query NOM`  | Resol un nom DNS             |
| `statistics` | Estadístiques de resolució   |

### Exemples pràctics

**Exemple 1: Veure configuració DNS**

```bash
resolvectl status
```

**Sortida:**

```
Global
       Protocols: +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (enp0s3)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 8.8.8.8
       DNS Servers: 8.8.8.8 8.8.4.4
        DNS Domain: ~.

Link 3 (enp0s8)
    Current Scopes: none
         Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
```

**Exemple 2: Resoldre un nom DNS**

```bash
resolvectl query google.com
```

**Sortida:**

```
google.com: 142.250.185.46
            2a00:1450:4003:80f::200e

-- Information acquired via protocol DNS in 12.3ms.
-- Data is authenticated: no; Data was acquired via local or encrypted transport: no
-- Data from: network

```

**Exemple 3: Estadístiques**

```bash
resolvectl statistics
```

### Interpretació de la sortida

**Per interfície:**

- **Current DNS Server:** DNS que s'està utilitzant ara
- **DNS Servers:** Llista de servidors DNS configurats
- **DNS Domain:** Domini de cerca

**Fitxers relacionats:**

- `/etc/resolv.conf` - Enllaç simbòlic a systemd-resolved
- `/run/systemd/resolve/resolv.conf` - Configuració real

**Comprovar DNS:**

```bash
# Veure DNS configurats
resolvectl status | grep "DNS Servers"

# Comprovar si resol correctament
resolvectl query google.com
```

## Cas d'ús real: Configurar IP estàtica al servidor

### Escenari

El teu servidor ha estat funcionant amb DHCP però l'empresa vol que es configura la seva IP com una **IP estàtica** per facilitar l'accés. Has de configurar:

- IP: 192.168.1.100/24
- Gateway: 192.168.1.1
- DNS: 8.8.8.8, 8.8.4.4

### Passos

**1. Verificar configuració actual**

```bash
ip a
ip route
```

**2. Identificar nom de la interfície**

```bash
ip a | grep "state UP" | awk '{print $2}' | cut -d: -f1
# Suposem que és enp0s3
```

**3. Editar Netplan**

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

**4. Afegir configuració estàtica**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

**5. Provar la configuració (amb rollback automàtic)**

```bash
sudo netplan try
# Tens 120 segons per verificar que funciona
# Si funciona, prem Enter per confirmar
# Si no, esperarà i revertirà automàticament
```

**6. Si està bé, aplicar permanentment**

```bash
sudo netplan apply
```

**7. Verificar els canvis**

```bash
ip a show enp0s3
ip route
resolvectl status
```

**8. Provar connectivitat**

```bash
ping -c 4 192.168.1.1      # Gateway
ping -c 4 8.8.8.8          # Internet
ping -c 4 google.com       # DNS
```

**9. Documentar**
Anota la configuració al teu sistema de documentació:

- Data del canvi
- IP antiga i nova
- Motiu del canvi
- Qui ho ha fet
