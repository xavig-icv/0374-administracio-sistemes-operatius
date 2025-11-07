# 02. Gestió de Processos

## Introducció

La gestió de processos és una tasca importantíssima de l'administració de sistemes Linux. Cada aplicació, servei o comanda que s'executa al servidor és considerada un procés. Saber com funcionen els processos, com gestionar-los, assignar-lis una prioritat i solucionar problemes és essencial per mantenir un servidor operatiu, eficient i segur.

Aquest bloc es centra en conèixer **els processos**, **com es gestionen** i **com resoldre problemes** que puguin ocasionar aquests processos.

## Per què és tan important?

Imagina aquestes situacions reals que trobaràs com a administrador de sistemes:

### Escenari 1: Servidor col·lapsat

> És divendres a les 17:30h. El servidor web va extremadament lent. Els clients es queixen que la pàgina no carrega. El teu cap et pregunta: "Què està passant?"

**Necessites:**

- Identificar ràpidament quin procés consumeix tota la CPU
- Decidir si "matar-lo", reiniciar-lo o ajustar la seva prioritat
- Documentar l'incident per evitar que torni a passar

### Escenari 2: Servei que no arrenca

> Després d'un reinici del servidor, l'aplicació principal no funciona. Intentes iniciar el servei associat manualment però sembla que arrenca i després s'atura. No hi ha errors evidents.

**Necessites:**

- Veure l'estat del procés i per què s'atura automàticament
- Analitzar els logs (registres) per identificar el problema
- Entendre la seqüència d'arrencada i dependències

### Escenari 3: Processos zombi

> Executant `top` (una comanda de monitorització) veus que hi ha 50 processos zombi. No saps què són, si són perillosos o com eliminar-los.

**Necessites:**

- Entendre què és un procés zombie
- Saber si representa un problema
- Aprendre com prevenir-los i gestionar-los

### Escenari 4: Execució en background

> Has d'executar un script per fer un backup o una tasca que trigarà 2-3 hores. No pots deixar el terminal obert tot aquest temps perquè has de desconnectar.

**Necessites:**

- Saber com executar el procés en background
- Assegurar que continuï si tanques la sessió SSH
- Poder consultar l'estat després

### Escenari 5: Atac de seguretat

> Detectes un procés desconegut que està saturant el servidor (molt trànsit de xarxa i CPU). Sospites que el servidor ha estat compromès.

**Necessites:**

- Identificar els processos sospitosos
- Analitzar què estan fent aquests processos
- Aturar-los de manera segura i investigar

## Conceptes fonamentals de processos

### Què és un procés?

Un **procés** és una instància en execució d'un programa. Quan executes qualsevol comanda o aplicació, el sistema crea un procés.

**Exemple pràctic:**

```bash
# Quan executes la comanda:
firefox

# El sistema crea un procés de Firefox amb:
# - Un PID únic (Process ID): 1234
# - Memòria assignada
# - Temps de CPU assignat
# - Fitxers oberts
# - Permisos de l'usuari que l'ha executat
```

**Diferència programa vs procés:**

- **Programa**: Fitxer executable o binari guardat en disc (`/usr/bin/firefox`)
- **Procés**: Programa executant-se a memòria (Firefox obert amb PID 1234)

Un mateix programa pot tenir múltiples processos:

```bash
# Si obres Firefox 3 vegades:
# - 3 processos diferents
# - 3 PIDs únics (1234, 1235, 1236)
# - 1 únic programa (/usr/bin/firefox)
```

### Tipus de processos

#### 1. Processos d'usuari

Iniciats per usuaris, ja sigui directament o a través d'aplicacions.

**Exemples:**

- Navegador web (`firefox`)
- Editor de text (`nano`)
- Script que executes (`./backup.sh`)
- Comandes del terminal (`ls`, `grep`, `cp`)

**Característiques:**

- S'executen amb permisos de l'usuari
- Finalitzen (moren) quan l'usuari tanca la sessió (si no s'utilitzen eines com `nohup`)
- Consumeixen recursos segons la seva funció

#### 2. Processos de sistema

Essencials pel funcionament del sistema operatiu. S'inicien durant l'arrencada del servidor.

**Exemples:**

