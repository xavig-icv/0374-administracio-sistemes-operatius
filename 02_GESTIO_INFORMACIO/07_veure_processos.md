## 07. Consultar els Processos

Aquest fitxer presenta les ordres per obtenir una **fotografia** dels processos en un moment concret: quins hi ha, de qui són, quins recursos consumeixen i com es relacionen. Per veure'ls en temps real (`top`, `htop`) aneu a [10. Monitorització de Recursos](./10_monitoritzacio.md).

| Ordre        | Funció                                                    |
| ------------ | --------------------------------------------------------- |
| `ps`         | Llista de processos amb les columnes que vulguem          |
| `pstree`     | Arbre de processos (relació pare-fill)                    |
| `pgrep`      | Cerca processos i en retorna el PID                       |
| `/proc/PID/` | Tota la informació d'un procés concret                    |
| `lsof`       | Fitxers i connexions que té oberts un procés              |

### `ps`: Llistar Processos

`ps` admet dos estils d'opcions, i tots dos s'utilitzen molt:

| Estil         | Ordre        | Característiques                                                        |
| ------------- | ------------ | ----------------------------------------------------------------------- |
| BSD (sense `-`) | `ps aux`   | Mostra l'ús de CPU i de memòria (`%CPU`, `%MEM`, `RSS`)                 |
| UNIX (amb `-`)  | `ps -ef`   | Mostra el PID del pare (`PPID`)                                         |
| Personalitzat | `ps -eo ...` | Només les columnes que demanem                                          |

```bash
ps aux
```

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.3  22540 13312 ?        Ss   08:29   0:02 /sbin/init
root           2  0.0  0.0      0     0 ?        S    08:29   0:00 [kthreadd]
root         780  0.0  0.0   7196  2816 ?        Ss   08:29   0:00 /usr/sbin/cron -f -P
openldap     905  0.0  0.5 1094064 21504 ?       Ssl  08:29   0:00 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d
root        1034  0.0  0.2  12020  7936 ?        Ss   08:29   0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
xavi        1520  0.0  0.1   8780  5504 pts/0    Ss   09:12   0:00 -bash
xavi        1745  0.0  0.0   5772  1920 pts/0    S    09:44   0:00 sleep 300
xavi        1750  0.0  0.1  11320  4480 pts/0    R+   09:45   0:00 ps aux
```

| Columna    | Significat                                                                          |
| ---------- | ----------------------------------------------------------------------------------- |
| `USER`     | Propietari del procés. Els serveis solen tenir un usuari propi (`openldap`, `www-data`) |
| `PID`      | Identificador del procés                                                            |
| `%CPU`     | Ús de CPU **mitjà des que va començar** (per a l'ús actual: `top`)                  |
| `%MEM`     | Percentatge de la RAM física que ocupa (`RSS` respecte del total)                    |
| `VSZ`      | Memòria virtual reservada, en KiB (inclou biblioteques i reserves no utilitzades)    |
| `RSS`      | Memòria física (RAM) que ocupa realment, en KiB. És la que cal mirar                |
| `TTY`      | Terminal associat. `?` vol dir que no en té: és un dimoni o un fil del kernel        |
| `STAT`     | Estat (vegeu la taula següent)                                                      |
| `START`    | Hora d'inici (o data, si fa més d'un dia)                                           |
| `TIME`     | Temps total de CPU que ha consumit                                                  |
| `COMMAND`  | Ordre amb els arguments. Entre claudàtors (`[kthreadd]`): fil del kernel            |

**Codis de la columna `STAT`:** la primera lletra és l'estat ([06](./06_processos_conceptes.md#estats-dels-processos)) i les següents en donen més detalls:

| Codi | Significat                                                             |
| ---- | ---------------------------------------------------------------------- |
| `s`  | Líder de sessió (per exemple, el `bash` d'una sessió)                  |
| `l`  | Té diversos fils (*multi-threaded*)                                    |
| `+`  | És al grup de primer pla del seu terminal                              |
| `<`  | Prioritat alta (*nice* negatiu)                                        |
| `N`  | Prioritat baixa (*nice* positiu)                                       |

`Nota:` El procés 1 apareix com `/sbin/init`, però és `systemd`: `/sbin/init` és un enllaç a l'executable de systemd.

**Opcions més útils:**

| Ordre                              | Funció                                                    |
| ---------------------------------- | --------------------------------------------------------- |
| `ps -ef`                           | Tots els processos amb el PID del pare (`PPID`)           |
| `ps -u xavi`                       | Processos d'un usuari                                     |
| `ps -C sshd`                       | Processos amb un nom concret                              |
| `ps -p 1520`                       | Un PID concret                                            |
| `ps --ppid 1`                      | Fills directes d'un procés                                |
| `ps -eo pid,ppid,user,stat,cmd`    | Columnes a mida (`-o`)                                    |
| `ps aux --sort=-%cpu`              | Ordenat per CPU, de més a menys (`-%mem` per memòria)     |
| `ps -eLf`                          | Una línia per fil (*thread*)                              |
| `--no-headers`                     | Sense la línia de capçalera (útil per comptar)            |

**Exemples amb filtres:**

```bash
ps -e --no-headers | wc -l                         # nombre total de processos
ps -u root --no-headers | wc -l                    # processos de root
ps --ppid 1 --no-headers | wc -l                   # fills directes de systemd
ps -o pid,ppid,stat,cmd -p $$                      # el shell actual ($$ = PID del shell)

