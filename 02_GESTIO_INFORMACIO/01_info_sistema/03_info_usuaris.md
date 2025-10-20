# 03. Usuaris i Sessions

## Introducció

La seguretat i l'auditoria d'un servidor comencen per conèixer quins són els **usuaris existents**, **qui hi està accedint** al sistema i **quines accions realitzen** aquests usuaris. Aquest bloc recull les comandes que permeten monitoritzar usuaris connectats, revisar l'historial de connexions i detectar possibles intents d'accés no autoritzats.

**Objectiu**: Dominar les eines per auditar l'accés d'usuaris al servidor i detectar activitat sospitosa.

## Comandes d'aquest tema

1. `who` - Usuaris connectats actualment
2. `w` - Usuaris connectats i activitat detallada
3. `last` - Històric de totes les connexions
4. `lastlog` - Últim login de cada usuari del sistema
5. `lastb` - Intents de login fallits (seguretat)
6. `id` - Informació de l'usuari actual (UID, GID, grups)
7. `getent passwd`- Llistat de tots els usuaris

## 1. who - Usuaris connectats

### Descripció

La comanda `who` mostra una llista de **tots els usuaris connectats** actualment al sistema: qui són, des d'on s'han connectat, quan i per quin terminal.

### Sintaxi

```bash
who [OPCIONS]
```

### Opcions principals

| Opció           | Descripció                           |
| --------------- | ------------------------------------ |
| (sense opcions) | Llista bàsica d'usuaris              |
| `-a`            | Mostra tota la informació disponible |
| `-b`            | Data i hora de l'últim boot          |
| `-H`            | Mostra capçaleres de columnes        |
| `-q`            | Només compta usuaris (quick)         |
| `-u`            | Mostra temps d'inactivitat           |

### Exemples pràctics

**Exemple 1: Veure usuaris connectats**

```bash
who
```

**Sortida:**

```
xavi     pts/0        2024-10-20 09:15 (192.168.1.50)
root     pts/1        2024-10-20 10:30 (10.0.1.5)
alumne   pts/2        2024-10-20 11:00 (192.168.1.100)
```

**Exemple 2: Amb capçaleres**

```bash
who -H
```

**Sortida:**

```
NAME     LINE         TIME             COMMENT
xavi     pts/0        2024-10-20 09:15 (192.168.1.50)
root     pts/1        2024-10-20 10:30 (10.0.1.5)
```

**Exemple 3: Comptar usuaris connectats**

```bash
who -q
```

**Sortida:**

```
xavi root alumne
# users=3
```

**Exemple 4: Amb inactivitat**

```bash
who -u
```

**Sortida:**

```
xavi     pts/0        2024-10-20 09:15 00:05 (192.168.1.50)
root     pts/1        2024-10-20 10:30   .   (10.0.1.5)
```

### Interpretació de la sortida

**Columnes:**

- **NAME**: Nom d'usuari
- **LINE**: Terminal (pts/0, pts/1, tty1, etc.)
- **TIME**: Data i hora de connexió
- **COMMENT**: IP o hostname d'origen (entre parèntesis)

**Tipus de terminals:**

- **pts/X**: Pseudo-terminal (connexió SSH o remota)
- **tty1-6**: Consola física (raro en servidors)
- **:0**: Sessió gràfica local (Desktop)

**Inactivitat (amb `-u`):**

