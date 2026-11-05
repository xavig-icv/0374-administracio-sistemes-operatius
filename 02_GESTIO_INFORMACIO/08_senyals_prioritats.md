## 08. Senyals, Prioritats i Control de Tasques

Aquest fitxer presenta com **actuar** sobre els processos: acabar-los, aturar-los i reprendre'ls amb senyals, gestionar tasques en segon pla i canviar la prioritat amb què fan servir la CPU.

| Ordre                     | Funció                                                  |
| ------------------------- | ------------------------------------------------------- |
| `kill`                    | Envia un senyal a un procés pel seu PID                 |
| `killall`                 | Envia un senyal a tots els processos amb un nom         |
| `pkill`                   | Envia un senyal als processos que coincideixen amb un patró |
| `jobs`, `fg`, `bg`        | Gestionen les tasques en segon pla d'un shell           |
| `nohup`                   | Executa una ordre que sobreviu al tancament de la sessió |
| `nice`, `renice`          | Fixen o canvien la prioritat d'un procés                |

### Senyals

| Núm.    | Nom                    | Acció per defecte            | Ús habitual                                                               |
| ------- | ---------------------- | ---------------------------- | ------------------------------------------------------------------------- |
| 1       | `SIGHUP`               | Acabar                       | S'ha tancat el terminal. Molts serveis el fan servir per **recarregar la configuració** |
| 2       | `SIGINT`               | Acabar                       | `Ctrl+C`                                                                  |
| 3       | `SIGQUIT`              | Acabar i bolcar la memòria   | `Ctrl+\`                                                                  |
| 9       | `SIGKILL`              | Acabar                       | Acaba el procés a la força. **No es pot capturar ni ignorar**             |
| 10 / 12 | `SIGUSR1` / `SIGUSR2`  | Acabar                       | Cada programa en defineix l'ús                                            |
| 15      | `SIGTERM`              | Acabar                       | Petició **educada** d'acabar. És el senyal per defecte de `kill`          |
| 17      | `SIGCHLD`              | Ignorar                      | El kernel avisa el pare que un fill ha acabat                             |
| 18      | `SIGCONT`              | Continuar                    | Reprèn un procés aturat                                                   |
| 19      | `SIGSTOP`              | Aturar                       | Atura el procés. No es pot capturar ni ignorar                            |
| 20      | `SIGTSTP`              | Aturar                       | `Ctrl+Z`                                                                  |

```bash
kill -l                                             # llista de tots els senyals
kill -l | grep -oE "[0-9]+\) SIG(HUP|INT|KILL|TERM)\b"   # només els més importants
kill -l 15                                          # nom del senyal 15: TERM
```

**`SIGTERM` o `SIGKILL`?**

| `SIGTERM` (15)                                              | `SIGKILL` (9)                                                    |
| ----------------------------------------------------------- | ---------------------------------------------------------------- |
| El procés el rep i pot acabar ordenadament                  | El kernel elimina el procés immediatament                        |
| Desa les dades, tanca fitxers i connexions, esborra temporals | No pot fer res: es poden perdre dades o quedar fitxers bloquejats |
| El procés el pot ignorar                                    | No es pot ignorar                                                |

Per això el procediment correcte és **primer `SIGTERM` i només si no funciona, `SIGKILL`**:

```bash
kill 2101                          # 1. demanar-li que acabi (SIGTERM)
sleep 5; ps -p 2101                # 2. comprovar si encara hi és
kill -9 2101                       # 3. només si continua viu
```

### `kill`: Enviar un Senyal per PID

```bash
sleep 300 &                        # [1] 2101: número de tasca i PID
echo $!                            # PID de l'últim procés en segon pla
PID=$!                             # desar-lo en una variable

