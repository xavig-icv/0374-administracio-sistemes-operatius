# 02. Gestió de Processos

## Introducció

Un cop hem après com funcionen els processos, identificar-los al sistema i entendre la seva informació, ara hem de saber com **gestionar-los**: com s'aturen processos que fallen, com ajustar les prioritats per optimitzar el rendiment del sistema, i comunicar-nos amb els processos mitjançant senyals del sistema predefinides. En aquest bloc treballem les comandes per gestionar els diferents processos.

**Objectiu**: Conèixer les comandes per la gestió pràctica de processos en entorns de producció.

## Comandes d'aquest tema

### Senyals i terminació:

1. `kill` - Enviar senyals a processos
2. `killall` - Finalitzar processos per nom
3. `pkill` - Finalitzar processos amb criteris avançats

### Gestió de prioritats:

4. `nice` - Executar comanda amb prioritat modificada
5. `renice` - Canviar prioritat d'un procés existent

## Part 1: Senyals i Terminació de Processos

### Què són els senyals?

Els **senyals** són notificacions que s'envien a un procés per indicar-li que faci alguna cosa. Són el mecanisme principal de comunicació entre el sistema (o usuaris) i els processos.

**Analogia:** És com trucar al timbre d'una casa. Segons com toquis (quin senyal envies), la persona farà una cosa diferent.

### Taula de senyals principals

| Senyal      | Número | Tecla   | Acció                 | Pot ser ignorat? | Quan s'utilitza                |
| ----------- | ------ | ------- | --------------------- | ---------------- | ------------------------------ |
| **SIGTERM** | 15     | -       | Terminació educada    | Sí               | Per defecte, permet netejar    |
| **SIGKILL** | 9      | -       | Matar forçosament     | No               | Últim recurs, mort immediata   |
| **SIGINT**  | 2      | Ctrl+C  | Interrupció           | Sí               | Cancel·lar comanda interactiva |
| **SIGQUIT** | 3      | Ctrl+\  | Quit amb core dump    | Sí               | Debug, genera fitxer core      |
| **SIGHUP**  | 1      | -       | Hang up               | Sí               | Recarregar configuració        |
| **SIGSTOP** | 19     | Ctrl+Z  | Pausar                | No               | Aturar temporalment            |
| **SIGCONT** | 18     | -       | Continuar             | -                | Reprendre procés pausat        |
| **SIGUSR1** | 10     | -       | Definit per usuari    | Sí               | Accions personalitzades        |
| **SIGUSR2** | 12     | -       | Definit per usuari    | Sí               | Accions personalitzades        |
| **SIGCHLD** | 17     | -       | Fill ha canviat estat | Sí               | Notificar al pare              |

### SIGTERM vs SIGKILL: Diferències clau

#### SIGTERM (15) - "El que hem de fer servir en primer lloc"

Acció "educada" : Si us plau, tanca't

**Característiques:**

- Senyal **educada** de terminació
- El procés **pot capturar-la** i decidir què fer
- Permet al procés **netejar** abans de morir:
  - Tancar fitxers oberts
  - Guardar dades
  - Alliberar recursos
  - Tancar connexions de xarxa

**Quan utilitzar-lo:**

- **Sempre com a primera opció.** És la manera correcta
- Serveis com Apache, MySQL, PostgreSQL
- Scripts que han obert fitxers
- Aplicacions que gestionen dades

**Risc:**
El procés pot **ignorar** el senyal si està programat per fer-ho.

#### SIGKILL (9) - "El que hem de fer servir com a últim recurs"

Acció "no educada": Finalitza (mor) ara mateix!

**Característiques:**

- Mata el procés **immediatament**
- **No pot ser capturada ni ignorada**
- El kernel mata el procés directament
- **No hi ha oportunitat** de netejar

**Quan utilitzar-lo:**

- Després d'intentar SIGTERM (i esperar 5-10 segons)
- Processos **penjats** que no responen
- **Emergències** (servidor col·lapsat o respon molt lentament)

**Riscos:**

- Pot **corrompre dades** (bases de dades, fitxers)
- Deixar **recursos bloquejats** (fitxers, sockets)
- **Perdre informació** no emmagatzemada (que està a la RAM)

**Regla/Norma general:** SIGTERM primer, SIGKILL només si no queda altra opció.

### Flux recomanat per matar un procés

```bash
# 1. Intent educat (SIGTERM)
kill 1234
echo "Esperant 5 segons..."
sleep 5

# 2. Comprovar si ha mort
if ps -p 1234 > /dev/null 2>&1; then
  echo "El procés no respon, forçant terminació..."
  kill -9 1234
  echo "Procés eliminat amb SIGKILL"
else
  echo "Procés finalitzat correctament"
fi
```

