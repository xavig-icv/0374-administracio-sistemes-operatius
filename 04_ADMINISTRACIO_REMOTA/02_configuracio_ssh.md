## 02. Configuració Avançada del Servidor i del Client SSH

### Fitxers de Configuració

| Fitxer                               | Funció                                                                         |
| ------------------------------------ | ------------------------------------------------------------------------------ |
| `/etc/ssh/sshd_config`               | Configuració del **servidor** (`sshd`)                                         |
| `/etc/ssh/sshd_config.d/*.conf`      | Fitxers addicionals del servidor. Es llegeixen **abans** que el fitxer principal |
| `/etc/ssh/ssh_config`                | Configuració del **client** per a tots els usuaris                             |
| `~/.ssh/config`                      | Configuració del client de cada usuari                                         |
| `~/.ssh/authorized_keys`             | Claus públiques autoritzades a entrar amb aquest usuari (fitxer 03)            |
| `~/.ssh/known_hosts`                 | Claus dels servidors coneguts (fitxer 03)                                      |

Recordeu (guia de [connexió SSH](../00_RECURSOS/02_connexio_ssh.md)): al servidor, **per a cada paràmetre val el primer valor que es llegeix**. Les configuracions pròpies es posen en un fitxer de `sshd_config.d/` amb un número baix (per exemple, `10-seguretat.conf`).

### Directives Principals del Servidor

| Directiva                       | Exemple                        | Funció                                                        |
| ------------------------------- | ------------------------------ | ------------------------------------------------------------- |
| `Port`                          | `Port 2222`                    | Port d'escolta                                                |
| `ListenAddress`                 | `ListenAddress 192.168.1.1`    | Adreça on escolta (per defecte, totes)                        |
| `PermitRootLogin`               | `no`                           | Accés directe de root                                         |
| `PasswordAuthentication`        | `no`                           | Autenticació amb contrasenya                                  |
| `PubkeyAuthentication`          | `yes`                          | Autenticació amb clau                                         |
| `AllowUsers` / `AllowGroups`    | `AllowGroups sudo admins-ssh`  | **Només** aquests usuaris o grups poden entrar                |
| `DenyUsers` / `DenyGroups`      | `DenyUsers asixprofe`          | Usuaris o grups que no poden entrar                           |
| `MaxAuthTries`                  | `3`                            | Intents d'autenticació per connexió                           |
| `LoginGraceTime`                | `30`                           | Segons per completar l'inici de sessió                        |
| `ClientAliveInterval` / `ClientAliveCountMax` | `300` / `2`      | Tanca les sessions inactives (300 s × 2 sense resposta)       |
| `AllowTcpForwarding`            | `no`                           | Permet o no els túnels (fitxer 05)                            |
| `X11Forwarding`                 | `no`                           | Aplicacions gràfiques a través de l'SSH                       |
| `Banner`                        | `/etc/issue.net`               | Missatge legal que es mostra abans d'iniciar sessió           |

Els paràmetres es poden aplicar només a alguns usuaris o xarxes amb blocs `Match`:

```
# Els usuaris del grup sftp-web només poden fer SFTP (mòdul 0375)
Match Group sftp-web
    ForceCommand internal-sftp
    AllowTcpForwarding no
```

**Aplicar els canvis amb seguretat:**

```bash
sudo sshd -t                                   # comprovar la sintaxi
sudo sshd -T | grep -i allowgroups             # valor que s'aplicarà
sudo systemctl restart ssh
```

No tanqueu mai la sessió oberta fins que hàgiu comprovat amb una connexió nova que podeu entrar.

### Canviar el Port a Ubuntu 24.04

A Ubuntu 24.04 l'SSH s'activa per socket: és `ssh.socket` qui escolta el port. Un generador de systemd llegeix `Port` i `ListenAddress` de la configuració i adapta el socket, però cal recarregar-lo:

```bash
sudo ufw allow 2222/tcp                        # 1. obrir el port nou al tallafoc (si està actiu)
echo "Port 2222" | sudo tee /etc/ssh/sshd_config.d/20-port.conf
sudo sshd -t
sudo systemctl daemon-reload                   # 2. el generador actualitza ssh.socket
sudo systemctl restart ssh.socket
sudo ss -tlnp | grep 2222                      # 3. comprovar-ho
```

Canviar el port **no protegeix** contra un atac dirigit (un escaneig de ports el troba), però redueix molt els intents automàtics que arriben al port 22.

### Configuració del Client: `~/.ssh/config`

El fitxer `~/.ssh/config` evita haver d'escriure cada vegada l'usuari, l'adreça, el port i la clau. A Windows és `C:\Users\<usuari>\.ssh\config`.

```
Host srv0374
    HostName 10.0.1.37
    User xavi
    IdentityFile ~/.ssh/id_ed25519

Host srv0375
    HostName 10.0.1.38
    User xavi

Host client
    HostName 192.168.1.50
    User alumne
    ProxyJump srv0374

Host *
    ServerAliveInterval 60
```

Amb aquesta configuració, `ssh srv0374` és equivalent a `ssh -i ~/.ssh/id_ed25519 xavi@10.0.1.37`.

- Als dos servidors s'hi arriba directament per la seva IP del centre. Les IP d'aquest exemple s'han de canviar per les que el DHCP del centre hagi donat als vostres servidors (`ip -br a`).
- El client només és a la xarxa interna: `ssh client` hi arriba passant pel 0374 (fitxer 05).

| Directiva            | Funció                                                        |
| -------------------- | ------------------------------------------------------------- |
| `Host`               | Nom curt (àlies). `Host *` s'aplica a totes les connexions    |
| `HostName`           | Adreça IP o nom real                                          |
| `User`               | Usuari remot                                                  |
| `Port`               | Port, si no és el 22                                          |
| `IdentityFile`       | Clau privada que s'ha de fer servir                           |
| `ProxyJump`          | Equip pel qual s'ha de passar per arribar al destí            |
| `ServerAliveInterval`| Envia un senyal cada N segons perquè la connexió no es talli  |

### Executar Ordres Remotes

```bash
ssh srv0374 "uptime"                           # executa una ordre i torna
ssh srv0374 "sudo systemctl status cron --no-pager"
ssh -t srv0374 "sudo journalctl -f"            # -t: terminal interactiu (cal per a sudo i ordres interactives)
ssh -v srv0374                                 # mode detallat per diagnosticar problemes de connexió
```

### Resolució de Problemes

| Problema                                                        | Causa                                                        | Solució                                                     |
| --------------------------------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| Després de canviar el port, no es pot connectar                 | Tallafoc o socket no recarregat                              | Obrir el port amb `ufw` i `daemon-reload` + reiniciar `ssh.socket` |
| `Permission denied` després d'afegir `AllowGroups`              | L'usuari no és a cap dels grups indicats                     | Afegir-lo al grup o revisar la directiva (consola de VirtualBox si cal) |
| Un canvi a `sshd_config` no té efecte                           | Un fitxer de `sshd_config.d/` defineix el mateix paràmetre abans | `sudo sshd -T` mostra el valor efectiu                  |
| `Bad configuration option` en reiniciar                         | Error d'escriptura a la configuració                         | `sudo sshd -t` indica el fitxer i la línia                  |
| `sudo: a terminal is required to read the password`             | L'ordre remota s'executa sense terminal                      | Afegir `-t`: `ssh -t srv0374 "sudo ..."`                    |
