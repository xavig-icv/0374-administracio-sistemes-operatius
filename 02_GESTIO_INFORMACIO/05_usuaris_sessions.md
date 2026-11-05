## 05. Usuaris i Sessions

Saber **qui utilitza el servidor** és una tasca bàsica de seguretat: qui hi ha connectat ara, des d'on, quan es va connectar cada usuari per última vegada i si algú intenta entrar sense permís.

### On es Registren les Sessions

| Fitxer                                 | Contingut                                   | Ordre          |
| -------------------------------------- | ------------------------------------------- | -------------- |
| `/run/utmp`                            | Sessions obertes ara                        | `who`, `w`     |
| `/var/log/wtmp`                        | Històric de sessions, reinicis i aturades   | `last`         |
| `/var/log/btmp`                        | Intents d'inici de sessió fallits           | `sudo lastb`   |
| `/var/log/lastlog`                     | Última connexió de cada usuari              | `lastlog`      |
| `/var/log/auth.log` i el diari de systemd | Detall de cada autenticació (SSH, `sudo`) | `journalctl`, `grep` |

`utmp`, `wtmp`, `btmp` i `lastlog` són fitxers binaris: no es poden llegir amb `cat`, cal fer servir l'ordre corresponent. `wtmp` i `btmp` es roten periòdicament amb `logrotate`: les dades anteriors queden a `wtmp.1` i `btmp.1`.

### Usuaris Connectats: `who`

```bash
who
```

```
xavi     pts/0        2026-11-05 09:12 (10.0.1.20)
asixprofe pts/1        2026-11-05 09:30 (10.0.1.21)
```

Cada línia és una sessió: usuari, terminal, hora d'inici i origen. El terminal indica el tipus de sessió:

- `tty1`, `tty2`...: consola local (la finestra de VirtualBox o la pantalla del servidor).
- `pts/0`, `pts/1`...: pseudoterminal (sessió SSH o terminal d'un escriptori gràfic).

| Opció      | Informació                                                                     |
| ---------- | ------------------------------------------------------------------------------ |
| `-H`       | Afegeix la capçalera de les columnes                                           |
| `-u`       | Temps d'inactivitat (`.` = actiu l'últim minut, `old` = més de 24 h) i PID de la sessió |
| `-q`       | Només els noms i el nombre d'usuaris connectats                                |
| `-b`       | Data i hora de l'última arrencada del sistema                                  |

### Què Fan els Usuaris: `w`

```bash
w
```

```
 09:41:07 up  1:12,  2 users,  load average: 0.00, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU  WHAT
xavi     pts/0    10.0.1.20        09:12    0.00s  0.08s  0.01s w
asixprof pts/1    10.0.1.21        09:30   11:02   0.03s  0.03s -bash
```

| Columna   | Significat                                                                    |
| --------- | ----------------------------------------------------------------------------- |
| 1a línia  | La mateixa informació que `uptime`: hora, temps engegat, usuaris i càrrega mitjana |
| `FROM`    | Equip d'origen de la connexió                                                 |
| `LOGIN@`  | Hora d'inici de la sessió                                                     |
| `IDLE`    | Temps sense escriure res al terminal                                          |
| `JCPU`    | Temps de CPU de tots els processos del terminal                               |
| `PCPU`    | Temps de CPU del procés actual                                                |
| `WHAT`    | Ordre que s'està executant. `-bash` vol dir que l'usuari no executa res       |

`w` retalla els noms d'usuari a 8 caràcters (`asixprof`). Per veure'ls sencers: `PROCPS_USERLEN=16 w`. Amb `w usuari` es mostra només un usuari i amb `w -h` s'omet la capçalera (útil per filtrar).

### Històric de Connexions: `last`

```bash
last
```

```
asixprof pts/1        10.0.1.21        Thu Nov  5 09:30   still logged in
xavi     pts/0        10.0.1.20        Thu Nov  5 09:12   still logged in
reboot   system boot  6.8.0-87-generic Thu Nov  5 08:29   still running
xavi     pts/0        10.0.1.20        Wed Nov  4 15:02 - 17:45  (02:43)
xavi     pts/1        10.0.1.20        Wed Nov  4 12:10 - crash  (02:51)
asixprof pts/0        10.0.1.21        Tue Nov  3 10:05 - 10:20  (00:15)

wtmp begins Mon Sep 15 10:11:42 2025
```

| Final de línia          | Significat                                                                 |
| ----------------------- | -------------------------------------------------------------------------- |
| `still logged in`       | La sessió continua oberta                                                  |
| `15:02 - 17:45 (02:43)` | Hora d'entrada, de sortida i durada                                        |
| `crash`                 | El sistema es va aturar sense tancar la sessió (apagada sobtada)           |
| `gone - no logout`      | La sessió va acabar sense registrar la sortida (per exemple, el procés va morir) |
| `reboot system boot`    | Arrencada del sistema i versió del kernel                                  |