kill $PID                          # SIGTERM (per defecte)
kill -9 $PID                       # SIGKILL (també: kill -KILL, kill -s KILL)
kill 2101 2102 2103                # diversos PID alhora
kill $(pgrep sleep)                # tots els PID que retorna pgrep
kill -0 $PID && echo "Encara viu"  # comprova si existeix sense enviar cap senyal
```

**Aturar i reprendre un procés:**

```bash
sleep 300 &
PID=$!
ps -o pid,stat,cmd -p $PID         # S (esperant)
kill -STOP $PID
ps -o pid,stat,cmd -p $PID         # T (aturat)
kill -CONT $PID
ps -o pid,stat,cmd -p $PID         # S (torna a funcionar)
```

**`SIGHUP`:** l'acció per defecte és acabar. Un `sleep` que rep `SIGHUP` mor. En canvi, molts dimonis (`sshd`, `rsyslogd`, `nginx`) el capturen per tornar a llegir la configuració sense aturar-se: això és el que fa `systemctl reload` en molts serveis. Quan es tanca una sessió, el shell envia `SIGHUP` a les seves tasques; per això les tasques en segon pla moren en desconnectar-se (vegeu `nohup` més avall).

### `killall`: Per Nom de Procés

`killall` envia el senyal a **tots** els processos amb un nom exacte.

| Opció         | Funció                                                    |
| ------------- | --------------------------------------------------------- |
| `-i`          | Interactiu: demana confirmació per a cada procés          |
| `-u usuari`   | Només els processos d'un usuari. Sense nom: tots els seus processos |
| `-w`          | Espera que els processos hagin acabat                     |
| `-v`          | Informa de cada senyal enviat                             |
| `-s SENYAL`   | Senyal a enviar (`-s KILL` o `-KILL`)                     |

```bash
killall sleep                      # tots els sleep (que pugui acabar l'usuari)
killall -i sleep                   # preguntant un per un
sudo killall -u asixprofe          # tots els processos d'asixprofe
```

```
Kill sleep(2101) ? (y/N) y
Kill sleep(2102) ? (y/N) y
Kill sleep(2103) ? (y/N) n
```

`Nota:` A Linux, `killall` és del paquet `psmisc`. En altres sistemes UNIX, com Solaris, `killall` acaba **tots** els processos del sistema. No el feu servir en sistemes que no conegueu.

### `pkill`: Per Patró

`pkill` selecciona els processos amb les **mateixes opcions que `pgrep`** ([07](./07_veure_processos.md)): nom, línia d'ordre (`-f`), usuari (`-u`), terminal (`-t`), el més nou (`-n`) o el més antic (`-o`). Abans d'executar `pkill`, comproveu amb `pgrep` què coincideix.

```bash
pgrep -af "http.server 9000"       # 1. què coincideix?
pkill -f "http.server 9000"        # 2. acabar només el servidor del port 9000
pkill -f http.server               # tots els servidors http.server
pkill -n sleep                     # només el sleep més recent
pkill -t pts/1                     # els processos del terminal pts/1
sudo pkill -KILL -u asixprofe      # tots els processos d'un usuari, a la força
```

**Compte amb els patrons massa generals:** `pkill -f python` acabaria també serveis del sistema escrits en Python, com `networkd-dispatcher` o `unattended-upgrades`.

| Ordre      | Selecciona per                                       | Provar abans amb   |
| ---------- | ---------------------------------------------------- | ------------------ |
| `kill`     | PID                                                  | `ps -p PID`        |
| `killall`  | Nom exacte del procés                                | `pgrep -x nom`     |
| `pkill`    | Patró: nom, línia d'ordre, usuari, terminal...       | `pgrep` amb les mateixes opcions |

**Un `bash` interactiu ignora `SIGTERM`.** Per això `pkill -t pts/1` o `sudo killall -u asixprofe` acaben els programes de la sessió, però la sessió continua oberta. Per tancar-la cal `SIGHUP` o `SIGKILL` (`sudo pkill -KILL -t pts/1`) o, amb systemd, `sudo loginctl terminate-user asixprofe`.

### Tasques en Primer i Segon Pla

Cada shell porta la llista de les seves **tasques** (*jobs*): ordres llançades en segon pla o aturades.

| Ordre / tecla | Funció                                                           |
| ------------- | ---------------------------------------------------------------- |
| `ordre &`     | Executa l'ordre en segon pla                                     |
| `Ctrl+Z`      | Atura la tasca de primer pla (`SIGTSTP`)                         |
| `Ctrl+C`      | Interromp la tasca de primer pla (`SIGINT`)                      |
| `jobs -l`     | Llista les tasques del shell amb el seu PID                      |
| `fg %n`       | Porta la tasca `n` al primer pla                                 |
| `bg %n`       | Continua la tasca `n` (aturada) en segon pla                     |
| `kill %n`     | Envia `SIGTERM` a la tasca `n`                                   |

```
$ sleep 300 &
[1] 2101
$ sleep 400
^Z
[2]+  Stopped                 sleep 400
$ jobs -l
[1]-  2101 Running                 sleep 300 &
[2]+  2102 Stopped                 sleep 400
$ bg %2
[2]+ sleep 400 &
$ fg %1
sleep 300
^C
```

`+` marca la tasca actual (la que fan servir `fg` i `bg` si no s'indica cap número) i `-`, l'anterior. Les tasques són de cada shell: des d'un altre terminal no apareixen a `jobs` (cal buscar-les amb `ps` o `pgrep`).

### `nohup`: Sobreviure al Tancament de la Sessió

Quan es tanca una sessió SSH, les tasques del shell reben `SIGHUP` i acaben. Una còpia de seguretat de dues hores llançada amb `&` s'aturaria en desconnectar-se. `nohup` fa que l'ordre ignori `SIGHUP`:

```bash
nohup ./copia-seguretat.sh &
# nohup: ignoring input and appending output to 'nohup.out'