- `systemd` (PID 1) - És el primer procés, pare de tots
- `kthreadd` (PID 2) - Gestiona threads del kernel
- `ksoftirqd` - Gestiona interrupcions de software
- `kworker` - Workers del kernel

**Característiques:**

- S'executen amb permisos d'administrador (root)
- Són crítics pel correcte funcionament sistema
- No es poden finalitzar (matar) sense consequències greus

#### 3. Processos dimoni (daemons)

Serveis que s'executen en segon pla (background) sense una interfície ni interacció directa amb usuaris.

**Exemples a Ubuntu Server:**

- `sshd` - Servei SSH per connexions remotes
- `apache2` - Servidors web
- `mysqld` - Servidor de base de dades
- `cron` - Planificador de tasques
- `rsyslogd` - Servei de registre d'esdeveniments (logs)

**Característiques:**

- El seu nom normalment acaba amb la lletra "d" (daemon)
- No disposen d'un terminal associat (TTY = ?)
- Alguns s'inicien amb el sistema i s'executen indefinidament
- Reben peticions i donen resposta als usuaris i altres programes

**Com identificar-los:**

```bash
ps aux | grep "?"
# Processos amb TTY = ? són dimonis
```

### Estats dels processos

Un procés passa per diferents estats durant el seu cicle de vida:

```
         [NOU]
           |
           | Crear procés (fork/exec)
           v
       [PREPARAT] <----------+
           |                 |
           | Planificador    | Temps esgotat
           | assigna CPU     | o prioritat major
           v                 |
      [EXECUTANT] -----------+
           |                 |
           |                 | Esperar I/O
           v                 | o recurs
      [BLOQUEJAT] -----------+
           |
           | I/O completat
           |
           v
       [ACABAT]
```

#### **Nou (New)**

El procés s'està creant. El sistema li està assignant recursos.

**Exemple:** Acabes d'executar `firefox` però encara no s'ha obert la finestra.

#### **Preparat (Ready)**

El procés està llest per executar-se però ha d'esperar a que el planificador li assigni temps de CPU.

**Exemple pràctic:**

```
Tens un servidor amb 4 CPUs (nuclis)
10 processos volen executar-se
6 processos tindran un estat de "Preparat" esperant el seu torn
```

**A l'executar `ps` apareixeran com:** `R` (Running)

#### **Executant-se (Running)**

El procés disposa d'accés a la CPU i està actualment executant instruccions a la CPU.

**Important:** En un sistema amb 4 nuclis, només poden haver-hi un màxim de 4 processos en aquest estat simultàniament.

**Exemple:**

```bash
# Veure processos executant-se ara
# Al vostre (0374 Server) li heu d'assignar 2CPUs.
ps aux | grep " R "
```

#### **Bloquejat (Sleeping / Waiting)**

El procés espera que hi hagi un esdeveniment extern abans de continuar.

**Motius comuns:**

- **Esperar la lectura/escriptura de disc**
  - Exemple: `cat fitxer_gran.txt` espera que el disc llegeixi les dades
- **Esperar a dades que ha d'obtenir de la xarxa**
  - Exemple: `curl https://api.web.cat` espera la resposta del servidor
- **Esperar l'entrada de dades d'usuari**
  - Exemple: `read variable` espera que escriguis alguna cosa
- **Esperar que un altre procés alliberi un recurs (bloquejat)**
  - Exemple: Dos processos volen escriure al mateix fitxer

**A l'executar `ps` apareix com:**

- `S` - Es pot interrompre - sleep (es pot despertar amb senyals)
- `D` - No es pot interrompre - sleep (esperant I/O crític, no es pot interrompre)

**Exemple pràctic:**

```bash
# La majoria de processos estan en "Sleep"
ps aux
# Veuràs molts processos amb estat "S"
```

#### **Acabat (Terminated / Zombie)**

El procés ha finalitzat però encara ocupa una entrada a la taula de processos.

**Per què passa això?**
El procés fill ha mort però el pare encara no ha llegit (coneix) el seu estat de sortida.

**Processos Zombie:**

```bash
ps aux | grep "Z"
# Veuràs processos amb <defunct> o Z
```

**Exemple:**