| Opció                    | Funció                                                            |
| ------------------------ | ----------------------------------------------------------------- |
| `last usuari`            | Només les sessions d'un usuari                                    |
| `last reboot`            | Només els reinicis                                                |
| `-n 10`                  | Les 10 últimes entrades                                           |
| `-x`                     | Inclou les aturades (`shutdown`)                                  |
| `-i`                     | Mostra l'origen sempre com a adreça IP                            |
| `-w`                     | Noms d'usuari sencers (sense retallar a 8 caràcters)              |
| `-F`                     | Dates completes (amb any i segons)                                |
| `-s` / `-t`              | Des de / fins a una data: `-s today`, `-s yesterday`, `-s -7days`, `-s 2026-11-01` |

```bash
last -s -7days xavi | grep -c "^xavi"                  # sessions de xavi els últims 7 dies
last -i xavi | awk '$1 == "xavi" {print $3}' | sort -u     # IP des de les quals s'ha connectat
last -x reboot shutdown | head -n 4                    # últims reinicis i aturades
```

### Última Connexió de Cada Usuari: `lastlog`

```bash
lastlog
```

```
Username         Port     From                                       Latest
root                                                                 **Never logged in**
daemon                                                               **Never logged in**
...
xavi             pts/0    10.0.1.20                                  Thu Nov  5 09:12:44 +0100 2026
asixprofe        pts/1    10.0.1.21                                  Thu Nov  5 09:30:02 +0100 2026
```

| Opció        | Funció                                                    |
| ------------ | --------------------------------------------------------- |
| `-u usuari`  | Només un usuari                                           |
| `-b 90`      | Usuaris que fa **més** de 90 dies que no es connecten (o mai) |
| `-t 7`       | Usuaris que s'han connectat els **últims** 7 dies         |

`root` apareix com `**Never logged in**` encara que feu servir `sudo`: `sudo` no és un inici de sessió. Els comptes de servei (`daemon`, `www-data`...) tampoc no inicien mai sessió.

`Nota:` Algunes distribucions recents (per exemple, Debian 13) substitueixen `lastlog` per `lastlog2` i el fitxer `wtmp` per una base de dades (`wtmpdb`). En aquests sistemes, les ordres equivalents són `lastlog2` i `wtmpdb last`.

### Intents d'Accés Fallits: `lastb` i Registres

```bash
sudo lastb -n 5
```

```
invent   ssh:notty    10.0.1.25        Thu Nov  5 10:02 - 10:02  (00:00)
root     ssh:notty    10.0.1.25        Thu Nov  5 10:02 - 10:02  (00:00)
root     ssh:notty    10.0.1.25        Thu Nov  5 10:01 - 10:01  (00:00)
xavi     ssh:notty    10.0.1.25        Thu Nov  5 10:01 - 10:01  (00:00)
asixprof ssh:notty    10.0.1.25        Thu Nov  5 10:00 - 10:00  (00:00)

btmp begins Thu Nov  5 10:00:12 2026
```

`ssh:notty` indica un intent per SSH. Apareixen tant usuaris que existeixen (`xavi`) com usuaris inventats (`invent`): els atacants proven noms habituals (`admin`, `test`, `root`...). `lastb` accepta les mateixes opcions que `last` (`-i`, `-s today`...).

A Ubuntu, `root` no té contrasenya i l'SSH no hi permet entrar amb contrasenya (`PermitRootLogin prohibit-password`). Tot i així, els intents queden registrats i són un bon indicador d'atac.

**Detall dels intents als registres de l'SSH:**

```bash
sudo journalctl -u ssh --since today | grep -E "Failed|Invalid"
```

```
Nov 05 10:01:33 srv0374 sshd[2101]: Failed password for xavi from 10.0.1.25 port 50412 ssh2
Nov 05 10:02:10 srv0374 sshd[2105]: Invalid user invent from 10.0.1.25 port 50420
Nov 05 10:02:12 srv0374 sshd[2105]: Failed password for invalid user invent from 10.0.1.25 port 50420 ssh2
```

La mateixa informació és a `/var/log/auth.log`, juntament amb cada ús de `sudo` (`grep sudo /var/log/auth.log`).

### Comptes del Sistema

Cada línia de `/etc/passwd` és un compte amb 7 camps separats per `:`.

```
xavi:x:1000:1000:Xavi Garcia,,,:/home/xavi:/bin/bash
```