- **.** = Actiu recentment (menys d'1 minut)
- **00:05** = 5 minuts inactiu
- **old** = Més de 24h inactiu

## 2. w - Usuaris i activitat

### Descripció

La comanda `w` és una versió **més completa** de `who`. Mostra els usuaris connectats i **què estan fent** en aquest moment, a més de la càrrega del sistema.

### Sintaxi

```bash
w [OPCIONS] [USUARI]
```

### Opcions principals

| Opció           | Descripció                    |
| --------------- | ----------------------------- |
| (sense opcions) | Informació completa           |
| `-h`            | Sense capçalera               |
| `-s`            | Format curt                   |
| `-i`            | Mostra IP en lloc de hostname |
| `-f`            | Mostra/oculta camp FROM       |

### Exemples pràctics

**Exemple 1: Veure tots els usuaris i activitat**

```bash
w
```

**Sortida:**

```
 11:23:45 up 2 days,  3:45,  3 users,  load average: 0.15, 0.20, 0.18
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
xavi     pts/0    192.168.1.50     09:15    0.00s  0.12s  0.01s w
root     pts/1    10.0.1.5         10:30    5:00   0.05s  0.03s vim /etc/hosts
alumne   pts/2    192.168.1.100    11:00    2:30   0.08s  0.08s top
```

**Exemple 2: Només un usuari específic**

```bash
w xavi
```

**Exemple 3: Format curt (sense capçalera del sistema)**

```bash
w -h
```

**Exemple 4: Amb IPs en lloc de hostnames**

```bash
w -i
```

### Interpretació de la sortida

**Línia de capçalera:**

```
11:23:45 up 2 days, 3:45, 3 users, load average: 0.15, 0.20, 0.18
```

- **11:23:45**: Hora actual
- **up 2 days, 3:45**: Temps d'activitat del sistema (uptime)
- **3 users**: Usuaris connectats
- **load average**: Càrrega mitjana (1, 5, 15 minuts)

**Columnes d'usuaris:**

- **USER**: Nom d'usuari
- **TTY**: Terminal
- **FROM**: IP o hostname d'origen
- **LOGIN@**: Hora de connexió
- **IDLE**: Temps d'inactivitat
- **JCPU**: Temps de CPU de tots els processos del terminal
- **PCPU**: Temps de CPU del procés actual (WHAT)
- **WHAT**: Comanda que està executant ara

**Exemple d'interpretació:**

```
root     pts/1    10.0.1.5    10:30    5:00   0.05s  0.03s vim /etc/hosts
```

- Root connectat des de 10.0.1.5 a les 10:30
- Porta 5 minuts sense activitat
- Està editant `/etc/hosts` amb nano

## 3. last - Històric de connexions

### Descripció

La comanda `last` mostra l'**històric de totes les sessions** d'usuaris: qui s'ha connectat, quan, des d'on i durant quant temps. Llegeix de `/var/log/wtmp`.

### Sintaxi

```bash
last [OPCIONS] [USUARI]
```

### Opcions principals

| Opció           | Descripció                          |
| --------------- | ----------------------------------- |
| (sense opcions) | Mostra tot l'històric               |
| `-n NUM`        | Mostra només NUM línies             |
| `-a`            | Mostra hostname a la última columna |
| `-d`            | Tradueix IP a hostname              |
| `-i`            | Mostra IP en lloc de hostname       |
| `-F`            | Mostra data/hora completa           |
| `-x`            | Mostra shutdown i runlevel          |

### Exemples pràctics

**Exemple 1: Últimes 10 connexions**

```bash
last -n 10
```

**Sortida:**

```
xavi     pts/0        192.168.1.50     Mon Oct 20 09:15   still logged in
root     pts/1        10.0.1.5         Mon Oct 20 10:30   still logged in
alumne   pts/2        192.168.1.100    Mon Oct 20 11:00 - 11:45  (00:45)
xavi     pts/0        192.168.1.50     Sun Oct 19 14:20 - 18:30  (04:10)
root     pts/1        10.0.1.5         Sun Oct 19 09:00 - 12:00  (03:00)
reboot   system boot  6.8.0-45-generic Sun Oct 19 08:55   still running
```

**Exemple 2: Connexions d'un usuari específic**

```bash
last xavi
```

**Exemple 3: Connexions des d'una IP específica**

```bash
last -i | grep "192.168.1.50"
```

**Exemple 4: Reinicis del sistema**

```bash
last reboot
```

**Sortida:**

```
reboot   system boot  6.8.0-45-generic Mon Oct 20 08:00   still running
reboot   system boot  6.8.0-45-generic Sun Oct 19 08:55 - 23:59  (15:04)
reboot   system boot  6.8.0-44-generic Sat Oct 18 10:00 - 22:30  (12:30)
```

**Exemple 5: Amb data completa**

```bash
last -F -n 5
```

### Interpretació de la sortida

**Format de cada línia:**

```
xavi     pts/0        192.168.1.50     Mon Oct 20 09:15 - 11:45  (02:30)
```

- **xavi**: Usuari
- **pts/0**: Terminal
- **192.168.1.50**: IP d'origen
- **Mon Oct 20 09:15**: Data i hora de login
- **11:45**: Hora de logout
- **(02:30)**: Durada de la sessió

**Estats especials:**

- **still logged in**: Sessió activa ara
- **gone - no logout**: Sessió tancada incorrectament (crash, kill)
- **reboot**: Reinici del sistema
- **shutdown**: Aturada del sistema

**Ús pràctic:**

```bash
# Veure qui s'ha connectat avui
last | grep "$(date '+%a %b %d')"

# Comptar connexions d'un usuari
last xavi | grep -v "wtmp" | wc -l

# Connexions durant el cap de setmana
last | grep -E "Sat|Sun"
```

## 4. lastlog - Últim login de cada usuari

### Descripció

La comanda `lastlog` mostra l'**últim login** de **tots els usuaris** del sistema (inclosos els que mai s'han connectat). Llegeix de `/var/log/lastlog`.