# Els 5 processos que consumeixen més CPU
ps aux --sort=-%cpu | head -n 6

# Els 5 que consumeixen més memòria: PID, usuari, %MEM i ordre
ps aux --sort=-%mem | awk 'NR > 1 && NR <= 6 {print $2, $1, $4, $11}'

# Quants processos hi ha en cada estat?
ps -eo stat --no-headers | cut -c1 | sort | uniq -c

# Quants processos no tenen terminal (dimonis i fils del kernel)?
ps -eo tty --no-headers | grep -c "^?"
```

```
     84 I
      1 R
     58 S
```

La majoria de processos estan en `S` (esperant) i en `I` (fils del kernel inactius). Que només n'hi hagi un en `R` (el mateix `ps`) indica un sistema poc carregat.

### `pstree`: Arbre de Processos

```bash
pstree -p
```

```
systemd(1)─┬─cron(780)
           ├─rsyslogd(801)─┬─{rsyslogd}(815)
           │               ├─{rsyslogd}(816)
           │               └─{rsyslogd}(817)
           ├─slapd(905)─┬─{slapd}(912)
           │            └─{slapd}(913)
           ├─sshd(1034)─┬─sshd(1490)───sshd(1519)───bash(1520)───pstree(1760)
           │            └─sshd(1688)───sshd(1700)───bash(1701)───sleep(1745)
           ├─systemd(1497)───(sd-pam)(1498)
           ├─systemd-journal(345)
           ├─systemd-logind(791)
           ├─systemd-network(590)
           └─systemd-resolve(612)
