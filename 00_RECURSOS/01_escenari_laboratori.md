## 01. Escenari del Laboratori

El mòdul 0374 es treballa amb el **servidor 0374**. És la mateixa màquina que el mòdul 0375 (Serveis de Xarxa i Internet) fa servir com a porta d'enllaç del client i com a servidor secundari. Els dos mòduls comparteixen l'escenari: les mateixes màquines, les mateixes adreces i el mateix client.

### Topologia

```
                         Xarxa del centre (adaptador pont) → Internet
              ┌───────────────────────────┴───────────────────────────┐
              │ enp0s3: 10.0.1.X (DHCP del centre)                     │ enp0s3: 10.0.1.X (DHCP del centre)
   ┌──────────┴──────────┐                                 ┌──────────┴──────────┐
   │   SERVIDOR 0374     │                                 │   SERVIDOR 0375     │
   │ Gateway (NAT)       │                                 │ Serveis del mòdul   │
   │ LDAP (Tema 1)       │                                 │ 0375 (DNS, web...)  │
   └──────────┬──────────┘                                 └──────────┬──────────┘
              │ enp0s8: 192.168.1.1/24                                 │ enp0s8: 192.168.1.2/24
              └───────────────────────────┬───────────────────────────┘
                         Xarxa interna "intnet" 192.168.1.0/24
                                          │
                                          │ enp0s3: 192.168.1.50
                               ┌──────────┴──────────┐
                               │   CLIENT UBUNTU     │
                               │ Gateway: .1         │
                               └─────────────────────┘
```

- **Els dos servidors** tenen dues interfícies. Per `enp0s3` són a la xarxa del centre: el DHCP del centre els dona una IP `10.0.1.X`, que fan servir per **sortir a Internet** i per rebre les connexions des de l'aula. Per `enp0s8` són a la xarxa interna.
- **El client** només és a la xarxa interna. Surt a Internet **a través del 0374** (NAT), i per connectar-s'hi per SSH des de l'aula cal saltar pel 0374 (Tema 4).

### Pla d'Adreces

| IP              | Equip         | Funció al mòdul 0374                                                 |
| --------------- | ------------- | -------------------------------------------------------------------- |
| `192.168.1.1`   | Servidor 0374 | Porta d'enllaç del client (NAT), servidor LDAP (Tema 1)              |
| `192.168.1.2`   | Servidor 0375 | Servidor dels serveis del mòdul 0375 (client LDAP a la integració)  |
| `192.168.1.3`   | VM Jitsi      | Només al mòdul 0375 (Tema 8, videoconferència)                       |
| `192.168.1.4`   | VM Samba AD   | Controlador de domini (Tema 6, fitxer 06)                            |
| `192.168.1.50`  | Client Ubuntu | Client de proves (autenticació LDAP, connexions SSH...)              |

**Dades comunes:**

- **Sistema operatiu:** Ubuntu Server 24.04 LTS (servidors) i Ubuntu Desktop 24.04 (client).
- **Nom del servidor 0374:** `srv0374`. Els exemples del temari el fan servir.
- **Directori LDAP (Tema 1):** `dc=educem,dc=cat`.
- **Domini DNS del laboratori:** `educem.cat` (es configura al mòdul 0375).
- **Tallafoc del 0374:** `ufw`, actiu des del principi. També fa el NAT (vegeu "Sortida a Internet del Client").

### Configuració de Xarxa del Servidor 0374

