## 11. Registres del Sistema i Diagnosi

Els **registres** (*logs*) són el diari del sistema: cada servei hi anota què fa, quins errors troba i qui s'hi connecta. Quan alguna cosa falla, són la primera font d'informació per saber **què ha passat i quan**.

A Ubuntu hi ha dos sistemes de registre que funcionen alhora:

| Sistema                  | On desa els registres                     | Com es consulten                    |
| ------------------------ | ----------------------------------------- | ----------------------------------- |
| **journald** (systemd)   | `/var/log/journal/` (format binari)       | `journalctl`                        |
| **rsyslog**              | Fitxers de text a `/var/log/`             | `tail`, `less`, `grep`              |

journald recull els missatges de tots els serveis i del kernel; rsyslog en rep una còpia i els escriu en fitxers de text.

### Fitxers de `/var/log`

| Fitxer                        | Contingut                                                    |
| ----------------------------- | ------------------------------------------------------------ |
| `syslog`                      | Missatges generals del sistema i dels serveis                |
| `auth.log`                    | Autenticacions: SSH, `sudo`, inicis de sessió ([05](./05_usuaris_sessions.md)) |
| `kern.log`                    | Missatges del kernel                                         |
| `dpkg.log`, `apt/history.log` | Paquets instal·lats, actualitzats i eliminats                |
| `cloud-init.log`              | Configuració inicial del servidor                            |
| `ufw.log`                     | Connexions bloquejades pel tallafoc (si `ufw` està actiu)    |
| `apache2/`, `nginx/`          | Registres dels servidors web (mòdul 0375)                    |
| `wtmp`, `btmp`, `lastlog`     | Sessions i intents fallits, en format binari ([05](./05_usuaris_sessions.md)) |

Els usuaris del grup `adm` poden llegir aquests fitxers sense `sudo`.

```bash
tail -n 20 /var/log/syslog                   # últimes 20 línies
tail -f /var/log/auth.log                    # seguir el fitxer en temps real (sortir amb Ctrl+C)
grep -i "error" /var/log/syslog              # cercar errors
zgrep "Failed password" /var/log/auth.log.2.gz   # cercar dins d'un registre antic comprimit
```

Els registres es **roten**: periòdicament el fitxer actual passa a dir-se `syslog.1`, els anteriors es comprimeixen (`syslog.2.gz`...) i els més antics s'esborren. Ho fa `logrotate`, que es configura al Tema 3.

### `journalctl`: Consultar el Diari de systemd

| Opció                            | Funció                                                          |
| -------------------------------- | --------------------------------------------------------------- |
| `-u ssh`                         | Missatges d'un servei                                           |
| `-b` / `-b -1`                   | De l'arrencada actual / de l'anterior                           |
| `--list-boots`                   | Llista les arrencades registrades                               |
| `-p err`                         | Només missatges d'aquesta prioritat o més greus                 |
| `-k`                             | Només missatges del kernel (com `dmesg`)                        |
| `--since` / `--until`            | Interval de temps: `"1 hour ago"`, `today`, `"2026-11-05 09:00"` |
| `-f`                             | Seguir en temps real                                            |
| `-n 50`                          | Les últimes 50 línies                                           |
| `-e`                             | Anar directament al final                                       |
| `-x`                             | Afegir explicacions als missatges que en tenen                  |
| `-g text`                        | Només les línies que contenen el text (com `grep`)              |
| `-o short-iso`                   | Dates en format ISO (`2026-11-05T10:20:01+01:00`)               |
| `_UID=1000`, `_PID=1520`         | Filtrar per usuari o per procés                                 |

**Prioritats dels missatges:**

| Núm. | Nom       | Significat                          |
| ---- | --------- | ----------------------------------- |
| 0    | `emerg`   | El sistema no es pot fer servir     |
| 1    | `alert`   | Cal actuar immediatament            |
| 2    | `crit`    | Situació crítica                    |
| 3    | `err`     | Error                               |
| 4    | `warning` | Avís                                |
| 5    | `notice`  | Normal, però rellevant              |
| 6    | `info`    | Informació                          |
| 7    | `debug`   | Depuració                           |