| Camp | Valor            | Significat                                                           |
| ---- | ---------------- | -------------------------------------------------------------------- |
| 1    | `xavi`           | Nom d'usuari                                                         |
| 2    | `x`              | La contrasenya (xifrada) és a `/etc/shadow`, que només pot llegir root |
| 3    | `1000`           | UID (identificador d'usuari)                                         |
| 4    | `1000`           | GID del grup principal                                               |
| 5    | `Xavi Garcia,,,` | Descripció (nom complet)                                             |
| 6    | `/home/xavi`     | Directori personal                                                   |
| 7    | `/bin/bash`      | Shell. Amb `/usr/sbin/nologin` o `/bin/false` el compte no pot iniciar sessió |

| UID          | Tipus de compte                                          |
| ------------ | -------------------------------------------------------- |
| `0`          | `root` (administrador)                                   |
| `1` a `999`  | Comptes del sistema i de serveis (`www-data`, `sshd`...) |
| `1000` o més | Persones                                                 |
| `65534`      | `nobody` (compte sense privilegis)                       |

**`getent` en lloc de `cat`:** `getent passwd` consulta totes les fonts d'usuaris configurades al sistema (`/etc/nsswitch.conf`). Si el servidor és client LDAP (Tema 1), també mostra els usuaris del directori, que no apareixen a `/etc/passwd`.

```bash
getent passwd xavi                 # un compte
getent group sudo                  # membres del grup sudo
id asixprofe                       # UID, grup principal i grups d'un usuari
```

```
sudo:x:27:xavi
uid=1001(asixprofe) gid=1001(asixprofe) groups=1001(asixprofe)
```

### Comprovacions de Seguretat

| Comprovació                              | Ordre                                                          | Resultat esperat             |
| ---------------------------------------- | -------------------------------------------------------------- | ---------------------------- |
| Comptes amb UID 0                        | `awk -F: '$3 == 0 {print $1}' /etc/passwd`                     | Només `root`                 |
| Comptes de persones                      | `getent passwd \| awk -F: '$3 >= 1000 && $3 < 65534 {print $1}'` | Només usuaris coneguts     |
| Comptes que poden iniciar sessió         | `grep -E '/(ba)?sh$' /etc/passwd`                              | `root` i les persones        |
| Membres del grup `sudo` (administradors) | `getent group sudo`                                            | Només els administradors     |
| Comptes sense contrasenya                | `sudo awk -F: '$2 == ""' /etc/shadow`                          | Cap línia                    |
| Comptes sense ús                         | `lastlog -b 90`                                                | Revisar si cal desactivar-los |

Un segon compte amb UID 0 és un forat de seguretat greu: té tots els permisos de `root` amb un altre nom.

### Cas Pràctic: Algú Intenta Entrar al Servidor

**1. Hi ha intents fallits recents?**

```bash
sudo lastb -s today | grep -c "ssh:notty"
```

**2. Des de quines IP? (rànquing)**

```bash
sudo lastb -i | awk 'NF > 0 && $1 != "btmp" {print $3}' | sort | uniq -c | sort -rn
```

```
     17 10.0.1.25
      3 10.0.1.31
```

`10.0.1.25` concentra els intents: és l'origen de l'atac. Per veure només les IP amb més de 10 intents, afegiu `| awk '$1 > 10'`.

**3. Quins usuaris ha provat?**

```bash
sudo lastb | awk 'NF > 0 && $1 != "btmp" {print $1}' | sort | uniq -c | sort -rn | head
```

**4. Ha aconseguit entrar?** Aquesta és la pregunta més important:

```bash
last -i | grep 10.0.1.25
```

Si no surt res, cap intent ha tingut èxit. Si apareix una sessió, cal revisar què s'ha fet amb aquest usuari i canviar-ne la contrasenya immediatament.

**5. Mesures:** bloquejar l'IP al tallafoc (si el servidor fa servir `ufw`: `sudo ufw insert 1 deny from 10.0.1.25`; `insert 1` la posa davant de la regla que permet l'SSH), bloquejar-la automàticament amb `fail2ban` (vist al mòdul 0375) i permetre l'accés SSH només amb claus (Tema 4).

### Resolució de Problemes

| Problema                                                | Causa                                                  | Solució                                         |
| ------------------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------- |
| `lastb: cannot open /var/log/btmp: Permission denied`   | Només root pot llegir `btmp`                            | `sudo lastb`                                    |
| `last` i `w` mostren `asixprof` en lloc d'`asixprofe`   | Retallen els noms a 8 caràcters                         | `last -w`, `PROCPS_USERLEN=16 w`                |
| `last` no mostra les sessions més antigues              | `wtmp` s'ha rotat                                       | `last -f /var/log/wtmp.1`                       |
| `lastlog: command not found`                            | Sistema que fa servir `lastlog2`                        | `lastlog2`                                      |
| `getent passwd` no mostra els usuaris LDAP              | El servidor no és client LDAP o `nsswitch.conf` no inclou `ldap`/`sss` | Revisar la configuració del client (Tema 1) |
| `who` no mostra un usuari que té un procés en marxa     | El procés s'executa sense sessió (servei, `nohup`, `cron`) | Buscar-lo amb `ps -u usuari`                 |