**Diferència amb `last`:**

- `last` mostra **tot l'històric**
- `lastlog` mostra **només l'últim login de cada usuari**

### Sintaxi

```bash
lastlog [OPCIONS]
```

### Opcions principals

| Opció           | Descripció                                        |
| --------------- | ------------------------------------------------- |
| (sense opcions) | Mostra tots els usuaris                           |
| `-u LOGIN`      | Només un usuari específic                         |
| `-t DIES`       | Usuaris que han fet login en els últims X dies    |
| `-b DIES`       | Usuaris que NO han fet login en els últims X dies |

### Exemples pràctics

**Exemple 1: Últim login de tots els usuaris**

```bash
lastlog
```

**Sortida:**

```
Username         Port     From             Latest
root             pts/1    10.0.1.5         Mon Oct 20 10:30:15 +0200 2024
daemon                                     **Never logged in**
bin                                        **Never logged in**
sys                                        **Never logged in**
xavi             pts/0    192.168.1.50     Mon Oct 20 09:15:22 +0200 2024
alumne           pts/2    192.168.1.100    Mon Oct 20 11:00:45 +0200 2024
```

**Exemple 2: Només un usuari**

```bash
lastlog -u xavi
```

**Sortida:**

```
Username         Port     From             Latest
xavi             pts/0    192.168.1.50     Mon Oct 20 09:15:22 +0200 2024
```

**Exemple 3: Usuaris que han fet login en els últims 7 dies**

```bash
lastlog -t 7
```

**Exemple 4: Usuaris que NO han fet login en els últims 30 dies (inactius)**

```bash
lastlog -b 30
```

### Interpretació de la sortida

**Columnes:**

- **Username**: Nom d'usuari
- **Port**: Terminal (pts/X)
- **From**: IP o hostname d'origen
- **Latest**: Data i hora de l'últim login

**Estats:**

- **Never logged in**: L'usuari mai s'ha connectat (normal per a usuaris de sistema com `daemon`, `bin`, etc.)
- **Data específica**: Últim cop que es va connectar

**Ús pràctic per seguretat:**

```bash
# Usuaris humans que mai s'han connectat (possible problema de configuració)
lastlog | grep -E "xavi|alumne|admin" | grep "Never"

# Usuaris humans inactius més de 90 dies (candidats a desactivar)
lastlog -b 90 | grep -E "xavi|alumne|admin"

# Últim login de tots els usuaris amb shell
lastlog | grep -v "Never" | grep -v "^Username"
```

## 5. lastb - Intents de login fallits

### Descripció

La comanda `lastb` (last bad) mostra l'**històric d'intents de login fallits**. És essencial per detectar **atacs de força bruta** o intents d'accés no autoritzats. Llegeix de `/var/log/btmp`.

**Important:** Requereix permisos de root (`sudo`).

### Sintaxi

```bash
sudo lastb [OPCIONS]
```

### Opcions principals

| Opció           | Descripció                          |
| --------------- | ----------------------------------- |
| (sense opcions) | Mostra tot l'històric de fallades   |
| `-n NUM`        | Mostra només NUM línies             |
| `-a`            | Mostra hostname a la última columna |
| `-i`            | Mostra IP en lloc de hostname       |
| `-F`            | Data/hora completa                  |

### Exemples pràctics

