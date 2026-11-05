## 01. Escenari del Laboratori

El mòdul 0374 es treballa amb el **servidor 0374**. És la mateixa màquina que el mòdul 0375 (Serveis de Xarxa i Internet) fa servir com a gateway i servidor secundari. Els dos mòduls comparteixen l'escenari: les mateixes màquines, les mateixes adreces i el mateix client.

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

### Pla d'Adreces

| IP              | Equip         | Funció al mòdul 0374                                                 |
| --------------- | ------------- | -------------------------------------------------------------------- |
| `192.168.1.1`   | Servidor 0374 | Sortida a Internet de la xarxa interna (NAT), servidor LDAP (Tema 1) |
| `192.168.1.2`   | Servidor 0375 | Servidor dels serveis del mòdul 0375 (client LDAP a la integració)  |
| `192.168.1.50`  | Client Ubuntu | Client de proves (autenticació LDAP, connexions SSH...)              |

**Dades comunes:**

- **Sistema operatiu:** Ubuntu Server 24.04 LTS (servidors) i Ubuntu Desktop 24.04 (client).
- **Nom del servidor 0374:** `srv0374`. Els exemples del temari el fan servir.
- **Directori LDAP (Tema 1):** `dc=empresa,dc=cat`.
- **Domini DNS del laboratori:** `educem.cat` (es configura al mòdul 0375).

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

### Sortida a Internet de la Xarxa Interna (NAT)

El client i el servidor 0375 surten a Internet a través del 0374. Calen dues coses: que el 0374 reenviï paquets entre interfícies (`ip_forward`) i que tradueixi les adreces internes per la seva IP del centre (NAT).

**1. Activar el reenviament de paquets (permanent):**

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl --system
sysctl net.ipv4.ip_forward        # ha de mostrar = 1
```

**2. NAT amb nftables.** Afegiu al final de `/etc/nftables.conf`:

```
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 192.168.1.0/24 oifname "enp0s3" masquerade
    }
}
```

```bash
sudo nft -c -f /etc/nftables.conf       # comprovar la sintaxi
sudo systemctl enable --now nftables    # aplicar-ho ara i a cada arrencada
sudo nft list table ip nat
```

`Nota:` Si al servidor feu servir `ufw`, no activeu el servei `nftables`: el seu fitxer comença amb `flush ruleset` i esborraria les regles d'`ufw`. En aquest cas, el NAT es configura a `/etc/ufw/before.rules` i cal permetre el reenviament (`DEFAULT_FORWARD_POLICY="ACCEPT"` a `/etc/default/ufw`).

### Comprovació de l'Escenari

**Al servidor 0374:**

```bash
ip -br a                          # enp0s3 amb IP del centre i enp0s8 amb 192.168.1.1
ip route | grep default           # default via ... dev enp0s3
sysctl net.ipv4.ip_forward        # = 1
ping -c1 www.google.com           # el servidor surt a Internet
```

**Al client:**

```bash
ip route | grep default           # default via 192.168.1.1
ping -c1 192.168.1.1              # arriba al 0374
ping -c1 8.8.8.8                  # surt a Internet a través del NAT del 0374
```

### Problemes Habituals

| Símptoma | Causa probable | Solució |
| -------- | -------------- | ------- |
| El client arriba al 0374 però no a Internet | `ip_forward` desactivat o sense regla de NAT | Revisar `sysctl net.ipv4.ip_forward` i `sudo nft list ruleset` |
| El client fa ping a `8.8.8.8` però no resol noms | No té cap servidor DNS configurat | Revisar `resolvectl dns` al client (DHCP o netplan) |
| Després de reiniciar, el client ja no surt a Internet | La configuració no es va fer permanent | Fitxer a `/etc/sysctl.d/` i servei `nftables` habilitat |
| `enp0s8` no té IP | Error al YAML de netplan | `sudo netplan try` mostra l'error |