## 1. kill - Enviar senyals a processos

### Descripció

La comanda `kill` envia **senyals** a processos indicant el seu **PID**. Tot i el nom, no només mata processos, també ens permet enviar qualsevol senyal.

### Sintaxi

```bash
kill [OPCIÓ] PID [PID...]
```

### Opcions principals

| Opció               | Descripció                                            |
| ------------------- | ----------------------------------------------------- |
| `kill PID`          | Envia SIGTERM (per defecte)                           |
| `kill -9 PID`       | Envia SIGKILL (mata immediatament)                    |
| `kill -15 PID`      | Envia SIGTERM (explícit)                              |
| `kill -SIGTERM PID` | Igual que -15                                         |
| `kill -SIGKILL PID` | Igual que -9                                          |
| `kill -l`           | Llista tots els senyals disponibles                   |
| `kill -HUP PID`     | Envia SIGHUP (recarregar config com systemctl reload) |

### Exemples pràctics

**Exemple 1: Finalitzar un procés (educadament)**

```bash
kill 1234
```

**Exemple 2: Finalitzar un procés immediatament (a la força)**

```bash
kill -9 1234
# O
kill -SIGKILL 1234
```

**Exemple 3: Recarregar la configuració d'un server (SIGHUP)**

```bash
# Nginx després de modificar la configuració
sudo kill -HUP $(pidof nginx | awk '{print $1}')
# O millor:
sudo systemctl reload nginx
```

**Exemple 4: Llistar senyals disponibles**

```bash
kill -l
```

**Sortida:**

```
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL
 5) SIGTRAP      6) SIGABRT      7) SIGBUS       8) SIGFPE
 9) SIGKILL     10) SIGUSR1     11) SIGSEGV     12) SIGUSR2
13) SIGPIPE     14) SIGALRM     15) SIGTERM     16) SIGSTKFLT
...
```

**Exemple 5: Enviar SIGUSR1 (senyal personalitzat)**

S'utilitza pels definir que ha de fer el programa o script. Mostrar informació detallada del procés, fer una rotació dels fitxers de log, recarregar la configuració del meu servei personalitzat, guardar l'estat del programal disc dur, escriure informació als fitxers de log, etc.

```bash
kill -USR1 1234
```

**Exemple 6: Pausar un procés**

```bash
kill -STOP 1234
# Per reprendre:
kill -CONT 1234
```

**Exemple 7: Finalitzar múltiples processos**

```bash
kill 1234 1235 1236
# O tots els PIDs d'un servei:
kill $(pgrep apache2)
```

### Verificar si un procés ha mort

```bash
# Opció 1: Amb ps
if ps -p 1234 > /dev/null; then
  echo "Encara viu"
else
  echo "Ha mort"
fi

# Opció 2: Comprovant el codi de sortida
ps -p 1234 > /dev/null 2>&1
if [ $? -eq 0 ]; then
  echo "Encara viu"
else
  echo "Ha mort"
fi
```

## 2. killall - Finalitzar processos per nom

### Descripció

`killall` envia senyals a **tots els processos** que coincideixin amb un **nom**. És més adequat que `kill` si vols finalitzar tots els processos d'un servei concret.

**Important:** `killall` és perillós! Coneix tots els processos actius abans d'utilitzar-lo.

Si fas `killall python` finalitzarà tots els processos de python del programa que vols finalitzar però també afectarà a altres serveis en execució que fan servir python com aplicacions web que executen python, scripts de manteniment, etc.

### Sintaxi

```bash
killall [OPCIONS] NOM_PROCÉS
```

### Opcions principals

| Opció                   | Descripció                                        |
| ----------------------- | ------------------------------------------------- |
| `killall NOM`           | Envia SIGTERM a tots els processos amb aquest nom |
| `killall -9 NOM`        | Envia SIGKILL                                     |
| `killall -u USUARI`     | Mata tots els processos d'un usuari               |
| `killall -i NOM`        | Mode interactiu (demana confirmació)              |
| `killall -w NOM`        | Espera que els processos morin                    |
| `killall -s SENYAL NOM` | Envia senyal específic                            |

### Exemples pràctics

**Exemple 1: Finalitza tots els processos d'Apache**

```bash
sudo killall apache2
```

**Exemple 2: Finalitza immediatament**

```bash
sudo killall -9 apache2
```

**Exemple 3: Finalitza processos d'un usuari**

```bash
sudo killall -u xavi
```

**Exemple 4: Mode interactiu (és més segur perquè demana confirmació)**

```bash
killall -i firefox
# Kill firefox(1234) ? (y/N) y
# Kill firefox(1235) ? (y/N) n
```

**Exemple 5: Esperar que finalitzin**