```

Cada sessió SSH és una branca de `sshd(1034)`, el procés que escolta el port 22. Els noms entre claus (`{rsyslogd}`) són fils. `systemd(1497)` és el gestor de serveis de l'usuari, que systemd crea per a cada usuari amb una sessió oberta.

| Opció         | Funció                                                         |
| ------------- | -------------------------------------------------------------- |
| `-p`          | Mostra els PID                                                 |
| `-u`          | Indica els canvis d'usuari: `slapd(openldap)`                  |
| `-a`          | Mostra els arguments de cada ordre                             |
| `-s PID`      | Mostra els pares (ascendents) d'un procés                      |
| `-T`          | Amaga els fils                                                 |
| `usuari`      | Només els processos d'un usuari: `pstree xavi`                 |

```bash
pstree -s -p $$                    # cadena de pares del shell actual
```

```
systemd(1)───sshd(1034)───sshd(1490)───sshd(1519)───bash(1520)───pstree(1762)
```

### `pgrep`: Cercar Processos

`pgrep` retorna els PID dels processos que coincideixen amb un patró. És més segur que `ps aux | grep`, perquè no es troba a si mateix.

| Opció         | Funció                                                       |
| ------------- | ------------------------------------------------------------ |
| `-l`          | Mostra també el nom                                          |
| `-a`          | Mostra la línia d'ordre completa                             |
| `-u usuari`   | Només els processos d'un usuari                              |
| `-c`          | Compta els processos en lloc de mostrar-los                  |
| `-n` / `-o`   | Només el més nou / el més antic                              |
| `-x`          | El nom ha de coincidir exactament                            |
| `-f`          | Cerca a tota la línia d'ordre, no només al nom               |
| `-t pts/1`    | Processos d'un terminal                                      |
| `-P PID`      | Fills d'un procés                                            |

```bash
pgrep -l sshd                      # PID i nom dels processos sshd
pgrep -a bash                      # PID i línia d'ordre dels bash
pgrep -c -u root                   # quants processos té root
pgrep -o systemd                   # el systemd més antic: 1
pgrep -af "http.server 9000"       # procés que conté aquest text als arguments
```

```
1034 sshd
1490 sshd
1519 sshd
```

`Nota:` El nom d'un procés (el que mira `pgrep` sense `-f`) té com a màxim 15 caràcters. Per a noms més llargs o per distingir processos pels arguments, feu servir `-f`.

### El Directori `/proc/PID/`

Cada procés té un directori a `/proc` amb el seu PID com a nom. Hi ha tota la informació que fan servir `ps` i `top`:

| Fitxer       | Contingut                                                          |
| ------------ | ------------------------------------------------------------------ |
| `cmdline`    | Ordre completa amb els arguments                                   |
| `exe`        | Enllaç a l'executable real                                         |
| `cwd`        | Enllaç al directori de treball                                     |
| `fd/`        | Fitxers oberts: cada número és un descriptor de fitxer             |
| `status`     | Estat en text: nom, estat, PPID, usuari, memòria, nombre de fils   |
| `environ`    | Variables d'entorn (només les pot llegir el propietari o root)     |

```bash
ls -l /proc/$$/exe                 # executable del shell actual
sudo ls -l /proc/1/exe             # executable del procés 1
ls /proc/$$/fd | wc -l             # quants fitxers té oberts el shell
grep -E "^(Name|State|PPid|Threads|VmRSS)" /proc/$$/status
```

```
lrwxrwxrwx 1 xavi xavi 0 Nov  5 09:50 /proc/1520/exe -> /usr/bin/bash
lrwxrwxrwx 1 root root 0 Nov  5 08:29 /proc/1/exe -> /usr/lib/systemd/systemd
4
Name:	bash
State:	S (sleeping)
PPid:	1519
Threads:	1
VmRSS:	    5504 kB
```

Els 4 fitxers oberts del shell són l'entrada, la sortida i la sortida d'errors (0, 1 i 2, tots apuntant al terminal `/dev/pts/0`) i el 255, que `bash` reserva per al terminal. Per consultar processos d'un altre usuari cal `sudo`.

**Informació global del sistema a `/proc`:** `/proc/cpuinfo` (CPU), `/proc/meminfo` (memòria), `/proc/loadavg` (càrrega), `/proc/uptime` (temps engegat), `/proc/version` (kernel).

### `lsof`: Fitxers i Connexions Oberts

```bash
sudo lsof -p 905                   # fitxers oberts pel procés 905 (slapd)
sudo lsof -i :22                   # quin procés fa servir el port 22
sudo lsof /var/log/syslog          # quins processos tenen obert aquest fitxer
```

`lsof` és útil quan un disc no es pot desmuntar (`target is busy`): mostra quin procés en té fitxers oberts.

### Detectar Processos Sospitosos

Un procés maliciós (per exemple, un programa de mineria de criptomonedes) sol deixar pistes:

| Pista                                                    | Com detectar-la                                                     |
| -------------------------------------------------------- | ------------------------------------------------------------------- |
| Consum de CPU alt i constant d'un procés desconegut      | `ps aux --sort=-%cpu \| head`                                       |
| Executable a `/tmp`, `/var/tmp` o `/dev/shm`             | `sudo ls -l /proc/*/exe 2>/dev/null \| grep -E "/tmp\|/dev/shm"`    |
| Executable esborrat que continua en marxa                | `sudo ls -l /proc/*/exe 2>/dev/null \| grep deleted`                |
| Nom que imita un procés del sistema (`[kworker]`, `sshd`) amb un executable estrany | Comparar `ps` amb `ls -l /proc/PID/exe`. Els fils del kernel no tenen executable |
| Port en escolta que no correspon a cap servei conegut    | `sudo ss -tulnp`                                                    |

Davant d'un procés sospitós, primer **recolliu informació** (usuari, executable, ordre completa, connexions amb `sudo lsof -p PID`) i després atureu-lo. Si l'acabeu de seguida, perdeu les proves. Comproveu també si s'ha programat per tornar a arrencar (`crontab -l`, serveis de systemd).

`Nota:` Després d'una actualització, alguns serveis legítims també apareixen amb l'executable `(deleted)`, perquè continuen executant la versió anterior. Es resol reiniciant el servei.

### Resolució de Problemes

| Problema                                                   | Causa                                                    | Solució                                         |
| ---------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------- |
| `ps aux \| grep nom \| wc -l` dona un procés de més        | Compta també el procés `grep`                            | `pgrep -c nom`                                  |
| `ls: cannot read symbolic link '/proc/1/exe': Permission denied` | El procés és d'un altre usuari                   | `sudo ls -l /proc/1/exe`                        |
| `pgrep` no troba un procés que es veu a `ps`               | El nom té més de 15 caràcters o el text és als arguments | `pgrep -f`                                      |
| El `%CPU` de `ps` no coincideix amb el de `top`            | `ps` mostra la mitjana des de l'inici; `top`, l'ús actual | Fer servir `top` per a l'ús actual             |
| `lsof: command not found`                                  | Paquet no instal·lat                                     | `sudo apt install lsof`                         |
