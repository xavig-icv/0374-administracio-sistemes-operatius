## 06. Bastionat de l'SSH

**Bastionar** (*hardening*) un servei vol dir reduir-ne al mínim les possibilitats d'atac. Un servidor SSH accessible des d'Internet rep intents d'accés automàtics constantment (Tema 2, fitxer 05): cal combinar diverses mesures, perquè cap no és suficient per si sola.

### Llista de Comprovació

| Mesura                                         | Com                                                                   | Fitxer          |
| ---------------------------------------------- | --------------------------------------------------------------------- | --------------- |
| Autenticació només amb claus                   | `PasswordAuthentication no`                                           | [00_RECURSOS/02](../00_RECURSOS/02_connexio_ssh.md) |
| Sense accés directe de root                    | `PermitRootLogin no`                                                  | [00_RECURSOS/02](../00_RECURSOS/02_connexio_ssh.md) |
| Només els usuaris que ho necessiten            | `AllowGroups` amb un grup per als administradors                      | [02](./02_configuracio_ssh.md) |
| Pocs intents i poc temps per autenticar-se     | `MaxAuthTries 3`, `LoginGraceTime 30`                                 | [02](./02_configuracio_ssh.md) |
| Sense túnels per als usuaris que no en necessiten | `AllowTcpForwarding no` en un bloc `Match`                         | [05](./05_tunels_ssh.md) |
| Port obert només a les xarxes necessàries      | Tallafoc (`ufw`)                                                      | Aquest fitxer   |
| Bloqueig automàtic de qui insisteix            | `fail2ban`                                                            | Aquest fitxer   |
| Sistema actualitzat                            | Actualitzacions de seguretat automàtiques (`unattended-upgrades`)     | Aquest fitxer   |
| Revisar els accessos                           | `last`, `lastb`, `journalctl -u ssh`                                  | Tema 2, fitxer 05 |

### Tallafoc: `ufw`

```bash
sudo ufw allow from 10.0.1.0/24 to any port 22 proto tcp     # SSH només des de la xarxa del centre
sudo ufw limit 22/tcp            # alternativa: limita les connexions repetides des d'una mateixa IP
sudo ufw enable
sudo ufw status numbered
```

**Afegiu la regla de l'SSH abans d'activar el tallafoc**: si no, la sessió es talla i només es pot entrar per la consola de VirtualBox. `ufw limit` bloqueja una IP que obre 6 connexions o més en 30 segons.

`Nota:` Si el servidor 0374 fa de porta d'enllaç amb el NAT de nftables ([escenari](../00_RECURSOS/01_escenari_laboratori.md)), no activeu `ufw` sense adaptar-hi el NAT: les dues configuracions es poden trepitjar.

### Bloqueig Automàtic: `fail2ban`

`fail2ban` llegeix els registres i, quan una IP falla massa vegades, la bloqueja al tallafoc durant un temps:

```bash
sudo apt install fail2ban
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled  = true
maxretry = 5
findtime = 10m
bantime  = 1h
ignoreip = 127.0.0.1/8 10.0.1.10
```

| Paràmetre   | Significat                                                           |
| ----------- | -------------------------------------------------------------------- |
| `maxretry`  | Intents fallits permesos                                             |
| `findtime`  | Període en què es compten els intents                                |
| `bantime`   | Temps de bloqueig                                                    |
| `ignoreip`  | Adreces que no es bloquegen mai (per exemple, l'ordinador de l'administrador) |

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd                     # IP bloquejades i intents
sudo fail2ban-client set sshd unbanip 10.0.1.25      # desbloquejar una IP
```

El mòdul 0375 fa servir `fail2ban` també per protegir el servidor de correu.

### Actualitzacions de Seguretat Automàtiques

Ubuntu Server porta instal·lat `unattended-upgrades`, que instal·la cada dia les actualitzacions de seguretat:

```bash
systemctl status unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades            # "1" = activat
sudo unattended-upgrade --dry-run --debug           # què s'instal·laria
```

### Mesures Addicionals

- **Segon factor d'autenticació (2FA):** el paquet `libpam-google-authenticator` afegeix un codi temporal d'una aplicació del mòbil a més de la clau.
- **Auditoria de la configuració:** l'eina `ssh-audit` (paquet `ssh-audit`) revisa els algorismes que accepta el servidor i avisa dels febles.
- **VPN:** en lloc d'obrir l'SSH a Internet, accedir primer a la xarxa de l'empresa per una VPN.

### Resolució de Problemes

| Problema                                                    | Causa                                                    | Solució                                                     |
| ----------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| La sessió s'ha tallat en activar `ufw`                      | No hi havia cap regla per a l'SSH                        | Consola de VirtualBox: `sudo ufw allow 22/tcp`              |
| L'administrador ha quedat bloquejat per `fail2ban`          | Massa intents fallits des de la seva IP                  | `fail2ban-client set sshd unbanip IP` des de la consola i afegir la IP a `ignoreip` |
| `fail2ban-client status sshd` diu que la presó no existeix  | La secció `[sshd]` no està activada o hi ha un error al fitxer | `sudo fail2ban-client -d` mostra la configuració i els errors |
| Després de configurar `AllowGroups` ningú no pot entrar     | Cap usuari és al grup indicat                            | Consola de VirtualBox: afegir l'usuari al grup              |