```bash
journalctl -p err -b                                # errors des de l'última arrencada
journalctl -u ssh --since today -g "Failed"         # intents fallits d'avui
journalctl -u monitor-carrega --since "10 min ago"  # el servei del fitxer 09
journalctl -b -1 -e                                 # final de l'arrencada anterior (què va passar abans d'un reinici?)
```

```
Nov 05 08:29:43 srv0374 kernel: piix4_smbus 0000:00:07.0: SMBus Host Controller not enabled!
Nov 05 11:02:31 srv0374 systemd[1]: Failed to start ssh.service - OpenBSD Secure Shell server.
```

El primer error és habitual a les VM de VirtualBox i no té cap efecte. El segon, en canvi, s'ha d'investigar (vegeu el cas pràctic).

**Espai que ocupa el diari:**

```bash
journalctl --disk-usage
sudo journalctl --vacuum-size=200M       # reduir-lo a 200 MB esborrant els registres més antics
sudo journalctl --vacuum-time=2weeks     # esborrar els registres de més de 2 setmanes
```

### Missatges del Kernel: `dmesg`

`dmesg` mostra els missatges del kernel: detecció de maquinari, controladors, errors de disc, connexió de dispositius USB, canvis d'enllaç de xarxa. A Ubuntu necessita `sudo`.

```bash
sudo dmesg -T | tail -n 20               # -T: dates llegibles
sudo dmesg -T -l err,warn                # només errors i avisos
sudo dmesg -w                            # seguir en temps real
```

```
[Thu Nov  5 08:29:41 2026] e1000 0000:00:03.0 enp0s3: renamed from eth0
[Thu Nov  5 08:29:44 2026] e1000: enp0s3 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
[Thu Nov  5 10:42:10 2026] e1000: enp0s8 NIC Link is Down
```

| Missatge                                    | Significat                                                   |
| ------------------------------------------- | ------------------------------------------------------------ |
| `NIC Link is Up` / `NIC Link is Down`       | S'ha connectat o desconnectat el cable de xarxa              |
| `I/O error, dev sda`                        | Errors de lectura o escriptura: el disc pot estar fallant    |
| `Out of memory: Killed process`             | L'OOM killer ha matat un procés per manca de memòria ([10](./10_monitoritzacio.md)) |
| `usb 1-1: new ... device`                   | S'ha connectat un dispositiu USB                             |

### Escriure als Registres: `logger`

Els scripts poden deixar constància del que fan als registres del sistema amb `logger`:

```bash
logger -t informe "Informe del servidor generat correctament"
journalctl -t informe                    # consultar els missatges amb aquesta etiqueta
```

És la manera recomanada perquè els scripts programats (Tema 3) informin de si han funcionat.

### Metodologia de Diagnosi

| Pas | Què cal fer                                                 | Ordres                                              |
| --- | ----------------------------------------------------------- | --------------------------------------------------- |
| 1   | Definir el símptoma: què falla, des de quan, a qui afecta   | Preguntar, reproduir el problema                    |
| 2   | Comprovar l'estat del servei afectat                        | `systemctl status servei`                           |
| 3   | Llegir els registres del moment de l'error                  | `journalctl -u servei --since ...`, `/var/log/...`  |
| 4   | Comprovar els recursos i la xarxa                           | `df -h`, `free -h`, `top`, `ss -tulnp`, `ip a`      |
| 5   | Fer **un sol canvi** cada vegada i comprovar-ne l'efecte    | Validar la configuració abans de reiniciar          |
| 6   | Documentar la causa i la solució                            | Informe de l'incident                               |

**Validar la configuració abans de reiniciar un servei.** Si un fitxer de configuració té un error, el servei no tornarà a arrencar. La majoria de serveis tenen una ordre per comprovar-ho:

| Servei               | Ordre de comprovació                      |
| -------------------- | ----------------------------------------- |
| SSH                  | `sudo sshd -t`                            |
| Nginx                | `sudo nginx -t`                           |
| Apache               | `sudo apache2ctl configtest`              |
| BIND (DNS)           | `sudo named-checkconf`                    |
| `sudo`               | `sudo visudo -c`                          |
| Netplan              | `sudo netplan try`                        |

### Problemes Habituals