**Exemple 1: Últims 10 intents fallits**

```bash
sudo lastb -n 10
```

**Sortida:**

```
admin    ssh:notty    185.33.50.51     Mon Oct 20 11:30 - 11:30  (00:00)
root     ssh:notty    198.51.100.88    Mon Oct 20 11:25 - 11:25  (00:00)
test     ssh:notty    185.33.50.51     Mon Oct 20 11:20 - 11:20  (00:00)
root     ssh:notty    185.33.50.51     Mon Oct 20 11:15 - 11:15  (00:00)
admin    ssh:notty    192.0.2.10       Mon Oct 20 10:50 - 10:50  (00:00)
```

**Exemple 2: Intents d'una IP específica**

```bash
sudo lastb | grep "185.33.50.51"
```

**Exemple 3: Comptar intents per usuari**

```bash
sudo lastb | awk '{print $1}' | sort | uniq -c | sort -rn
```

**Sortida:**

```
  45 root
  23 admin
  12 test
   5 user
```

**Exemple 4: Comptar intents per IP**

```bash
sudo lastb | awk '{print $3}' | sort | uniq -c | sort -rn
```

**Sortida:**

```
  35 185.33.50.51
  20 198.51.100.88
  15 192.0.2.10
```

### Interpretació de la sortida

**Format:**

```
admin    ssh:notty    185.33.50.51     Mon Oct 20 11:30 - 11:30  (00:00)
```

- **admin**: Usuari que va intentar entrar
- **ssh:notty**: Servei i tipus de terminal
- **185.33.50.51**: IP d'origen de l'atac
- **Mon Oct 20 11:30**: Data i hora de l'intent

**Patrons d'atac comuns:**

**Atac de força bruta:**

```bash
# Múltiples intents de la mateixa IP
sudo lastb | grep "185.33.50.51" | wc -l
# 50 intents → Possible atac
```

**Escaneig d'usuaris:**

```bash
# Intents amb diferents usuaris des de la mateixa IP
sudo lastb | grep "185.33.50.51" | awk '{print $1}' | sort -u
# admin, root, test, user → Està provant usuaris comuns
```

**Ús pràctic per seguretat:**

```bash
# IPs amb més de 10 intents (possibles atacants)
sudo lastb | awk '{print $3}' | sort | uniq -c | sort -rn | awk '$1 > 10'

# Intents d'avui
sudo lastb | grep "$(date '+%a %b %d')"

# Bloquejar IP amb iptables (exemple)
# sudo iptables -A INPUT -s 185.33.50.51 -j DROP
```

## 6. id - Informació de l'usuari actual

### Descripció

La comanda `id` mostra informació sobre l'**usuari actual**: UID (User ID), GID (Group ID) i els grups als quals pertany. És útil per verificar permisos i pertinença a grups.

### Sintaxi

```bash
id [OPCIONS] [USUARI]
```

### Opcions principals

| Opció           | Descripció                     |
| --------------- | ------------------------------ |
| (sense opcions) | Mostra tota la informació      |
| `-u`            | Només UID                      |
| `-g`            | Només GID principal            |
| `-G`            | Tots els GIDs                  |
| `-n`            | Mostra noms en lloc de números |
| `-un`           | Nom d'usuari                   |
| `-gn`           | Nom del grup principal         |

### Exemples pràctics

**Exemple 1: Informació de l'usuari actual**

```bash
id
```

**Sortida:**

```
uid=1000(xavi) gid=1000(xavi) groups=1000(xavi),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),122(lpadmin),134(lxd),135(sambashare)
```

**Exemple 2: Informació d'un altre usuari**

```bash
id root
```

**Sortida:**

```
uid=0(root) gid=0(root) groups=0(root)
```

**Exemple 3: Només el UID**

```bash
id -u
```

**Sortida:**

```
1000
```

**Exemple 4: Només el nom d'usuari**

```bash
id -un
```

**Sortida:**

```
xavi
```

**Exemple 5: Comprovar si un usuari està al grup sudo**

```bash
id xavi | grep sudo
```

### Interpretació de la sortida

**Format complet:**

```
uid=1000(xavi) gid=1000(xavi) groups=1000(xavi),27(sudo),46(plugdev)
```

**UID (User ID):**