nohup ./copia-seguretat.sh > copia.log 2>&1 &    # sortida a un fitxer concret
```

| Situació                                             | Solució                                                     |
| ---------------------------------------------------- | ----------------------------------------------------------- |
| Abans d'executar l'ordre                             | `nohup ordre &`                                             |
| La tasca ja està en marxa                            | `disown -h %1` (el shell no li enviarà `SIGHUP`)           |
| Cal tornar a veure el terminal més tard              | `tmux`: `tmux new -s feina`, desconnectar amb `Ctrl+B d` i recuperar amb `tmux attach -t feina` |
| La tasca s'ha d'executar sempre o a una hora concreta | Un servei de systemd ([09](./09_serveis_systemd.md)) o una tasca programada (Tema 3) |

### Prioritats: `nice` i `renice`

El valor ***nice*** indica com d'"amable" és un procés amb els altres: com més alt, més CPU cedeix.

| Valor *nice* | Prioritat       | Ús                                                    |
| ------------ | --------------- | ----------------------------------------------------- |
| `-20`        | Màxima          | Processos crítics (només root)                        |
| `0`          | Normal          | Valor per defecte (s'hereta del pare)                 |
| `19`         | Mínima          | Tasques de manteniment: còpies, compressions, indexacions |

**Permisos:** qualsevol usuari pot **augmentar** el *nice* dels seus processos (baixar-ne la prioritat). Només root pot posar valors negatius o tornar a baixar un *nice* que ja s'ha augmentat.

| Ordre                              | Funció                                               |
| ---------------------------------- | ---------------------------------------------------- |
| `nice -n 10 ordre`                 | Executa l'ordre amb *nice* 10                        |
| `sudo nice -n -10 ordre`           | Executa l'ordre amb *nice* -10 (més prioritat)       |
| `renice -n 10 -p PID`              | Canvia el *nice* d'un procés en marxa                |
| `sudo renice -n 5 -u asixprofe`    | Canvia el *nice* de tots els processos d'un usuari   |
| `ps -o pid,ni,cmd -p PID`          | Mostra el *nice* (columna `NI`)                      |

A `top`, la columna `NI` és el *nice* i `PR` la prioritat interna (`PR = 20 + NI`).

```
$ nice -n -10 sleep 300
nice: cannot set niceness: Permission denied
```

Un usuari normal no pot demanar més prioritat. `nice` avisa, però **executa l'ordre igualment amb *nice* 0**.

**Comprovar l'efecte del *nice*:** la prioritat només es nota quan dos processos **competeixen per la mateixa CPU**. Si la VM té 2 CPU i llanceu dos processos, cadascun en tindrà una per a ell sol i tots dos arribaran al 100%. Amb `taskset -c 0` es fixen tots dos a la CPU 0:

```bash
taskset -c 0 sha256sum /dev/zero &             # nice 0
taskset -c 0 nice -n 19 sha256sum /dev/zero &  # nice 19, a la mateixa CPU
top                                            # comparar la columna %CPU (sortir amb q)
pkill sha256sum
```

El procés amb *nice* 0 s'endurà gairebé tota la CPU 0 i el de *nice* 19, una part molt petita.

### Resolució de Problemes

| Problema                                                    | Causa                                                         | Solució                                               |
| ----------------------------------------------------------- | ------------------------------------------------------------- | ----------------------------------------------------- |
| `kill: (2101) - Operation not permitted`                    | El procés és d'un altre usuari                                | `sudo kill 2101`                                      |
| `kill: (2101) - No such process`                            | El procés ja ha acabat o el PID és incorrecte                 | Comprovar-ho amb `pgrep` o `ps`                       |
| El procés no acaba amb `kill`                               | Captura o ignora `SIGTERM`                                    | Esperar uns segons i, si cal, `kill -9`               |
| No acaba ni amb `kill -9`                                   | Està en estat `D` (esperant el disc o la xarxa) o és un zombi | `D`: resoldre el problema de disc/xarxa o reiniciar. Zombi: actuar sobre el pare |
| Després de `pkill -t pts/1` la sessió continua oberta       | El `bash` interactiu ignora `SIGTERM`                         | `pkill -KILL -t pts/1` o `pkill -HUP -t pts/1`        |
| `nice: cannot set niceness: Permission denied`              | Només root pot donar més prioritat                            | `sudo nice -n -10 ...`                                |
| `renice: failed to set priority ... Permission denied`      | Un usuari no pot tornar a baixar el *nice*                    | `sudo renice ...`                                     |
| El *nice* no canvia el `%CPU` dels processos                | Cada procés té una CPU per a ell sol                          | Fixar-los a la mateixa CPU amb `taskset -c 0`         |
| Les tasques en segon pla moren en tancar la sessió SSH      | Reben `SIGHUP`                                                | `nohup`, `disown -h` o `tmux`                         |