| Problema                          | Com detectar-lo                                     | Solució                                                       |
| --------------------------------- | --------------------------------------------------- | ------------------------------------------------------------- |
| Disc ple                          | `df -h` (ús ≥ 90%); serveis que fallen en escriure  | Localitzar què ocupa: `sudo du -xh / --max-depth=1 \| sort -rh \| head`. Netejar: `sudo apt clean`, `sudo journalctl --vacuum-size=200M` |
| Un servei no arrenca              | `systemctl status`, `journalctl -u servei -n 30`    | Corregir la configuració (comprovar-la abans) i tornar-lo a engegar |
| No es pot entrar per SSH          | `ss -tln \| grep :22`, `sudo ufw status`, `journalctl -u ssh` | Engegar el servei, obrir el port, revisar la configuració |
| S'esgota la memòria               | `free -h`, `sudo dmesg -T \| grep -i "out of memory"` | Identificar el procés que consumeix ([10](./10_monitoritzacio.md)) |
| El disc dona errors               | `sudo dmesg -T \| grep -i "i/o error"`              | Còpia de seguretat immediata i substituir el disc             |
| Les hores dels registres no quadren | `timedatectl`                                     | `sudo timedatectl set-timezone Europe/Madrid`                 |

### Cas Pràctic: l'SSH no Arrenca després d'un Canvi

Un administrador ha modificat `/etc/ssh/sshd_config` i ha reiniciat el servei:

```
$ sudo systemctl restart ssh
Job for ssh.service failed because the control process exited with error code.
See "systemctl status ssh.service" and "journalctl -xeu ssh.service" for details.
```

**1. Què diuen els registres?**

```bash
journalctl -u ssh -n 5
```

```
Nov 05 11:02:31 srv0374 sshd[2290]: /etc/ssh/sshd_config: line 14: Bad configuration option: PermitRootLogn
Nov 05 11:02:31 srv0374 sshd[2290]: /etc/ssh/sshd_config: terminating, 1 bad configuration options
Nov 05 11:02:31 srv0374 systemd[1]: ssh.service: Control process exited, code=exited, status=255/EXCEPTION
Nov 05 11:02:31 srv0374 systemd[1]: ssh.service: Failed with result 'exit-code'.
Nov 05 11:02:31 srv0374 systemd[1]: Failed to start ssh.service - OpenBSD Secure Shell server.
```

El registre indica el fitxer, la línia (14) i l'error: `PermitRootLogn` està mal escrit (`PermitRootLogin`).

**2. Corregir i validar:**

```bash
sudo nano +14 /etc/ssh/sshd_config      # obrir el fitxer a la línia 14
sudo sshd -t                            # sense sortida = configuració correcta
sudo systemctl restart ssh
systemctl status ssh
```

**Lliçó:** comproveu sempre la configuració (`sshd -t`) **abans** de reiniciar, i no tanqueu la sessió SSH fins que el servei torni a funcionar. Les sessions obertes continuen, però les noves connexions fallaran. Si es tanca l'última sessió, només es podrà entrar per la consola de la màquina.

### Resolució de Problemes

| Problema                                                       | Causa                                                    | Solució                                              |
| -------------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| `journalctl` només mostra els meus missatges                   | L'usuari no és dels grups `adm` o `systemd-journal`      | `sudo journalctl` o afegir l'usuari al grup `adm`    |
| `dmesg: read kernel buffer failed: Operation not permitted`    | A Ubuntu, `dmesg` està restringit                         | `sudo dmesg`                                         |
| `journalctl -b -1` no troba l'arrencada anterior               | El diari no es desa al disc (només a memòria)            | `sudo mkdir -p /var/log/journal` i reiniciar `systemd-journald` |
| Les hores dels registres estan desplaçades                     | La zona horària és UTC                                    | `sudo timedatectl set-timezone Europe/Madrid`        |
| `grep` no troba res en un registre antic                       | El fitxer està comprimit (`.gz`)                         | `zgrep`, `zless`                                     |
| `sshd -t` diu `Missing privilege separation directory: /run/sshd` | El servei està aturat i el directori temporal no existeix | `sudo mkdir -p /run/sshd` i tornar a comprovar      |