- **0**: root (superusuari)
- **1-999**: Usuaris de sistema (daemons, serveis)
- **1000+**: Usuaris normals

**GID (Group ID):**

- Grup principal de l'usuari
- Normalment té el mateix nom que l'usuari

**Groups:**

- Llista de tots els grups als quals pertany
- **sudo (27)**: Pot executar comandes com a root
- **adm (4)**: Pot llegir logs del sistema
- **plugdev (46)**: Pot muntar dispositius USB

**Ús pràctic:**

```bash
# Verificar si ets root
if [ $(id -u) -eq 0 ]; then
    echo "Ets root"
else
    echo "No ets root"
fi

# Comprovar si pots usar sudo
id | grep -q sudo && echo "Tens permisos sudo" || echo "No tens sudo"

# Veure els grups d'un usuari
id -Gn xavi
```

## 7. getent passwd - Llistat complet d'usuaris

### Descripció

La comanda `getent passwd` mostra **tots els usuaris del sistema**, tant locals (de `/etc/passwd`) com d'altres fonts com LDAP, NIS o altres bases de dades configurades. És essencial per fer auditories d'usuaris i saber qui té accés al servidor.

**Diferència amb `/etc/passwd`:**

- `/etc/passwd` només mostra usuaris locals
- `getent passwd` mostra usuaris locals + externs (LDAP, etc.)

### Sintaxi

```bash
getent passwd [USUARI]
```

### Opcions principals

`getent` no té moltes opcions, però és molt potent per filtrar:

| Ús                     | Descripció                       |
| ---------------------- | -------------------------------- |
| `getent passwd`        | Tots els usuaris                 |
| `getent passwd USUARI` | Informació d'un usuari específic |
| `getent group`         | Tots els grups                   |
| `getent group GRUP`    | Membres d'un grup                |

### Exemples pràctics

**Exemple 1: Llistar tots els usuaris**

```bash
getent passwd
```

**Sortida (parcial):**

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
xavi:x:1000:1000:Xavi Admin:/home/xavi:/bin/bash
alumne:x:1001:1001:Alumne Pràctiques:/home/alumne:/bin/bash
mysql:x:115:120:MySQL Server,,,:/nonexistent:/bin/false
```

**Exemple 2: Informació d'un usuari específic**

```bash
getent passwd xavi
```

**Sortida:**

```
xavi:x:1000:1000:Xavi Profe:/home/xavi:/bin/bash
```

**Exemple 3: Només usuaris humans (UID >= 1000)**

```bash
getent passwd | awk -F: '$3 >= 1000 {print $1, "UID:"$3}'
```

**Sortida:**

```
xavi UID:1000
alumne UID:1001
nobody UID:65534
```

**Exemple 4: Usuaris amb shell de login (poden connectar-se)**

```bash
getent passwd | grep -v "nologin\|false" | awk -F: '{print $1, $7}'
```

**Sortida:**

```
root /bin/bash
xavi /bin/bash
alumne /bin/bash
```

**Exemple 5: Comptar usuaris del sistema**

```bash
# Total d'usuaris
getent passwd | wc -l

# Usuaris humans
getent passwd | awk -F: '$3 >= 1000 && $3 != 65534 {print $1}' | wc -l