```
usuari   1234  0.0  0.0      0     0 ?        Z    10:00   0:00 [apache2] <defunct>
```

**Són perillosos?**

- **No consumeixen CPU ni memòria** (només una entrada a la taula)
- **Problema:** Si n'hi ha molts, poden omplir la taula de processos
- **Solució:** El procés pare ha de "recollir-los" (eliminar-los de la taula) o reiniciar el pare si mai "recull"

### Cicle de vida complet d'un procés

**Exemple real: Executar un backup**

```bash
./backup.sh
```

1. **Nou**: El sistema crea el procés, assigna PID 1234
2. **Preparat**: El procés espera torn a la cua de la CPU
3. **Executant**: El backup comença a comprimir fitxers
4. **Bloquejat**: Espera que el disc escrigui les dades
5. **Preparat**: Disc ha acabat, torna a la cua (espera torn)
6. **Executant**: Continua comprimint
7. **(repeteix aquestes accions varies vegades)**
8. **Acabat**: Backup complet, procés finalitza

**Temps total:** 30 minuts  
**Temps real de CPU:** Potser 2 minuts (la resta esperant al disc)

## Diferències: Procés vs Thread vs Job

### Procés

**Definició:** Unitat bàsica d'execució amb el seu propi espai de memòria aïllat.

**Característiques:**

- Memòria independent (no compartida amb altres processos)
- PID únic
- La comunicació entre processos és costosa (IPC - Inter-Process Communication)
- Crear un procés és relativament lent

**Exemple:**

```bash
# Cada comanda és un procés diferent
firefox &     # PID 1234
chrome &      # PID 1235
# No comparteixen memòria
```

**Avantatges:**

- **Aïllament:** Si un procés falla, no afecta els altres
- **Seguretat:** No poden accedir a la memòria d'altres processos

**Desavantatges:**

- **Cost:** Crear i canviar entre processos és lent
- **Comunicació:** Compartir dades és un procés complex

### Thread (Fil d'execució)

**Definició:** Unitat d'execució dins d'un procés que comparteix memòria amb altres threads del mateix procés.

**Analogia:** Si un procés és una casa, els threads són les persones que hi viuen. Comparteixen recursos (cuina, bany, cadires) però fan tasques diferents.

**Característiques:**

- Comparteix memòria amb altres threads del procés
- Més lleuger i ràpid de crear
- Comunicació eficient (memòria compartida)
- Si un thread falla, pot afectar tot el procés

**Exemple real: Servidor web Apache**

```
Apache (procés principal)
├─ Thread 1: Atén al Client A
├─ Thread 2: Atén al Client B
├─ Thread 3: Atén al Client C
└─ Thread 4: Atén al Client D

Tots comparteixen:
- Configuració del servidor
- Memòria caché
- Connexions a base de dades
```

**Avantatges:**

- **Rendiment:** Crear threads és molt ràpid
- **Eficiència:** Compartir dades és fàcil i ràpid

**Desavantatges:**

- **Risc:** Un thread pot corrompre la memòria de tot el procés
- **Complexitat:** Programar amb threads és més difícil

### Job (Treball)

**Definició:** Concepte de la shell que agrupa un o més processos relacionats (normalment connectats per pipes).

**Característiques:**

- És una abstracció de la shell (bash, zsh)
- Permet gestionar múltiples processos com una unitat
- Útil per controlar processos en background/foreground

**Exemple:**

```bash
# Aquest és un JOB que conté 3 processos
cat access.log | grep "ERROR" | wc -l &

[1] 5678  # Job número 1, PID del primer procés: 5678
```

**Gestió de jobs:**

```bash
jobs              # Llistar jobs actius
fg %1             # Portar job 1 a foreground (primer pla)
bg %1             # Continuar job 1 en background
kill %1           # Finalitzar tot el job
Ctrl+Z            # Pausar job en foreground
```

**Exemple pràctic:**

```bash
# Iniciar un procés llarg en background
./backup.sh &
[1] 5678

# Continuar treballant al terminal

# Veure l'estat
jobs
[1]+  Running     ./backup.sh &

# Si cal, portar-lo a primer pla
fg %1
```

### Comparativa

