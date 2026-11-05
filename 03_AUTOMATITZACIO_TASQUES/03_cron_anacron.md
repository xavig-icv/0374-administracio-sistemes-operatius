## 03. Tasques Periòdiques: `cron` i `anacron`

`cron` és el planificador clàssic dels sistemes UNIX: el dimoni `cron` comprova cada minut si hi ha alguna tasca programada per a aquell moment i l'executa. A Ubuntu està instal·lat i en marxa per defecte (`systemctl status cron`).

### El `crontab` d'un Usuari

Cada usuari té la seva taula de tasques (*crontab*). Es gestiona amb l'ordre `crontab`:

| Ordre                     | Funció                                                         |
| ------------------------- | -------------------------------------------------------------- |
| `crontab -e`              | Edita les tasques de l'usuari (la primera vegada demana l'editor) |
| `crontab -l`              | Llista les tasques                                             |
| `crontab -r`              | **Esborra totes** les tasques, sense preguntar                 |
| `sudo crontab -e`         | Edita les tasques de root                                      |
| `sudo crontab -u anna -l` | Llista les tasques d'un altre usuari                           |

Els crontabs es desen a `/var/spool/cron/crontabs/`, però no s'han d'editar directament: `crontab -e` comprova la sintaxi en desar.

### Format d'una Línia

```
┌───────────── minut (0-59)
│ ┌─────────── hora (0-23)
│ │ ┌───────── dia del mes (1-31)
│ │ │ ┌─────── mes (1-12)
│ │ │ │ ┌───── dia de la setmana (0-7; 0 i 7 = diumenge, 1 = dilluns)
│ │ │ │ │
* * * * *  ordre
```

| Operador | Significat               | Exemple         | Vol dir                                |
| -------- | ------------------------ | --------------- | -------------------------------------- |
| `*`      | Tots els valors          | `* * * * *`     | Cada minut                             |
| `,`      | Llista                   | `0 8,14 * * *`  | A les 8:00 i a les 14:00               |
| `-`      | Interval                 | `0 9 * * 1-5`   | A les 9:00, de dilluns a divendres     |
| `/`      | Cada N                   | `*/15 * * * *`  | Cada 15 minuts                         |

**Exemples:**

```
# m   h   dm  mes ds  ordre
*/5   *   *   *   *   /usr/local/bin/vigila_serveis.sh
0     2   *   *   *   /usr/local/bin/copia_etc.sh >> /var/log/copies.log 2>&1
30    7   *   *   1-5 /usr/local/bin/informe-srv0374.sh
0     0   1   *   *   /usr/local/bin/neteja_temporals.sh
@reboot               /usr/local/bin/avis_arrencada.sh
```

| Paraula clau | Equivalent    |
| ------------ | ------------- |
| `@reboot`    | En arrencar el sistema |
| `@hourly`    | `0 * * * *`   |
| `@daily`     | `0 0 * * *`   |
| `@weekly`    | `0 0 * * 0`   |
| `@monthly`   | `0 0 1 * *`   |

`Nota:` Si es restringeixen alhora el dia del mes i el dia de la setmana, la tasca s'executa quan es compleix **qualsevol** dels dos: `0 9 1 * 1` s'executa el dia 1 de cada mes **i** cada dilluns.

### L'Entorn de `cron`

Les tasques de `cron` no s'executen com a la terminal:

| Element         | Valor a `cron`                     | Conseqüència                                                 |
| --------------- | ---------------------------------- | ------------------------------------------------------------ |
| Shell           | `/bin/sh`                          | La línia del crontab no admet sintaxi pròpia de bash. Dins dels scripts, el *shebang* `#!/bin/bash` sí que funciona |
| `PATH`          | `/usr/bin:/bin`                    | Ordres de `/usr/sbin` (`useradd`, `logrotate`...) no es troben: rutes absolutes |
| Directori       | El directori personal de l'usuari  | Rutes relatives incorrectes: rutes absolutes                 |
| Sortida         | S'envia per correu a l'usuari      | Sense servidor de correu es perd: cal redirigir-la           |
| Caràcter `%`    | Té un significat especial (salt de línia) | Cal escriure'l com `\%`: `date +\%F`                   |

Al principi del crontab es poden definir variables per a totes les tasques:

```
PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
MAILTO=""
```

`MAILTO=""` desactiva l'enviament de correus (útil si la sortida ja es desa en fitxers).

### Tasques del Sistema

A més dels crontabs dels usuaris, hi ha les tasques del sistema, que porten un camp més: **l'usuari** amb què s'executen.