Fitxer `/etc/netplan/50-cloud-init.yaml` (o el que hi hagi a `/etc/netplan/`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:                       # adaptador pont: xarxa del centre i sortida a Internet
      dhcp4: true
    enp0s8:                       # xarxa interna
      addresses: [192.168.1.1/24]
```

```bash
sudo netplan apply
ip -br a                          # enp0s8 ha de tenir la 192.168.1.1/24
```

### Sortida a Internet del Client (NAT)

El client fa servir el 0374 (`192.168.1.1`) com a porta d'enllaç. Perquè pugui sortir a Internet, al 0374 calen dues coses: que reenviï paquets entre interfícies (`ip_forward`) i que tradueixi les adreces internes per la seva IP del centre (NAT).

El servidor 0375 no passa pel 0374: surt a Internet per la seva pròpia interfície del centre.

**1. Activar el reenviament de paquets (permanent):**

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl --system
sysctl net.ipv4.ip_forward        # ha de mostrar = 1
```

El NAT el fa el tallafoc **`ufw`**, que queda actiu des d'ara. Per això, cada tema que instal·la un servei al 0374 (en aquest mòdul i al 0375) també n'obre el port.

**2. Permetre l'SSH** abans d'activar el tallafoc. Si no, la sessió es talla i només es pot entrar per la consola de VirtualBox:

```bash
sudo ufw allow OpenSSH
```

**3. NAT.** Afegiu aquestes línies al **principi** de `/etc/ufw/before.rules`, abans de la línia `*filter`:

```
# NAT: la xarxa interna surt a Internet amb la IP del centre (enp0s3)
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.1.0/24 -o enp0s3 -j MASQUERADE
COMMIT
```

**4. Permetre el reenviament** de la xarxa interna cap a la del centre, i activar el tallafoc:

```bash
sudo ufw route allow in on enp0s8 out on enp0s3
sudo ufw enable
sudo ufw status verbose                       # deny (routed) i la regla ALLOW FWD
sudo iptables -t nat -L POSTROUTING -n -v     # regla MASQUERADE de 192.168.1.0/24
```

Per defecte, `ufw` no deixa passar res d'una xarxa a l'altra (`deny (routed)`). La regla `route allow` permet les connexions que surten de la xarxa interna, i `ufw` deixa tornar les respostes perquè formen part d'una connexió establerta. Des de la xarxa del centre no es pot obrir cap connexió cap a la xarxa interna.

`Nota:` No activeu el servei `nftables` en aquest servidor: el seu fitxer comença amb `flush ruleset` i esborraria les regles d'`ufw`. Si ja l'havíeu activat, feu `sudo systemctl disable --now nftables` i després `sudo ufw reload`.

### Comprovació de l'Escenari

**Al servidor 0374:**

```bash
ip -br a                          # enp0s3 amb IP del centre i enp0s8 amb 192.168.1.1
ip route | grep default           # default via ... dev enp0s3
sysctl net.ipv4.ip_forward        # = 1
sudo ufw status                   # Status: active, amb l'SSH i la regla ALLOW FWD
ping -c1 www.google.com           # el servidor surt a Internet
```

**Al client:**

```bash
ip route | grep default           # default via 192.168.1.1
ping -c1 192.168.1.1              # arriba al 0374
ping -c1 8.8.8.8                  # surt a Internet a través del NAT del 0374
```

**Al servidor 0375:**

```bash
ip route | grep default           # default via ... dev enp0s3: surt per la xarxa del centre, no pel 0374
```

### Problemes Habituals

| Símptoma | Causa probable | Solució |
| -------- | -------------- | ------- |
| El client arriba al 0374 però no a Internet | `ip_forward` desactivat, sense NAT o sense la regla de reenviament | `sysctl net.ipv4.ip_forward`, `sudo iptables -t nat -L POSTROUTING -n -v` i `sudo ufw status verbose` |
| El client fa ping a `8.8.8.8` però no resol noms | No té cap servidor DNS configurat | Revisar `resolvectl dns` al client (DHCP o netplan) |
| Després de reiniciar, el client ja no surt a Internet | El reenviament no és permanent, o el servei `nftables` ha esborrat les regles d'`ufw` | Fitxer a `/etc/sysctl.d/`; `systemctl is-enabled nftables` no ha de dir `enabled` |
| Un servei del 0374 (LDAP, CUPS, NFS...) no respon des del client | El tallafoc no en permet el port | `sudo ufw status` i obrir el port, com indica cada tema |
| `enp0s8` no té IP | Error al YAML de netplan | `sudo netplan try` mostra l'error |
| El 0375 es queda sense Internet quan s'apaga el 0374 | Té una ruta per defecte cap a `192.168.1.1` | Al netplan del 0375, `enp0s8` no ha de tenir `routes` ni `gateway4` |