# Usuaris de sistema
getent passwd | awk -F: '$3 < 1000 {print $1}' | wc -l
```

### Interpretació de la sortida

**Format de cada línia:**

```
xavi:x:1000:1000:Xavi Profe:/home/xavi:/bin/bash
```

**Camps separats per `:`**

1. **xavi**: Nom d'usuari (login)
2. **x**: Placeholder de contrasenya (està a `/etc/shadow`)
3. **1000**: UID (User ID)
4. **1000**: GID (Group ID principal)
5. **Xavi Profe**: GECOS (nom complet, comentaris)
6. **/home/xavi**: Directori home
7. **/bin/bash**: Shell per defecte

**Interpretació dels camps:**

**UID (User ID):**

- **0**: root (superusuari)
- **1-999**: Usuaris de sistema (daemons, serveis)
- **1000+**: Usuaris normals/humans
- **65534**: nobody (usuari sense privilegis)

**Shell:**

- **/bin/bash**: Pot fer login interactiu
- **/usr/sbin/nologin**: No pot fer login
- **/bin/false**: No pot fer login (alternativa)
- **/bin/sh**: Shell bàsic

**Home directory:**

- **/home/usuari**: Usuari normal
- **/root**: Root
- **/var/lib/servei**: Usuaris de sistema
- **/nonexistent**: Sense directori (usuaris de servei)

### Exemples amb getent group

**Exemple 1: Llistar tots els grups**

```bash
getent group
```

**Exemple 2: Veure membres del grup sudo**

```bash
getent group sudo
```

**Sortida:**

```
sudo:x:27:xavi,alumne
```

**Exemple 3: Trobar tots els grups d'un usuari**

```bash
getent group | grep xavi
```

### Ús pràctic per seguretat

**Detectar usuaris amb shell sospitós:**

```bash
# Usuaris amb UID < 1000 però amb shell de login
getent passwd | awk -F: '$3 < 1000 && $7 ~ /bash|sh$/ {print $1, "UID:"$3, "Shell:"$7}'
```

**Llistar usuaris sense home directory:**

```bash
getent passwd | awk -F: '!system("test -d "$6) {print $1, $6}'
```

**Trobar usuaris duplicats (mateix UID):**

```bash
getent passwd | awk -F: '{print $3}' | sort | uniq -d
```

**Auditoria d'usuaris amb privilegis:**

```bash
# Usuaris al grup sudo
getent group sudo | cut -d: -f4 | tr ',' '\n'

# Usuaris amb UID 0 (hauria de ser només root)
getent passwd | awk -F: '$3 == 0 {print $1}'
```

### Combinació amb altres comandes

**Comparar usuaris del sistema amb els que s'han connectat:**

```bash
# Usuaris que existeixen
getent passwd | awk -F: '$3 >= 1000 && $3 != 65534 {print $1}' > usuaris-sistema.txt

# Usuaris que han fet login
lastlog | grep -v "Never" | grep -v "Username" | awk '{print $1}' > usuaris-login.txt

# Usuaris que mai s'han connectat
comm -23 <(sort usuaris-sistema.txt) <(sort usuaris-login.txt)
```

## Cas d'ús real: Detectar un intent d'intrusió

### Escenari

És dilluns al matí i veus un correu del sistema d'alertes que indica activitat SSH sospitosa durant el cap de setmana. Has d'investigar què ha passat.

### Investigació pas a pas

**1. Comprovar qui està connectat ara**

```bash
w
```

Resultat: Només tu estàs connectat. Cap sessió sospitosa activa.

**2. Revisar l'històric de connexions del cap de setmana**

```bash
last | grep -E "Sat|Sun"
```

Resultat: Veus connexions normals dels administradors.

**3. Revisar intents de login fallits**

```bash
sudo lastb -n 50
```

Resultat: **Detectes 45 intents fallits des de la IP 185.33.50.51**

**4. Analitzar l'atac en detall**

```bash
# Comptar intents totals d'aquesta IP
sudo lastb | grep "185.33.50.51" | wc -l
# Resultat: 120 intents

# Veure quins usuaris ha provat
sudo lastb | grep "185.33.50.51" | awk '{print $1}' | sort | uniq -c | sort -rn
# Resultat:
#   50 root
#   30 admin
#   20 user
#   15 test
#    5 backup
```

**Conclusió:** Atac de força bruta intentant usuaris comuns.

**5. Verificar si algun intent ha tingut èxit**

```bash
last | grep "185.33.50.51"
```

Resultat: Cap connexió exitosa. L'atac ha fallat.

**6. Accions a prendre**

**Bloquejar la IP:**

```bash
sudo iptables -A INPUT -s 185.33.50.51 -j DROP
sudo iptables-save > /etc/iptables/rules.v4
```

**Revisar contrasenyes dels usuaris comuns:**

```bash
# Forçar canvi de contrasenya
sudo passwd -e admin
sudo passwd -e backup
```

**Instal·lar fail2ban (prevenir futurs atacs):**

```bash
sudo apt install fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

**Documentar l'incident:**

- IP atacant: 185.33.50.51
- Data: Cap de setmana (Sat-Sun)
- Usuaris objectiu: root, admin, user, test, backup
- Resultat: Atac fallit, IP bloquejada