| Característica        | Procés                   | Thread                        | Job                |
| --------------------- | ------------------------ | ----------------------------- | ------------------ |
| **Memòria**           | Independent              | Compartida                    | N/A                |
| **Velocitat creació** | Lenta                    | Ràpida                        | N/A                |
| **Comunicació**       | Costosa (IPC)            | Eficient                      | N/A                |
| **Aïllament**         | Alt (segur)              | Baix (risc)                   | N/A                |
| **Gestió**            | `ps`, `kill`             | Dins del procés               | `jobs`, `fg`, `bg` |
| **Ús típic**          | Aplicacions independents | Tasques paral·leles d'una app | Gestió de shell    |

## Prioritats i planificació

### Com decideix Linux quin procés s'executa?

Linux utilitza un **planificador per torns** (gestor de processos) que decideix quin procés té accés a la CPU i durant quant temps.

**Problema:** 100 processos volen executar-se però només hi ha 4 CPUs.

**Solució:** El planificador els organitza per **prioritat** i els proporciona un **temps de CPU per torns**.

### Nice value (Prioritat)

Cada procés disposa d'un **nice value** que determina la seva prioritat d'accés a la CPU.

**Rang:** -20 (prioritat més alta) a +19 (prioritat més baixa)  
**Per defecte:** 0 (prioritat normal)

**Regla:** Com més "nice" (amable) sigui un procés, menys prioritat té (deixa la CPU als altres).

**Permisos:**

- Qualsevol usuari pot **reduir** prioritat (augmentar nice: 0 → 19)
- Només **root** pot **augmentar** prioritat (reduir nice: 0 → -20)

**Exemple pràctic:**

```bash
# Executar un backup amb baixa prioritat (no molesti altres processos)
nice -n 19 ./backup.sh

# Executar scripts importants amb alta prioritat (només root)
sudo nice -n -10 ./script_critic.sh

# Canviar prioritat d'un procés existent
renice -n 10 -p 5678   # Reduir prioritat del PID 5678
```

**Quan s'utilitza:**

- **nice 19:** Tasques de manteniment (backups, compressions)
- **nice 0:** Serveis normals (Apache, MySQL)
- **nice -20:** Processos crítics del sistema (només si és urgent)

### Quantum de temps (Time slice)

El planificador no deixa un procés executar-se indefinidament. Li dona un **quantum** (porció de temps).

**Valor típic:** 10-100 mil·lisegons

**Funcionament:**

```
1. Procés A s'executa durant 20ms
2. Temps esgotat → Procés A passa a "Preparat"
3. Procés B comença a executar-se durant 20ms
4. El cicle continua...
```

**Per què es fa això?**

- **Equitat:** Tots els processos reben temps de CPU
- **Resposta:** El sistema sembla que fa moltes coses alhora
- **Interactivitat:** Les aplicacions responen ràpidament

**Nota:** Els processos amb prioritat més alta reben quantums més llargs.

### CFS (Completely Fair Scheduler)

Linux modern utilitza el **CFS**, que intenta donar temps de CPU de manera **proporcional** a cada procés.

**Concepte:** Si tens 4 processos i 1 CPU:

- Cada procés hauria de rebre aproximadament un 25% del temps
- Si un té nice -10 i altres nice 0, el -10 rep més

**Avantatge:** Els processos interactius (terminal, editor, navegador) responen ràpidament fins i tot amb càrrega alta.

## Senyals: Comunicació amb processos

### Què són els senyals?

Els **senyals** són notificacions que envia el sistema (o un usuari) a un procés per indicar-li que faci alguna cosa.

**Analogia:** És com trucar al timbre o a la porta d'una casa. Depenent de com toquis (senyal), la persona farà una cosa diferent.

### Senyals més importants

| Senyal        | Número | Acció                | Quan s'utilitza                                          |
| ------------- | ------ | -------------------- | -------------------------------------------------------- |
| **SIGTERM**   | 15     | Terminació correcta  | Per defecte, permet al procés netejar abans de morir     |
| **SIGKILL**   | 9      | Matar forçosament    | Quan SIGTERM no funciona, es fa una terminació immediata |
| **SIGHUP**    | 1      | Hang up (penjar)     | Recarregar la configuració sense reiniciar               |
| **SIGINT**    | 2      | Interrupció (Ctrl+C) | Cancel·lar la comanda interactiva                        |
| **SIGSTOP**   | 19     | Pausar el procés     | Aturar temporalment (no es pot ignorar)                  |
| **SIGCONT**   | 18     | Continuar            | Reprendre un procés pausat                               |
| **SIGUSR1/2** | 10/12  | Definit per l'usuari | Accions personalitzades de l'aplicació                   |