```bash
killall -w apache2
echo "Tots els processos Apache han finalitzat"
```

**Exemple 6: Enviar SIGHUP a tots els nginx**

```bash
sudo killall -HUP nginx
```

### Precaucions amb killall

**Perill:** Si et confons de nom, pots matar processos importants!

**Exemple perillós:**

```bash
# TERRIBLE - Pot matar processos del sistema si hi ha coincidència
sudo killall init   # NO FER AIXÒ!
sudo killall systemd # AIXÒ TAMPOC!
```

**Recomanació:** Primer comprova quins processos mataràs:

```bash
# 1. Veure quins processos hi ha
ps aux | grep apache2

# 2. Si estàs segur, matar
sudo killall apache2
```

## 3. pkill - Finalitzar amb criteris avançats

### Descripció

La comanda `pkill` és com `killall` però **molt més potent**. Permet cercar processos per:

- Nom (com killall)
- Usuari
- Grup
- Terminal
- Sessió
- Comanda completa (arguments inclosos)

**Avantatge:** És com combinar `pgrep` (cercar) amb `kill` (matar).

### Sintaxi

```bash
pkill [OPCIONS] PATRÓ
```

### Opcions principals

| Opció             | Descripció                                              |
| ----------------- | ------------------------------------------------------- |
| `pkill PATRÓ`     | Envia SIGTERM a processos que coincideixin amb un patró |
| `pkill -9 PATRÓ`  | Envia SIGKILL                                           |
| `pkill -u USUARI` | Mata processos d'un usuari concret                      |
| `pkill -t TTY`    | Mata processos d'un terminal especificat                |
| `pkill -f PATRÓ`  | Cerca a la comanda completa (no només el nom)           |
| `pkill -x NOM`    | Coincidència exacta del nom                             |
| `pkill -n`        | Mata només el procés més recent                         |
| `pkill -o`        | Mata només el procés més antic                          |

### Exemples pràctics

**Exemple 1: Finalitzar processos per nom**

```bash
pkill apache
# Igual que: killall apache
```

**Exemple 2: Finalitzar processos d'un usuari**

```bash
sudo pkill -u xavi
```

**Exemple 3: Finalitzar per comanda completa**

```bash
# Matar tots els Python que executin server.py
pkill -f "python.*server.py"
```

**Exemple 4: Finalitzar processos d'un terminal específic**

```bash
# Veure terminals
who
# xavi pts/0

# Matar tots els processos d'aquest terminal
sudo pkill -t pts/0
```

**Exemple 5: Finalitzar només el més recent**

```bash
pkill -n firefox
# Mata només l'última finestra de Firefox oberta
```

**Exemple 6: Finalitzar només el més antic**

```bash
pkill -o apache
# Mata el procés pare d'Apache (el primer)
```

**Exemple 7: Coincidència exacta**

```bash
pkill -x sshd
# Mata "sshd" però no "sshd: user@pts/0"
```

**Exemple 8: Enviar SIGHUP**

```bash
sudo pkill -HUP -f nginx
```

### Diferències: kill vs killall vs pkill

| Característica     | kill                      | killall              | pkill                    |
| ------------------ | ------------------------- | -------------------- | ------------------------ |
| **Identifica per** | PID                       | Nom exacte           | Patró/criteris           |
| **Flexibilitat**   | Baixa                     | Mitjana              | Alta                     |
| **Seguretat**      | Alta (PID únic)           | Mitjana              | Baixa (patró ampli)      |
| **Ús comú**        | Matar un procés específic | Matar servei per nom | Cerca avançada           |
| **Exemple**        | `kill 1234`               | `killall apache2`    | `pkill -f "python.*app"` |

**Recomanació:**

- `kill` - Quan saps el PID exacte
- `killall` - Quan vols matar tots els processos d'un servei
- `pkill` - Quan vols fer una cerca avançada (una part del nom, un usuari concret, etc.)

### Precaucions amb pkill

**Molt perillós:** Els patrons massa genèrics poden matar processos inesperats!

**Exemple d'ús perillós:**

```bash
# TERRIBLE - Pot matar molts processos
sudo pkill python   # Mata TOTS els Python (inclosos scripts del sistema!)
```

**Millor:**

```bash
# CORRECTE - S'ha de ser específic
sudo pkill -f "python /opt/app/server.py"
```

**Sempre comprova abans:**

```bash
# 1. Veure què mataràs
pgrep -f "python.*app" -l

# 2. Si estàs segur, matar
pkill -f "python.*app"
```

## Part 2: Gestió de Prioritats

### Concepte de prioritat

Linux assigna **una prioritat** a cada procés per decidir quant de temps de CPU (en mil·lisegons) se li assigna. Els processos amb prioritat més alta obtenen més temps de CPU per la seva execució.

