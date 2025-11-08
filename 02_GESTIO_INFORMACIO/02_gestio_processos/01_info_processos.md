# 01. Veure Processos

Abans de poder gestionar processos, necessitem saber quins són els processos que estan executant, quin usuari els ha iniciat, quants recursos consumeixen i quina acció estan realitzant. En aquest bloc treballarem les comandes essencials per visualitzar i cercar processos al sistema.

**Objectiu**: Conèixer les comandes per identificar processos al servidor.

## Comandes d'aquest tema

1. `ps` - Llista de processos actius
2. `pstree` - Vista jeràrquica (arbre) de processos
3. `pgrep` - Cerca processos per nom o atributs
4. `/proc/` - Sistema de fitxers virtual amb informació detallada

## 1. ps - Process Status

### Descripció

La comanda `ps` mostra una **fotografia** dels processos actius en un moment determinat. És la comanda més bàsica i més utilitzada per veure processos.

### Sintaxi

```bash
ps [OPCIONS]
```

### Opcions principals

| Opció          | Descripció                                            |
| -------------- | ----------------------------------------------------- |
| `ps`           | Processos de l'usuari actual al terminal actual       |
| `ps aux`       | Tots els processos de tots els usuaris (en un format) |
| `ps -ef`       | Tots els processos (en un format diferent)            |
| `ps -u USUARI` | Processos associats a un usuari específic             |
| `ps -p PID`    | Informació associats a un PID específic               |
| `ps --forest`  | Vista jeràrquica de processos (format d'arbre)        |

### Estils de sintaxi

`ps` disposa de dos estils de sintaxi (històrics):

**Estil BSD (sense guió):**

```bash
ps aux
```

**Estil System V (amb guió):**

```bash
ps -ef
```

**Recomanació:** Utilitza `ps aux` (més comú i la sortida disposa d'un format més intuïtiu).

### Exemples pràctics

**Exemple 1: Veure tots els processos**

```bash
ps aux
```

**Sortida:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1 168304 11234 ?        Ss   09:00   0:03 /sbin/init
root         2  0.0  0.0      0     0 ?        S    09:00   0:00 [kthreadd]
www-data  1234  2.3  1.5 245678 98765 ?        S    10:30   1:23 /usr/sbin/apache2
xavi      5678  0.1  0.3  45678  8901 pts/0    S+   11:00   0:02 vim server.py
mysql     9012  1.5  5.2 1234567 345678 ?      Sl   09:05  10:45 /usr/sbin/mysqld
```

**Exemple 2: Processos d'un usuari específic**

```bash
ps -u xavi
```

**Exemple 3: Buscar un procés per nom**

```bash
ps aux | grep apache
```

**Exemple 4: Ordenar per ús de CPU**

```bash
ps aux --sort=-%cpu | head -n 10
```

**Exemple 5: Ordenar per ús de memòria**

```bash
ps aux --sort=-%mem | head -n 10
```

**Exemple 6: Vista jeràrquica (arbre)**

```bash
ps auxf
# O
ps aux --forest
```

**Exemple 7: Informació d'un PID específic**

```bash
ps -p 1234 -o pid,user,%cpu,%mem,command
```

**Exemple 8: Només processos en execució (Running)**

```bash
ps aux | awk '$8 ~ /R/ {print $0}'
```

### Interpretació de les columnes

**Columnes de `ps aux`:**

| Columna     | Significat                                   |
| ----------- | -------------------------------------------- |
| **USER**    | Usuari propietari del procés                 |
| **PID**     | Process ID (identificador únic)              |
| **%CPU**    | Percentatge d'ús de CPU                      |
| **%MEM**    | Percentatge d'ús de memòria RAM              |
| **VSZ**     | Virtual Memory Size (memòria virtual en KB)  |
| **RSS**     | Resident Set Size (memòria física en KB)     |
| **TTY**     | Terminal associat (? = cap terminal, dimoni) |
| **STAT**    | Estat del procés                             |
| **START**   | Hora d'inici del procés                      |
| **TIME**    | Temps de CPU acumulat                        |
| **COMMAND** | Comanda executada                            |

### Estats dels processos (STAT)

| Codi  | Estat           | Significat                                |
| ----- | --------------- | ----------------------------------------- |
| **R** | Running         | Executant-se o preparat                   |
| **S** | Sleeping        | Esperant (interruptible)                  |
| **D** | Disk sleep      | Esperant I/O (no interruptible)           |
| **T** | Stopped         | Aturat (Ctrl+Z o SIGSTOP)                 |
| **Z** | Zombie          | Procés mort, esperant que el pare reculli |
| **<** | Alta prioritat  | Nice < 0                                  |
| **N** | Baixa prioritat | Nice > 0                                  |
| **l** | Multi-threaded  | Procés amb múltiples threads              |
| **s** | Session leader  | Líder de sessió                           |
| **+** | Foreground      | Al primer pla del terminal                |

**Exemples combinats:**

- **Ss**: Sleeping + Session leader (típic de dimonis - serveis)
- **R+**: Running + Foreground (comanda executada actualment al terminal)
- **D**: Disk sleep (esperant al disc, pot indicar un problema)
- **Z**: Zombie (procés mort però amb presència a la taula de processos)

### VSZ vs RSS (Memòria)

**VSZ (Virtual Size):**

- Memòria virtual total que el procés **pot** utilitzar
- Inclou memòria compartida, swap, etc.
- **Més gran** que la memòria real utilitzada

**RSS (Resident Set Size):**

- Memòria **física (RAM)** que el procés està utilitzant **actualment**
- Més representatiu del consum real
- **Utilitza aquest valor** per saber la quantitat de RAM que consumeix

**Exemple:**

```
VSZ: 500MB  → El procés pot utilitzar fins a 500MB
RSS: 50MB   → Actualment utilitza 50MB de RAM física
```

### Casos d'ús pràctics

**Trobar processos que consumeixen més CPU:**

```bash
ps aux --sort=-%cpu | head -n 11
# (head -n 11 perquè la primera línia és la capçalera)
```

**Trobar els processos que consumeixen més memòria:**

```bash
ps aux --sort=-%mem | head -n 11
```

**Comptar els processos d'un usuari:**

```bash
ps -u www-data | wc -l
```

**Veure els processos d'Apache:**

```bash
ps aux | grep apache | grep -v grep
```

**Llistar només els PIDs d'un servei:**

```bash
ps aux | grep mysql | grep -v grep | awk '{print $2}'
```

**Veure els threads d'un procés:**

```bash
ps -eLf | grep 1234
```

## 2. pstree - Arbre de processos

### Descripció

`pstree` mostra els processos en una **vista jeràrquica** (d'arbre), mostrant la relació pare-fill entre processos. És molt útil per entendre com s'organitzen els processos.

### Sintaxi

```bash
pstree [OPCIONS] [PID o USUARI]
```

### Opcions principals

| Opció           | Descripció                            |
| --------------- | ------------------------------------- |
| (sense opcions) | Arbre complet des de systemd (PID 1)  |
| `-p`            | Mostra PIDs                           |
| `-u`            | Mostra canvis d'usuari                |
| `-a`            | Mostra arguments de la comanda        |
| `-c`            | No agrupa processos idèntics          |
| `-h`            | Ressalta el procés actual i ancestres |
| `PID`           | Arbre des d'un PID específic          |

### Exemples pràctics

**Exemple 1: Arbre complet del sistema**

```bash
pstree
```

**Sortida (simplificada):**

```
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─accounts-daemon───2*[{accounts-daemon}]
        ├─apache2───5*[apache2]
        ├─cron
        ├─dbus-daemon
        ├─mysqld───20*[{mysqld}]
        ├─networkd-dispatcher
        ├─sshd───sshd───bash───pstree
        ├─systemd-journald
        ├─systemd-logind
        ├─systemd-networkd
        └─systemd-timesyncd
```

**Exemple 2: Amb PIDs**

```bash
pstree -p
```

**Sortida (parcial):**

```
systemd(1)─┬─apache2(1234)─┬─apache2(1235)
           │                ├─apache2(1236)
           │                ├─apache2(1237)
           │                └─apache2(1238)
           ├─cron(567)
           ├─mysqld(890)─┬─{mysqld}(891)
           │             ├─{mysqld}(892)
           │             └─{mysqld}(893)
           └─sshd(1000)───sshd(5678)───bash(5679)
```

**Exemple 3: Arbre d'un procés específic**

```bash
pstree -p 1234
```

**Sortida:**

```
apache2(1234)─┬─apache2(1235)
              ├─apache2(1236)
              ├─apache2(1237)
              └─apache2(1238)
```

**Exemple 4: Amb arguments de comandes**

```bash
pstree -a
```

**Exemple 5: Processos d'un usuari concret**

```bash
pstree www-data
```

**Exemple 6: Ressaltar procés actual**

```bash
pstree -h
# Ressalta el terminal actual i els seus ancestres
```

### Interpretació de l'arbre

**Notació:**

- `─` : Línia de connexió
- `┬` : Bifurcació (múltiples fills)
- `├` : Fill intermig
- `└` : Últim fill
- `5*[apache2]` : 5 processos idèntics d'Apache

**Exemple d'interpretació:**

```
systemd(1)───sshd(1000)───sshd(5678)───bash(5679)───nano(5680)
```

**Significat:**

1. systemd (PID 1) és el pare de tot
2. sshd (1000) és el dimoni SSH
3. sshd (5678) és la sessió SSH d'un usuari
4. bash (5679) és el shell de l'usuari
5. nano (5680) és l'editor obert per l'usuari

**Quina és la relació entre processos?** nano és fill de bash, bash és fill de sshd, etc.

### Casos d'ús pràctics

**Veure jerarquia d'Apache:**

```bash
pstree -p | grep apache
```

**Trobar el pare d'un procés:**

```bash
pstree -p | grep 5678
# Veuràs tota la cadena de pares
```

**Comptar fills d'un procés:**

```bash
pstree -p 1234 | grep -o "([0-9]*)" | wc -l
```

## 3. pgrep - Cercar processos

### Descripció

`pgrep` cerca processos a partir d'uns **criteris** (nom, usuari, terminal...) i retorna els **PIDs**. És com unificar les comandes `ps` i `grep`. És molt útil per fer scripts i automatització de tasques que es treballen a la RA3 - AUTOMATITZACIÓ DE TASQUES i la RA7 - SCRIPTING.

**Diferència amb `ps | grep`:**

- `pgrep apache` és més net que `ps aux | grep apache | grep -v grep`
- Retorna només PIDs (fàcil per scripts)

### Sintaxi

```bash
pgrep [OPCIONS] PATRÓ
```

### Opcions principals

| Opció       | Descripció                                    |
| ----------- | --------------------------------------------- |
| `-l`        | Mostra nom del procés a més del PID           |
| `-u USUARI` | Processos d'un usuari                         |
| `-x`        | Coincidència exacta del nom                   |
| `-f`        | Cerca a la comanda completa (no només el nom) |
| `-n`        | Mostra només el procés més recent             |
| `-o`        | Mostra només el procés més antic              |
| `-c`        | Compta processos (no mostra PIDs)             |

### Exemples pràctics

**Exemple 1: Trobar PIDs d'Apache**

```bash
pgrep apache
```

**Sortida:**

```
1234
1235
1236
1237
```

**Exemple 2: Amb noms de processos**

```bash
pgrep -l apache
```

**Sortida:**

```
1234 apache2
1235 apache2
1236 apache2
```

**Exemple 3: Processos d'un usuari**

```bash
pgrep -u www-data
```

**Exemple 4: Coincidència exacta**

```bash
pgrep -x sshd
# Troba "sshd" però no "sshd: user@pts/0"
```

**Exemple 5: Cerca la comanda completa**

```bash
pgrep -f "python.*server.py"
# Troba processos Python executant server.py
```

**Exemple 6: Procés més recent**

```bash
pgrep -n apache
# Només el PID de l'Apache més recent
```

**Exemple 7: Comptar processos**

```bash
pgrep -c apache
# Sortida: 5 (nombre de processos Apache)
```

### Casos d'ús pràctics

**Matar tots els processos d'un usuari:**

```bash
kill $(pgrep -u testuser)
```

**Comprovar si un servei està executant-se:**

```bash
if pgrep -x mysqld > /dev/null; then
  echo "MySQL està executant-se"
else
  echo "MySQL NO està executant-se"
fi
```

**Obtenir el PID del procés pare d'Apache:**

```bash
pgrep -o apache
# El més antic és el procés pare
```

**Llistar els processos Python:**

```bash
pgrep -l python
```

**Script per monitoritzar un servei:**

```bash
#!/bin/bash
SERVICE="apache"

if ! pgrep -x $SERVICE > /dev/null; then
  echo "$(date): $SERVICE no està executant-se!" >> /var/log/monitor.log
  systemctl start $SERVICE
fi
```

## 4. Sistema de fitxers /proc/

### Descripció

`/proc/` és un **sistema de fitxers virtual** que proporciona una interfície per accedir a informació del kernel sobre processos i el sistema. No existeix al disc, es genera en temps real.

**Contingut:**

- Directoris numerats (PIDs): `/proc/1234/`
- Fitxers de sistema: `/proc/cpuinfo`, `/proc/meminfo`, etc.

### Estructura de /proc/[PID]/

Cada procés disposa d'un directori amb el seu PID:

```bash
ls /proc/1234/
```

**Fitxers importants:**

| Fitxer    | Contingut                                       |
| --------- | ----------------------------------------------- |
| `cmdline` | Comanda completa amb arguments                  |
| `cwd`     | Directori de treball actual (enllaç)            |
| `environ` | Variables d'entorn                              |
| `exe`     | Executable (enllaç simbòlic)                    |
| `fd/`     | Directori amb file descriptors (fitxers oberts) |
| `status`  | Estat detallat del procés                       |
| `stat`    | Estadístiques (format brut)                     |
| `maps`    | Mapes de memòria                                |
| `limits`  | Límits de recursos                              |
| `io`      | Estadístiques d'I/O                             |

### Exemples pràctics

**Exemple 1: Veure la comanda d'un procés**

```bash
cat /proc/1234/cmdline
```

**Sortida:**

```
/usr/bin/python3/opt/app/server.py--port8080
```

_Nota: Els arguments estan separats per `\0` (null bytes)_

**Per fer-ho més llegible amb trim:**

```bash
cat /proc/1234/cmdline | tr '\0' ' '
```

**Sortida:**

```
/usr/bin/python3 /opt/app/server.py --port 8080
```

---

**Exemple 2: Des d'on s'executa el procés**

```bash
ls -l /proc/1234/cwd
```

**Sortida:**

```
lrwxrwxrwx 1 user user 0 Oct 20 10:00 /proc/1234/cwd -> /opt/app
```

---

**Exemple 3: Quin executable està executant**

```bash
ls -l /proc/1234/exe
```

**Sortida:**

```
lrwxrwxrwx 1 user user 0 Oct 20 10:00 /proc/1234/exe -> /usr/bin/python3.10
```

---

**Exemple 4: Fitxers oberts pel procés**

```bash
ls -l /proc/1234/fd/
```

**Sortida:**

```
total 0
lrwx------ 1 user user 64 Oct 20 10:00 0 -> /dev/pts/0
lrwx------ 1 user user 64 Oct 20 10:00 1 -> /dev/pts/0
lrwx------ 1 user user 64 Oct 20 10:00 2 -> /dev/pts/0
lrwx------ 1 user user 64 Oct 20 10:00 3 -> socket:[12345]
lrwx------ 1 user user 64 Oct 20 10:00 4 -> /var/log/app.log
```

**Interpretació:**

- **0, 1, 2**: stdin, stdout, stderr (terminal)
- **3**: Socket de xarxa
- **4**: Fitxer de log obert

**Exemple 5: Variables d'entorn**

```bash
cat /proc/1234/environ | tr '\0' '\n'
```

**Sortida:**

```
PATH=/usr/local/bin:/usr/bin
HOME=/home/user
USER=user
SHELL=/bin/bash
```

---

**Exemple 6: Estat detallat del procés**

```bash
cat /proc/1234/status
```

**Sortida (parcial):**

```
Name:   python3
State:  S (sleeping)
Pid:    1234
PPid:   1000  ← Pare (Parent PID)
Uid:    1000  1000  1000  1000
Gid:    1000  1000  1000  1000
VmSize:  123456 kB  ← Memòria virtual
VmRSS:    45678 kB  ← Memòria física
Threads:  1
```

**Exemple 7: Límits de recursos**

```bash
cat /proc/1234/limits
```

**Sortida (parcial):**

```
Limit                     Soft Limit           Hard Limit           Units
Max open files            1024                 1048576              files
Max processes             31161                31161                processes
Max locked memory         65536                65536                bytes
```

**Exemple 8: Estadístiques d'I/O**

```bash
cat /proc/1234/io
```

**Sortida:**

```
rchar: 123456789         ← Bytes llegits
wchar: 987654321         ← Bytes escrits
syscr: 12345             ← Crides de lectura
syscw: 6789              ← Crides d'escriptura
read_bytes: 98765432     ← Bytes llegits del disc
write_bytes: 54321098    ← Bytes escrits al disc
```

### Fitxers globals de /proc/

**Informació del sistema (no específica de processos):**

```bash
# CPU
cat /proc/cpuinfo

# Memòria
cat /proc/meminfo

# Versió del kernel
cat /proc/version

# Temps d'activitat
cat /proc/uptime

# Càrrega mitjana
cat /proc/loadavg

# Estadístiques de xarxa
cat /proc/net/dev

# Particions
cat /proc/partitions

# Mòduls del kernel carregats
cat /proc/modules
```

## Combinacions útils

### Trobar processos que consumeixen més recursos

**CPU:**

```bash
ps aux --sort=-%cpu | head -n 11 | awk '{print $2, $3, $11}'
```

**Memòria:**

```bash
ps aux --sort=-%mem | head -n 11 | awk '{print $2, $4, $11}'
```

### Trobar el pare d'un procés

**Opció 1: Amb ps**

```bash
ps -o ppid= -p 1234
```

**Opció 2: Amb /proc/**

```bash
grep "PPid" /proc/1234/status | awk '{print $2}'
```

**Opció 3: Amb pstree**

```bash
pstree -p | grep 1234
```

### Llistar processos d'un servei amb tots els detalls

```bash
ps aux | grep apache | grep -v grep | awk '{print "PID:"$2, "CPU:"$3"%", "MEM:"$4"%", "CMD:"$11}'
```