### SIGTERM vs SIGKILL: Diferències crítiques

#### **SIGTERM (15) - "Si us plau, tanca't"**

**Quina acció realitza?**

- Demana "educadament" al procés que finalitzi (terminació correcta i per defecte)
- El procés pot capturar el senyal
- Pot netejar recursos abans de morir (tancar fitxers, guardar dades)

**Exemple:**

```bash
kill 5678        # Per defecte envia SIGTERM
kill -15 5678    # Igual que l'anterior
```

**Quan s'utilitza:**

- **Sempre hauria de ser la primera opció** És la manera correcta
- Serveis com Apache, MySQL (necessiten tancar connexions)
- Scripts que han obert fitxers

**Què pot passar:**
El procés pot **ignorar** el senyal si està programat per fer-ho.

#### **SIGKILL (9) - "Finalitza ara mateix!"**

**Quan s'utilitza:**

- Per matar el procés immediatament
- **No es pot capturar ni ignorar**
- El procés no té oportunitat de netejar

**Exemple:**

```bash
kill -9 5678
kill -SIGKILL 5678
```

**Quan s'utilitza:**

- Després d'intentar SIGTERM (espera 5-10 segons)
- Processos penjats que no responen
- Emergències (forats de seguretat, etc.)

**Risc:**

- Pots corrompre dades (fitxers oberts, bases de dades)
- Deixar recursos bloquejats
- **S'hauria d'utilitzar com a últim recurs**

#### **Flux recomanat:**

```bash
# 1. Intent educat
kill 5678
sleep 5

# 2. Comprovar si ha mort
ps -p 5678 > /dev/null 2>&1
if [ $? -eq 0 ]; then
  # Encara viu, força la finalització
  echo "El procés no respon, matant amb SIGKILL..."
  kill -9 5678
fi
```

### SIGHUP (1) - Recarregar la configuració

Molts serveis poden **recarregar la configuració** sense reiniciar amb SIGHUP.

**Exemple pràctic:**

```bash
# Has modificat /etc/nginx/nginx.conf
# En lloc de reiniciar (perd connexions actives):
sudo systemctl reload nginx

# Internament fa:
kill -HUP $(pidof nginx)
```

**Serveis que suporten SIGHUP:**

- Apache (`apache2`)
- Nginx (`nginx`)
- SSH (`sshd`)
- Rsyslog (`rsyslogd`)

**Avantatge:** No es perden connexions actives, actualització que manté el servei en marxa.

### SIGINT (2) - Ctrl+C

**Quan s'utilitza:**

Per cancel·lar una comanda interactiva al terminal.

**Exemple:**

```bash
ping google.com
# Prem Ctrl+C
# Envia SIGINT al procés ping
```

**Programàticament:**

```bash
kill -INT 5678
kill -2 5678
```

## Processos problemàtics

### Processos Zombie

**Definició:** Procés que ha acabat però el seu pare no ha "recollit" el seu estat de sortida.

**Com identificar-los:**

```bash
ps aux | grep "Z"
# O
ps aux | grep defunct
```

**Exemple de sortida:**

```
USER     PID  %CPU %MEM   VSZ  RSS TTY  STAT START   TIME COMMAND
www-data 5678  0.0  0.0     0    0 ?    Z    10:00   0:00 [apache2] <defunct>
```

**Característiques:**

- **Estat:** Z (Zombie)
- **Memòria:** 0 (no consumeix recursos)
- **CPU:** 0% (no fa res)
- Apareix com `<defunct>`

**Són perillosos?**

- **No** consumeixen CPU ni RAM
- **Problema:** Ocupen una entrada a la taula de processos
- Si n'hi ha centenars, poden esgotar PIDs disponibles

**Com eliminar-los:**