**Problema:** 100 processos volen executar-se, però només hi ha 4 nuclis (cores).  
**Situació:** Hi ha 96 processos esperant a la cua el seu torn.
**Solució:** El planificador els ordena per prioritat.

### Nice value

Cada procés disposa d'un **nice value** que determina la seva prioritat:

- **Rang:** -20 (prioritat més alta) a +19 (prioritat més baixa)
- **Per defecte:** 0 (prioritat normal)

**Regla:** Com més "nice" (amable) és un procés, menys prioritat disposa.

**Permisos:**

- Qualsevol usuari pot **reduir** la prioritat (augmentant el nice: 0 → 19)
- Només l'usuari **root** pot **augmentar** la prioritat (reduïnt el nice: 0 → -20)

### Visualitzar la prioritat

**Amb ps:**

```bash
ps -eo pid,ni,cmd
```

**Sortida:**

```
  PID  NI CMD
    1   0 /sbin/init
 1234   0 /usr/sbin/apache2
 5678  10 /usr/bin/backup.sh
 9012 -10 /usr/bin/critical-app
```

**La columna NI:** NIce value del procés

**Amb top:**

```bash
top
# Columna "NI" mostra nice value
```

## 4. nice - Executar per modificar la prioritat

### Descripció

La comanda `nice` permet executar una comanda amb un **nice value** específic. Permet iniciar processos amb prioritat baixa o alta des del principi.

### Sintaxi

```bash
nice [OPCIÓ] COMANDA
```

### Opcions principals

| Opció                   | Descripció                             |
| ----------------------- | -------------------------------------- |
| `nice COMANDA`          | Executa amb nice +10 (prioritat baixa) |
| `nice -n VALOR COMANDA` | Executa amb nice específic             |
| `nice -n 19 COMANDA`    | Prioritat mínima (més amable)          |
| `nice -n -20 COMANDA`   | Prioritat màxima (només root)          |

### Exemples pràctics

**Exemple 1: Executar un backup amb baixa prioritat**

```bash
nice -n 19 ./backup.sh
# El backup no molestarà altres processos
```

**Exemple 2: Comprimir un fitxer amb baixa prioritat**

```bash
nice -n 15 tar -czf backup.tar.gz /data/
```

**Exemple 3: Prioritat per defecte (nice +10)**

```bash
nice ./script-llarg.sh
```

**Exemple 4: Alta prioritat (només ho pot fer root)**

```bash
sudo nice -n -10 ./script-important.sh
```

**Exemple 5: Verificar el nice value**

```bash
nice -n 5 sleep 100 &
ps -o pid,ni,cmd | grep sleep
```

**Sortida:**

```
 5678   5 sleep 100
```

### Descripció

`renice` modifica el **nice value** d'un procés que **ja està executant-se**. Permet ajustar prioritats en temps d'execució.

### Sintaxi

```bash
renice [OPCIÓ] VALOR -p PID
```

### Opcions principals

| Opció                    | Descripció                                    |
| ------------------------ | --------------------------------------------- |
| `renice VALOR -p PID`    | Canvia nice del PID                           |
| `renice VALOR -u USUARI` | Canvia nice de tots els processos de l'usuari |
| `renice VALOR -g GRUP`   | Canvia nice de tots els processos del grup    |

### Exemples pràctics

**Exemple 1: Reduir la prioritat d'un procés**

```bash
# Veure nice actual
ps -o pid,ni,cmd -p 1234

# Reduir prioritat (augmentar nice)
renice +10 -p 1234

# Verificar
ps -o pid,ni,cmd -p 1234
```

**Exemple 2: Augmentar la prioritat (només ho pot fer root)**

```bash
sudo renice -10 -p 1234
```

**Exemple 3: Canviar la prioritat de tots els processos d'un usuari**

```bash
sudo renice +5 -u www-data
```

**Exemple 4: Canviar la prioritat de múltiples processos**

```bash
renice +10 -p 1234 -p 1235 -p 1236
```

**Exemple 5: Restaurar la prioritat (establir-la al valor per defecte)**

```bash
renice 0 -p 1234
```

### Limitacions de renice

**Usuaris normals:**

- **Només poden augmentar** nice (reduir prioritat) Com més amabilitat menys prioritat
- **No poden reduir** nice (augmentar prioritat)
- **No poden** modificar processos d'altres usuaris

**Exemple:**

```bash
# Tens un procés amb nice 5
renice +10 -p 1234   # OK - Augmenta nice a 15
renice 0 -p 1234     # ERROR - No pots reduir nice a 0

# Només root pot fer-ho:
sudo renice 0 -p 1234   # OK
```