| Ubicació                      | Ús                                                                       |
| ----------------------------- | ------------------------------------------------------------------------ |
| `/etc/crontab`                | Crontab principal del sistema                                            |
| `/etc/cron.d/`                | Un fitxer per aplicació o tasca. **Recomanat** per a les tasques pròpies del servidor |
| `/etc/cron.hourly/`, `cron.daily/`, `cron.weekly/`, `cron.monthly/` | Scripts que s'executen cada hora, dia, setmana o mes |

```
# /etc/cron.d/copies - Còpies de seguretat de srv0374
PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
# m  h  dm mes ds  usuari  ordre
0    2  *  *   *   root    /usr/local/bin/copia_etc.sh
```

Els scripts dels directoris `cron.daily/`... els executa l'ordre `run-parts`, que segueix una regla important: **el nom del fitxer només pot tenir lletres, números, guions i guions baixos**. Un script anomenat `copia.sh` (amb punt) **no s'executarà**. La mateixa regla s'aplica als fitxers de `/etc/cron.d/`.

```bash
run-parts --test /etc/cron.daily        # mostra quins scripts s'executarien
```

### Registres de `cron`

```bash
journalctl -u cron --since today
grep CRON /var/log/syslog
```

```
Nov 05 02:00:01 srv0374 CRON[3021]: (root) CMD (/usr/local/bin/copia_etc.sh)
Nov 05 02:00:02 srv0374 CRON[3020]: (CRON) info (No MTA installed, discarding output)
```

La segona línia indica que la tasca ha escrit alguna cosa per pantalla i, com que no hi ha servidor de correu, s'ha perdut. Cal redirigir la sortida.

### Evitar Execucions Solapades: `flock`

Si una tasca programada cada 5 minuts tarda més de 5 minuts, s'executarà dues vegades alhora. `flock` ho impedeix amb un fitxer de bloqueig:

```
*/5 * * * *  flock -n /tmp/vigila.lock /usr/local/bin/vigila_serveis.sh
```

Amb `-n`, si l'execució anterior encara no ha acabat, la nova es descarta.

### Control d'Accés

`/etc/cron.allow` i `/etc/cron.deny` funcionen com els d'`at`: si existeix `cron.allow`, només hi poden programar tasques els usuaris que hi apareixen.

### `anacron`: Equips que no Estan Sempre Engegats

`cron` executa les tasques a l'hora exacta: si l'equip està apagat en aquell moment, la tasca no es fa. `anacron` està pensat per a equips que s'apaguen (portàtils, ordinadors de les aules): comprova cada dia si les tasques diàries, setmanals o mensuals s'han executat i, si no, les executa en engegar-se.

```bash
sudo apt install anacron                 # a Ubuntu Desktop ja ve instal·lat
cat /etc/anacrontab
```

```
# període  retard  identificador  ordre
1          5       cron.daily     run-parts --report /etc/cron.daily
7          10      cron.weekly    run-parts --report /etc/cron.weekly
@monthly   15      cron.monthly   run-parts --report /etc/cron.monthly
```

| Camp            | Significat                                                      |
| --------------- | --------------------------------------------------------------- |
| Període         | Cada quants dies s'ha d'executar (o `@monthly`)                 |
| Retard          | Minuts que espera després d'engegar-se (per no carregar l'arrencada) |
| Identificador   | Nom de la tasca; la data de l'última execució es desa a `/var/spool/anacron/` |

Quan `anacron` està instal·lat, `/etc/crontab` li deixa els directoris `cron.daily`, `cron.weekly` i `cron.monthly`, i és ell qui els executa.

### Resolució de Problemes

| Problema                                                  | Causa                                                    | Solució                                                     |
| --------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| L'script funciona a mà però no des de `cron`              | `PATH` reduït o rutes relatives                          | Rutes absolutes o definir `PATH` al crontab                 |
| La tasca no deixa cap rastre                              | La sortida s'ha perdut (`No MTA installed`)              | `>> fitxer.log 2>&1` o `logger` dins de l'script            |
| `date +%F` dins del crontab no funciona                   | `%` és un caràcter especial a `cron`                     | `date +\%F`                                                 |
| Un script de `/etc/cron.daily/` no s'executa              | El nom té un punt (`copia.sh`) o no és executable        | Canviar-li el nom (`copia`) i `chmod +x`                    |
| Una línia de `/etc/cron.d/` no s'executa                  | Falta el camp de l'usuari                                | `0 2 * * * root /ruta/script`                               |
| `crontab -r` ha esborrat totes les tasques                | `-r` no demana confirmació                               | Fer una còpia abans: `crontab -l > crontab.bak`             |