1. **No pots matar un zombie** (`kill -9` no funciona perquè ja estan morts jajajaja)
2. **Solució:** Matar o reiniciar el **procés pare**
3. El pare recollirà els fills morts (duríssim)

**Exemple:**

```bash
# Trobar el pare del zombie (parent pid)
ps -o ppid= -p 5678
# Sortida: 1234 (PID del pare)

# Enviar SIGCHLD al pare (li recorda que reculli fills)
kill -SIGCHLD 1234

# Si no funciona, reiniciar el pare
sudo systemctl restart apache2
```

**Prevenir zombies:**
Els programes ben desenvolupats no deixen zombies. Si tens molts zombies, el software disposa d'un bug.

### Processos Orfes

**Definició:** Procés fill que el pare ha mort.

**Què passa?**
El sistema automàticament **reassigna** l'orfa a **systemd (PID 1)**, que passa a ser el seu nou pare.

**Exemple:**

```bash
# Pare (PID 1234) crea fill (PID 5678)
# El pare mor
# Fill (5678) és "adoptat" per systemd (PID 1)

ps -o ppid= -p 5678
# Sortida: 1 (ara el pare és systemd)
```

**És un problema?**

- **No** normalment
- systemd recull tots els orfes
- És un comportament normal del sistema

### Processos Penjats (Hung)

**Definició:** Procés que no respon però no ha mort.

**Símptomes:**

- Estat `D` (Uninterruptible sleep)
- No respon a SIGTERM
- Consumeix recursos però no fa progressos

**Causes comuns:**

- Esperant I/O d'un disc defectuós
- Deadlock (espera un recurs que mai arribarà)
- Bug del kernel o driver

**Com detectar-los:**

```bash
# Processos en estat D
ps aux | awk '$8 ~ /D/ {print $0}'
```

**Com gestionar-los:**

1. **Esperar:** A vegades es resolen sols (I/O acaba)
2. **SIGKILL:** Pot funcionar (però sovint no)
3. **Reiniciar el servei:** Si és un dimoni
4. **Últim recurs:** Reiniciar el servidor

## Sistema de fitxers /proc/

### Què és /proc/?

`/proc/` és un **sistema de fitxers virtual** que no existeix al disc. El kernel el genera en temps d'execució amb informació sobre processos i el sistema.

**Ubicació:** `/proc/`

**Contingut:**

```bash
ls /proc/
# Veuràs que hi ha directoris amb números (PIDs) i fitxers de sistema
1/  2/  5678/  cpuinfo  meminfo  version  ...
```

### Informació per PID

Cada procés disposa d'un directori `/proc/[PID]/`:

```bash
ls /proc/5678/
# cmdline  cwd  environ  exe  fd/  maps  stat  status  ...
```

**Fitxers importants:**

| Fitxer    | Contingut                                         |
| --------- | ------------------------------------------------- |
| `cmdline` | Comanda completa utilitzada per iniciar el procés |
| `cwd`     | Directori actual del procés (enllaç simbòlic)     |
| `environ` | Variables d'entorn                                |
| `exe`     | Executable (enllaç simbòlic)                      |
| `fd/`     | File descriptors (fitxers oberts)                 |
| `status`  | Estat detallat (memòria, threads, etc.)           |
| `maps`    | Mapes de memòria                                  |
| `limits`  | Límits de recursos                                |

### Exemples pràctics

**Veure quina comanda està executant un procés:**

```bash
cat /proc/5678/cmdline
# /usr/bin/python3/opt/app/server.py
```

**Veure des d'on s'està executant:**

```bash
ls -l /proc/5678/cwd
# lrwxrwxrwx 1 user user 0 Oct 20 10:00 /proc/5678/cwd -> /opt/app
```

**Veure els fitxers oberts:**

```bash
ls -l /proc/5678/fd/
# 0 -> /dev/pts/0 (stdin)
# 1 -> /dev/pts/0 (stdout)
# 2 -> /dev/pts/0 (stderr)
# 3 -> socket:[12345] (connexió de xarxa)
# 4 -> /var/log/app.log
```

**Veure l'ús de la memòria:**

```bash
cat /proc/5678/status | grep -i mem
# VmSize:   123456 kB  (memòria virtual total)
# VmRSS:     45678 kB  (memòria física resident)
```

### Informació del sistema

`/proc/` també disposa d'informació global del sistema:

```bash
cat /proc/cpuinfo       # Informació de la CPU
cat /proc/meminfo       # Memòria i swap
cat /proc/version       # Versió del kernel
cat /proc/uptime        # Temps d'activitat
cat /proc/loadavg       # Càrrega mitjana
cat /proc/net/dev       # Estadístiques de xarxa
```

## Seqüència d'arrencada del sistema

### Què passa quan arrenques el servidor?

```
1. BIOS/UEFI
   └─> Comprova el hardware i carrega el bootloader

2. Bootloader (GRUB)
   └─> Carrega el kernel de Linux

3. Kernel
   └─> Inicialitza hardware i drivers
   └─> Munta el sistema de fitxers arrel (/)
   └─> Executa /sbin/init (o systemd)

4. systemd (PID 1)
   └─> El primer procés del sistema
   └─> Pare de tots els altres processos
   └─> Gestiona l'arrencada de serveis

5. Serveis i dimonis
   └─> systemd inicia serveis segons les dependències
   └─> SSH, Apache, MySQL, etc.

6. Login
   └─> Sistema llest per acceptar connexions
```

### systemd: El gestor de sistema modern

**systemd** (PID 1) és el primer procés i controla tot el sistema.

**Funcions:**

- Iniciar serveis en paral·lel (arrencada ràpida)
- Gestionar dependències entre serveis
- Monitoritzar i reiniciar serveis caiguts
- Gestionar logs (journald)
- Gestionar sockets, timers, muntatges

**Conceptes clau:**

- **Unit:** Qualsevol recurs gestionat per systemd (servei, socket, timer...)
- **Target:** Grup de units (similar a runlevels)
- **Service:** Un dimoni/servei (Apache, MySQL, SSH...)

**Exemples bàsics:**

```bash
# Veure estat d'un servei
systemctl status ssh

# Iniciar/aturar serveis
systemctl start apache2
systemctl stop apache2

# Activar a l'arrencada
systemctl enable mysql

# Veure serveis actius
systemctl list-units --type=service --state=running
```

**Nota:** Els detalls de systemd es veuran al **Bloc 03 - Gestió de Serveis**.

## Dimonis (Daemons)

### Què són els dimonis?

Processos que s'executen en **background** (segon pla) de manera contínua, proporcionant serveis.

**Característiques:**

- Nom acaba normalment amb la lletra "d" (sshd, mysqld, httpd)
- No tenen terminal (TTY = ?)
- S'inicien normalment amb el sistema
- S'executen com a root o l'usuari dedicat (www-data, etc.)
- Reben peticions i donen respostes

### Dimonis comuns a Ubuntu Server

| Dimoni              | Funció                  | Port    |
| ------------------- | ----------------------- | ------- |
| `sshd`              | Connexions SSH remotes  | 22      |
| `apache2` / `nginx` | Servidor web            | 80, 443 |
| `mysqld`            | Base de dades MySQL     | 3306    |
| `cron`              | Planificador de tasques | -       |
| `rsyslogd`          | Sistema de logs         | -       |
| `systemd-journald`  | Logs de systemd         | -       |
| `NetworkManager`    | Gestió de xarxa         | -       |

### Com identificar dimonis

```bash
# Processos sense terminal
ps aux | grep "?"

# Serveis gestionats per systemd
systemctl list-units --type=service
```

## Seguretat: Processos sospitosos

### Com detectar processos malintencionats

#### 1. Processos desconeguts

```bash
# Llistar tots els processos
ps aux

# Buscar processos amb noms estranys
ps aux | grep -E "cryptominer|backdoor|shell"
```

**Indicis sospitosos:**

- Noms aleatoris (`a1b2c3`, `...`, `httpd` en un servidor sense servidor web Apache)
- Ubicats a `/tmp/` o `/var/tmp/`
- Executant-se amb noms d'usuaris estranys

#### 2. Alt consum de recursos sense motiu

```bash
# Veure processos que consumeixen més CPU
ps aux --sort=-%cpu | head -n 10

# Veure processos que consumeixen més RAM
ps aux --sort=-%mem | head -n 10
```

**Exemples sospitosos:**

- Procés desconegut al 100% CPU (potser cryptominer)
- Consum excessiu de xarxa (exfiltració de dades)

#### 3. Ports oberts inesperats

```bash
# Veure ports oberts i processos
sudo ss -tulnp

# Buscar ports no estàndard
sudo ss -tulnp | grep -v ":22\|:80\|:443"
```

**Sospitós:**

- Port 4444 obert (típic de reverse shells)
- Ports aleatoris oberts (investigar si és un servidor)

#### 4. Connexions de xarxa sospitoses

```bash
# Connexions actives
sudo ss -tnp

# Connexions a IPs estrangeres
sudo ss -tnp | grep ESTAB | awk '{print $5}' | cut -d: -f1 | sort -u
```

#### 5. Processos que s'executen des de /tmp/

```bash
# Trobar processos executant-se des de /tmp
sudo lsof +D /tmp | grep -i exec
```

**Per què és sospitós?**

- `/tmp/` disposa de permisos de lectura, escriptura i execució per tothom
- Atacants sovint depositen malware en aquest directori
- La informació inclosa és temporal (es neteja després de cada reinici)

### Accions davant un procés sospitós

**NO matar immediatament!** Primer investiga.

**Pas 1: Recull informació**

```bash
PID=5678  # El PID del procés sospitós

# Comanda completa
cat /proc/$PID/cmdline

# Fitxers oberts
ls -l /proc/$PID/fd/

# Connexions de xarxa
sudo ss -tnp | grep $PID

# Executable
ls -l /proc/$PID/exe
```

**Pas 2: Documenta**

```bash
# Captura l'estat
ps aux | grep $PID > proces-sospitos-$PID.txt
ls -l /proc/$PID/ >> proces-sospitos-$PID.txt
```

**Pas 3: Bloqueja (si és urgent)**

```bash
# Bloquejar connexions de xarxa del procés amb cgroups o firewall
# És una acció d'alta complexitat, depèn del cas

# O aturar el procés
sudo kill -STOP $PID  # Pausar (no matar encara)
```

**Pas 4: Analitza l'executable**

```bash
# Còpia de seguretat de l'executable per anàlisi forense (súper interessant)
cp /proc/$PID/exe /tmp/malware-$PID

# Comprovar el hash a la web de VirusTotal
sha256sum /tmp/malware-$PID
```

**Pas 5: Eliminar**

```bash
# Si confirmes que és maliciós
sudo kill -9 $PID

# Elimina l'executable
sudo rm -f /ruta/a/executable

# Comprova si hi ha persistència (cron, systemd, rc.local)
crontab -l
sudo systemctl list-unit-files | grep enabled
```

## Flux de treball: Gestió de processos

Quan tens un problema amb els processos, segueix aquest ordre:

```
1. Identificar el problema
   └─> top, ps aux, systemctl status
       "Què està passant?"

2. Localitzar el procés problemàtic
   └─> ps aux --sort=-%cpu, pgrep
       "Quin procés causa el problema?"

3. Investigar
   └─> /proc/[PID]/, lsof -p PID
       "Què està fent exactament?"

4. Decidir acció
   └─> Reiniciar? Matar? Ajustar prioritat?

5. Actuar
   └─> kill, renice, systemctl restart
       "Aplicar la acció/solució"

6. Verificar
   └─> ps, top, logs
       "S'ha resolt el problema?"

7. Documentar
   └─> Anotar què ha passat i com s'ha resolt
```

## Objectius d'aprenentatge

Al finalitzar aquest bloc, seràs capaç de:

1. Entendre què és un procés i com funciona el sistema operatiu
2. Diferenciar entre processos, threads i jobs
3. Interpretar els estats dels processos i el seu cicle de vida
4. Gestionar prioritats amb nice i renice
5. Utilitzar senyals per comunicar-te amb processos
6. Identificar i resoldre processos zombie i òrfens
7. Navegar pel sistema /proc/ per obtenir informació detallada
8. Comprendre la seqüència d'arrencada i el paper de systemd
9. Detectar processos sospitosos i actuar davant incidents de seguretat
10. Documentar processos crítics i procediments de gestió
