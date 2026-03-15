# 04. Monitorització del Sistema

La monitorització del sistema és una de les tasques imprescindibles que ha de realitzar un administrador de sistemes. Mentre que la recollida d'informació estàtica ens permet conèixer **les característiques del servidor**, la monitorització ens permet saber **com està funcionant en aquest moment** i **si disposa de problemes de rendiment**.

La manca de monitorització d'un servidor ens impossibilita conèixer el seu estat actual; si hi ha algun component o servei que dona error o va lent, si hi ha un procés que està consumint molts recursos, si els usuaris estan realitzant accions nocives o que alenteixen el funcionament normal del servidor.

Aquest bloc es centra en treballar amb les eines que permeten **monitoritzar en temps real** el rendiment del sistema: la CPU, la memòria, el disc, l'entrada/sortida (I/O), etc.

## Per què és important?

Imagina aquestes situacions reals que et trobaràs com a administrador de sistemes:

### Escenari 1: "El servidor va molt lent"

> És dilluns al matí, 9:00h. Reps un correu urgent: "La web no carrega, tot va molt lent". Els clients es queixen. El teu cap pregunta: "Què està passant?"

**Necessites saber:**

- Quina és la càrrega actual del servidor?
- Quin procés està consumint tota la CPU?
- S'ha esgotat la memòria RAM?
- El disc està col·lapsat amb processos de lectures/escriptures?

**Sense monitorització:** Has d'investigar molt per tenir informació detallada dels errors.  
**Amb monitorització:** En 2 minuts identifiques que un backup automàtic està saturant el disc.

### Escenario 2: Load average de 8.0

> Executes `uptime` i veus: `load average: 8.50, 7.23, 6.15`. No saps si això és normal o preocupant.

**Preguntes que necessites respondre:**

- És greu un load average de 8.X?
- Aquest valor depèn dels nuclis del servidor?
- És un valor que fa referència a la CPU, la memòria o el disc?

**Necessites:** Entendre què significa cada mètrica i com interpretar-la segons el teu hardware.

### Escenari 3: Memòria al límit

> Un servei crític (base de dades) es reinicia sol a la poca estona de reiniciar-se. Revises els logs i veus: `Out of memory: Kill process 5678 (mysqld)`.

**El problema:**

- El sistema està matant processos perquè no hi ha RAM suficient
- Pot ser que hi hagi un memory leak (fuita de memòria)?
- Potser la swap està saturada i el sistema va molt lent?

**Necessites:**

- Veure l'ús de memòria en temps real
- Identificar quin procés o processos consumeixen més RAM
- Comprovar si s'està utilitzant molta swap (senyal de problemes)
- Decidir si cal reiniciar algun servei, ampliar la RAM i la SWAP

### Escenari 4: Disc lent

> Els usuaris reporten que a l'intentar guardar fitxers triga molt de temps. A l'intentar treballar amb fitxers compartits en xarxa dona errors.

**Causes possibles:**

- El disc està al 100% d'ús (I/O wait alt)
- Hi ha un procés fent moltes lectures/escriptures
- El disc físic pot estar fallant
- Hi ha problemes de xarxa (si és un disc remot/NFS/SMB)

**Necessites:**

- Monitoritzar l'ús del disc en temps real
- Identificar quins processos fan més I/O
- Veure si hi ha coll d'ampolla a la xarxa

### Escenari 5: Planificar capacitat

> El cap de projecte pregunta: "Podem afegir 50 usuaris més al servidor o necessitem ampliar?"

**Per respondre correctament:**

- Quin és l'ús mitjà de CPU durant el dia?
- Com evoluciona la memòria al llarg de la setmana?
- Hi ha pics d'ús en hores concretes?
- Els recursos actuals són suficients o alguns estan al límit?

**Necessites:** Dades històriques i monitorització continuada per prendre decisions basades en mètriques reals.

## Diferència: Informació estàtica vs Monitorització dinàmica

És important entendre la diferència entre el que vam veure al tema anterior i el que veurem ara:

### Informació estàtica del sistema

**Què ens diu:** Característiques permanents del servidor.

**Comandes:**

- `lscpu` → Quants nuclis té la CPU
- `free -h` → Quanta RAM total està instal·lada
- `lsblk` → Quins discos hi ha
- `df -h` → Capacitat total dels discos

**Exemple:**

```bash
lscpu | grep "^CPU(s):"
# CPU(s): 4
# Resposta: "El servidor disposa de 4 CPUs"
```

**Quan s'utilitza:** Per conèixer les especificacions tècniques, fer inventaris, planificar ampliacions, etc.

### Monitorització dinàmica (en temps real)

**Què ens diu:** Com estan funcionant els recursos **actualment**.

**Comandes:**

- `top` / `htop` → Quanta CPU s'està utilitzant ara
- `vmstat` → Com evoluciona la memòria cada segon
- `iostat` → Rendiment dels discos en temps real
- `mpstat` → Ús de CPU per cada nucli

**Exemple:**

```bash
top
# %Cpu(s): 95.3 us, 2.1 sy, 0.0 ni, 2.6 id
# Resposta: "La CPU està al 95% d'ús."
```

**Quan s'utilitza:** Per diagnosticar problemes de rendiment, detectar colls d'ampolla, optimitzar el sistema, etc.

### Comparativa Estàtic vs Dinàmic

| Aspecte      | Informació estàtica            | Monitorització dinàmica                |
| ------------ | ------------------------------ | -------------------------------------- |
| **Pregunta** | Què té el servidor?            | Com està funcionant?                   |
| **Resposta** | 8GB RAM, 4 CPUs                | CPU al 80%, RAM al 60%                 |
| **Canvia?**  | No (fins que amplies hardware) | Sí (cada segon)                        |
| **Eines**    | `lscpu`, `lsblk`, `df`         | `top`, `vmstat`, `iostat`              |
| **Exemple**  | "Tinc 4 nuclis de CPU"         | "Ara estic utilitzant 3.5 de 4 nuclis" |

## Conceptes fonamentals de monitorització

Abans de començar amb les comandes, és essencial entendre els conceptes clau que trobaràs constantment.

### 1. Load Average (Càrrega mitjana)

El **load average** mesura la quantitat de processos que estan esperant per executar-se o s'estan executant.

```bash
uptime
# 10:30:15 up 2 days, 3:45, 2 users, load average: 1.50, 2.30, 3.10
#                                                  ----  ----  ----
#                                                  1min  5min  15min
```

**Interpretació:**

La regla bàsica és comparar-ho amb el **número de CPUs**:

```
Load average = 1.0 en un sistema amb 1 CPU → 100% ocupat (sistema saturat)
Load average = 1.0 en un sistema amb 4 CPUs → 25% ocupat
Load average = 4.0 en un sistema amb 4 CPUs → 100% ocupat (sistema saturat)
Load average = 8.0 en un sistema amb 4 CPUs → 200% ocupat (saturat i amb cua d'espera)
```

**Exemple:**

```bash
# Servidor amb 4 CPUs
lscpu | grep "^CPU(s):"
# CPU(s): 4

uptime
# load average: 2.50, 2.30, 2.10

# Càlcul: 2.5 / 4 = 62.5% d'ús → Correcte :)
```

**Per què hi ha 3 valors?**

- **1 minut**: Càrrega recent (instantània)
- **5 minuts**: Tendència a curt termini
- **15 minuts**: Tendència general

**Patrons a interpretar:**

```
load average: 0.50, 2.30, 5.10
# La càrrega està baixant → El "problema" s'està resolent

load average: 5.10, 2.30, 0.50
# La càrrega està pujant → Problema recent, s'ha d'investigar

load average: 4.50, 4.30, 4.20
# La càrrega és alta tota l'estona → Problema constant (persistent)!
```

### 2. CPU States (Estats de la CPU)

Quan monitoritzes la CPU, veuràs que el temps es divideix en diferents tipus d'ús:

```bash
top
%Cpu(s): 25.0 us, 5.0 sy, 0.0 ni, 68.0 id, 2.0 wa, 0.0 hi, 0.0 si, 0.0 st
```

**Significat de cada estat:**

| Estat  | Nom                 | Descripció                              | Quan és alt                                  |
| ------ | ------------------- | --------------------------------------- | -------------------------------------------- |
| **us** | User Space          | Temps per processos d'usuari            | Aplicacions normals (Apache, MySQL)          |
| **sy** | System              | Temps per processos del kernel          | Moltes crides al sistema, problemes drivers  |
| **ni** | Nice                | Temps per processos amb prioritat baixa | Tasques de manteniment (backups)             |
| **id** | Idle                | Temps de CPU sense fer res              | **CPU lliure** (correcte si és un valor alt) |
| **wa** | I/O Wait            | Esperant operacions de disc/xarxa       | **Coll d'ampolla en I/O** (disc lent)        |
| **hi** | Hardware Interrupts | Interrupcions de hardware               | Problemes de hardware o drivers              |
| **si** | Software Interrupts | Interrupcions de software               | Trànsit de xarxa elevat                      |
| **st** | Steal               | Temps robat (per VMs)                   | Hipervisor sobrecarregat                     |

**Interpretació pràctica:**

**Escenari 1: Alta càrrega d'aplicacions d'usuaris**

```
%Cpu(s): 85.0 us, 5.0 sy, 0.0 ni, 10.0 id, 0.0 wa
```

- 85% en user space → Aplicacions que estan consumint CPU
- **Acció:** Identificar quin procés o processos estan consumint més (`top`, `ps`)

**Escenari 2: Coll d'ampolla al disc**

```
%Cpu(s): 10.0 us, 5.0 sy, 0.0 ni, 5.0 id, 80.0 wa
```

- 80% en I/O wait → La CPU espera el disc
- **Problema:** Disc lent o amb molta càrrega de lectures/escriptures
- **Acció:** Comprovar amb `iostat`, `iotop`

**Escenari 3: Problemes del kernel**

```
%Cpu(s): 10.0 us, 70.0 sy, 0.0 ni, 20.0 id, 0.0 wa
```

- 70% en system → Els processos del Kernel ocupen molt de temps de CPU
- **Possibles causes:** Moltes crides al sistema, problemes de drivers, etc.
- **Acció:** Revisar logs del kernel (`dmesg`)

**Escenari 4: VM sobrecarregada**

```
%Cpu(s): 10.0 us, 2.0 sy, 0.0 ni, 50.0 id, 0.0 wa, 0.0 hi, 0.0 si, 38.0 st
```

- 38% steal time → L'hipervisor no proporciona suficient CPU a la VM
- **Problema:** Host físic saturat
- **Acció:** Contactar amb el proveïdor de cloud/virtualització

### 3. Ús de Memòria

La memòria RAM és un recurs crític. Entendre com s'utilitza és essencial.

**Tipus de memòria:**

```bash
free -h
              total        used        free      shared  buff/cache   available
Mem:           7.7Gi       2.1Gi       3.2Gi       180Mi       2.4Gi       5.1Gi
Swap:          2.0Gi       512Mi       1.5Gi
```

**Components:**

| Component      | Descripció                       | És problemàtic?         |
| -------------- | -------------------------------- | ----------------------- |
| **total**      | RAM total instal·lada            | -                       |
| **used**       | Memòria utilitzada per processos | Alt = molts processos   |
| **free**       | Memòria completament lliure      | Baix no és dolent       |
| **shared**     | Memòria compartida (tmpfs, etc.) | -                       |
| **buff/cache** | Memòria utilitzada per caché     | **És reutilitzable**    |
| **available**  | Memòria disponible realment      | **La més important**    |
| **swap**       | Memòria virtual al disc          | Alt = **problema greu** |

**Concepte clau: buff/cache**

Linux utilitza la RAM "lliure" com a caché per accelerar l'accés a discos. **Això és positiu**, no negatiu.

**Escenari 1: Memòria "saludable"**

```
Mem:   8Gi total,  3Gi used,  1Gi free,  4Gi buff/cache,  5Gi available
Swap:  2Gi total,  0Gi used,  2Gi free
```

- 5Gi available → Hi ha memòria disponible suficient
- 0Gi swap → No s'està utilitzant swap (**això és ideal**)

**Escenari 2: Memòria suficient (limitada)**

```
Mem:   8Gi total,  6Gi used,  500Mi free,  1.5Gi buff/cache,  2Gi available
Swap:  2Gi total,  200Mi used,  1.8Gi free
```

- 2Gi available → És suficient, però just, només un 25% de memòria disponible.
- 200Mi swap → S'està utilitzant una mica de swap (**cal vigilar relativament**)

**Escenario 3: Problema de Memòria**

```
Mem:   8Gi total,  7.8Gi used,  100Mi free,  200Mi buff/cache,  300Mi available
Swap:  2Gi total,  1.9Gi used,  100Mi free
```

- 300Mi available → Pràcticament la totalitat de la memòria està sent utilitzada.
- 1.9Gi swap → La SWAP està pràcticament plena (el sistema pot anar molt lent).
- **Acció urgent:** Identificar processos que consumeixen RAM, reiniciar serveis i ampliar RAM si es requereix.

### 4. I/O Wait i rendiment de disc

**I/O Wait** és el percentatge de temps que la CPU passa esperant operacions d'entrada/sortida (disc o xarxa).

**Regla bàsica:**

- **< 10% I/O wait**: Normal
- **10-20% I/O wait**: Cal vigilar
- **> 20% I/O wait**: **Coll d'ampolla a I/O** (el disc va lent)

**Per què és un problema?**

Si la CPU passa el 80% del temps esperant el disc:

- Els processos van lents (esperen dades durant molt de temps)
- El sistema sembla "penjat" tot i disposar de la CPU lliure (sense treballar)
- Les aplicacions no poden progressar (executar-se amb normalitat i tenir un comportament fluit)

**Causes comunes:**

1. **Disc lent o saturat**

   - Massa lectures/escriptures simultànies
   - Disc mecànic (HDD) en lloc de SSD
   - Disc defectuós (comprovar el disc o canviar)

2. **Procés fent moltes Lectures/Escriptures (I/O)**

   - Pot ser un backup en execució
   - Hi ha base de dades amb moltes consultes i escriptures (SELECT/INSERTS)
   - Sistema de logs excessiu (moltes escriptures dels serveis)

3. **Problema de xarxa (NFS, SMB, iSCSI)**
   - El disc remot va molt lent
   - Hi ha molta latència de xarxa

**Com detectar-ho:**

```bash
# Veure I/O wait
top
%Cpu(s): 10.0 us, 5.0 sy, 0.0 ni, 5.0 id, 80.0 wa
#                                         ^^^^
#                                         Problema!

# Veure quin procés realitza més I/O
iotop
# Mostra processos ordenats per lectura/escriptura

# Estadístiques dels discos
iostat -x 1
# Mostra la utilització de cada disc
```

### 5. Swap (Memòria virtual)

**La SWAP** és l'espai al disc que s'utilitza com a "memòria extra" quan l'ús de la RAM és molt alt o s'esgota.

**Com funciona:**

```
1. RAM s'omple majoritariament o en la seva totalitat
2. El kernel mou les pàgines de memòria RAM menys utilitzades a la memòria swap (disc)
3. Allibera RAM per a processos nous
4. Si un procés necessita una pàgina de la swap, la torna a carregar a la RAM
```

**Per què és lent:**

- **Disc vs RAM**: El disc és entre 100 i 1000 vegades és lent que la RAM
- Si hi ha molt d'intercanvi RAM/DISC (swapping constant), el sistema es ralentitza molt

**Interpretació:**

```bash
free -h
Swap:  2.0Gi total,  1.5Gi used,  500Mi free
```

| Ús de Swap | Estat       | Acció                                    |
| ---------- | ----------- | ---------------------------------------- |
| 0-10%      | Perfecte    | Cap                                      |
| 10-50%     | Acceptable  | Vigilar (fer seguiment)                  |
| 50-80%     | Problemàtic | Identificar processos, reiniciar serveis |
| 80-100%    | Crític      | Ampliar RAM o reduir càrrega urgentment  |

**Monitoritzar swapping actiu:**

```bash
vmstat 1
# Columnes "si" i "so" mostren swap in/out per segon
# Si són alts, hi ha swapping constant (això és molt dolent)
```

## Tipus de recursos que monitoritzarem

### 1. CPU (Processador)

**Què monitoritzes:**

- Percentatge d'ús per nucli
- Load average
- Processos que consumeixen més CPU
- Temps en diferents estats (user, system, wait)

**Eines:**

- `top` / `htop` - Visió general interactiva
- `mpstat` - Estadístiques per CPU
- `sar` - Històric de CPU

**Pregunta clau:** Hi ha algun procés consumint massa CPU?

### 2. Memòria RAM

**Què monitoritzes:**

- Memòria total vs utilitzada
- Memòria disponible real (available)
- Ús de swap
- Processos que consumeixen més RAM

**Eines:**

- `free` - Resum de memòria
- `vmstat` - Estadístiques de memòria i swap
- `top` / `htop` - Per procés
- `slabtop` - Caché del kernel

**Pregunta clau:** Estem sense memòria? S'està utilitzant swap?

### 3. Disc i I/O

**Què monitoritzes:**

- Percentatge d'ús de cada disc
- Velocitat de lectura/escriptura (MB/s)
- I/O wait (temps esperant disc)
- Processos que fan més I/O

**Eines:**

- `iostat` - Estadístiques de discos
- `iotop` - Processos per I/O
- `lsof` - Fitxers oberts

**Pregunta clau:** El disc és el coll d'ampolla?

### 4. Xarxa (menys detall en aquest tema)

**Què monitoritzes:**

- Trànsit entrant/sortint
- Paquets per segon
- Errors de xarxa

**Eines:**

- `sar -n DEV` - Estadístiques de xarxa
- `iftop` - Trànsit en temps real
- `ss` / `netstat` - Connexions actives

## Flux de treball: Diagnosticar problemes de rendiment

Quan reps l'alerta "el servidor va lent", segueix aquest ordre metòdic:

### Pas 1: Visió general ràpida (30 segons)

```bash
# 1. Càrrega del sistema
uptime
# load average: 8.50, 7.23, 6.15

# 2. Visió general dels recursos
top
# (observa: %CPU, %MEM, load average, processos amb més ús)
```

**Pregunta:** Quin recurs és el problemàtic? És la CPU, la memòria o el disc?

### Pas 2: Identificar el recurs saturat (2 minuts)

**Si l'ús de CPU és alt (> 80%):**

```bash
# Veure els processos que consumeixen més CPU
ps aux --sort=-%cpu | head -n 10
```

**Si l'ús de memòria és alt (available < 20%):**

```bash
# Veure la memòria disponible i la swap
free -h

# Processos que consumeixen més RAM
ps aux --sort=-%mem | head -n 10
```

**Si l'I/O wait és alt (> 20%):**

```bash
# Veure el rendiment dels discos
iostat -x 1 5

# Veure els processos que realitzen més I/O
sudo iotop
```

### Pas 3: Identificar el procés problemàtic (2 minuts)

```bash
# Amb top/htop, ordena per:
# - CPU: tecla 'P'
# - Memòria: tecla 'M'
# - Temps: tecla 'T'

# O directament:
ps aux --sort=-%cpu | head -n 10    # Top 10 CPU
ps aux --sort=-%mem | head -n 10    # Top 10 RAM
```

### Pas 4: Investigar el procés (3 minuts)

```bash
PID=5678  # El PID del procés problemàtic

# Obtenir informació del procés
ps -p $PID -o pid,user,%cpu,%mem,cmd

# Mostrar els fitxers oberts d'aquell procés
lsof -p $PID

# Veure les connexions de xarxa
ss -tnp | grep $PID

# Mostrar els detalls del procés
cat /proc/$PID/status
```

### Pas 5: Decidir l'acció a realitzar

**Opcions més utilitzades:**

1. **Reiniciar el servei:**

   ```bash
   sudo systemctl restart apache2
   ```

2. **Matar el procés:**

   ```bash
   kill -15 $PID     # Intent "educat"
   sleep 5
   kill -9 $PID      # Si no respon
   ```

3. **Ajustar prioritat:**

   ```bash
   renice -n 10 -p $PID  # Reduir prioritat (fins a +19)
   ```

4. **Optimitzar la configuració:**
   - Millorar la configuració del servei web d'Apache.
   - Millorar la configuració del servei de bases de dades de MySQL.

### Pas 6: Verificar i documentar

```bash
# Comprovar que el problema s'ha resolt
top
uptime
free -h

# Documentar l'incident
echo "$(date): Procés $PID (apache2) consumint 100% CPU. Reiniciat servei." >> /var/log/incidents.log
```

## Cas pràctic: "El servidor va lent aquest matí"

### Situació

És dimarts, 9:15h. Acabes d'arribar a la feina i tens 5 missatges urgents:

> "La web no carrega"  
> "Tot va lentíssim (molt descriptiu)"  
> "No puc accedir a les carpetes compartides"  
> "Les consultes a la BD triguen minuts"  
> "El cap vol saber què passa"

### Investigació

**1. Primera comprovació (connectar-te)**

```bash
ssh admin@servidor-prod
# Triga 10-15 segons a connectar (normalment és instantani)
# Senyal de problema
```

**2. Visió general**

```bash
uptime
# 09:15:45 up 15 days, 3:22, 2 users, load average: 12.50, 11.23, 8.15

# ALERTA: Load average de 12.5 en un servidor amb 4 CPUs
# Càlcul: 12.5 / 4 = 312.5% d'ús!
```

**3. Comprovar recursos**

```bash
top
# %Cpu(s): 15.0 us, 5.0 sy, 0.0 ni, 5.0 id, 75.0 wa
#                                           ^^^^^
#                                           75% I/O WAIT!

# Problema identificat: Coll d'ampolla en I/O
```

**4. Identificar quin procés fa I/O**

```bash
sudo iotop -o
# PID   USER   DISK READ  DISK WRITE  COMMAND
# 5678  mysql  850 MB/s   1200 MB/s   mysqld
```

**5. Investigar el procés MySQL**

```bash
ps aux | grep mysql
# mysql  5678  15.2  35.6  2456784 1456920 ?  D   03:00  185:45 /usr/sbin/mysqld

# Estat "D" = Uninterruptible sleep (esperant I/O)
# Fa 6 hores que s'executa (START 03:00) Des les 3:00 fins les 9:15
```

**6. Comprovació addicional**

```bash
# Veure què està fent el MySQL
sudo lsof -p 5678 | grep -i database
# mysqld  5678  mysql  5w  REG  /var/lib/mysql/productes.ibd (escriptura constant)

# Revisar si hi ha una consulta problemàtica
mysql -e "SHOW PROCESSLIST;" | grep -v Sleep
# Descoberta: Query de backup fent full table scan
```

### Causa arrel

Un **backup manual** iniciat a les 3:00h encara s'està executant:

- Estava llegint tota la base de dades
- Saturant el disc amb lectures
- Provocant I/O wait del 75%
- Ralentint tots els altres processos

### Solució

```bash
# 1. Aturar el backup problemàtic
kill -15 5678
sleep 10
kill -9 5678

# 2. Comprovar millora
uptime
# load average: 2.5, 5.3, 7.8
# (baixant ràpidament)

top
# %Cpu(s): 25.0 us, 5.0 sy, 0.0 ni, 65.0 id, 5.0 wa
# I/O wait baixat a 5% (normal)

# 3. Reiniciar MySQL per si té problemes residuals
sudo systemctl restart mysql

# 4. Verificar que tot funciona
curl -I http://localhost
# HTTP/1.1 200 OK (web funciona)
```

### Accions posteriors

**Immediates:**

- Avisar l'equip que el problema s'ha resolt
- Monitoritzar durant 30 minuts per confirmar l'estabilitat del sistema

**A curt termini:**

- Revisar l'script de backup (per què ha trigat tant?)
- Configurar alertes per I/O wait > 50%
- Programar backups en un altre horari (potser ha de començar a les 01:00)

**Documentació:**

```bash
# Afegir a l'informe d'incidents
cat >> /var/log/incidents.log << EOF
Data: $(date)
Problema: I/O wait 75%, load average 12.5
Causa: Backup manual de MySQL saturat disc
Solució: Aturat procés, reiniciat MySQL
Durada: 15 minuts
Acció preventiva: Revisar scripts backup
EOF
```

### Lliçons apreses

1. **I/O wait alt** és un indicador clar de problemes de disc
2. El **load average** pot ser enganyós (no sempre ha de ser CPU)
3. L'**iotop** és essencial per identificar processos amb I/O intens
4. Els **backups** s'han de programar en horaris de baixa càrrega
5. Cal tenir **alertes** configurades per detectar problemes abans que els usuaris es queixen

## Objectius d'aprenentatge

Al finalitzar aquest tema, seràs capaç de:

1. Entendre i interpretar el **load average** segons el nombre de CPUs
2. Diferenciar els **estats de CPU** (user, system, idle, wait, steal)
3. Interpretar l'**ús de memòria** real (available vs free, buff/cache)
4. Comprendre què és la memòria **swap** i per què pot ser problemàtica
5. Identificar els **colls d'ampolla** de CPU, memòria o I/O
6. Utilitzar **top/htop** per realitzar una monitorització interactiva general
7. Analitzar el rendiment de la CPU amb **mpstat** i dades històriques amb **sar**
8. Monitoritzar la memòria i la swap amb **vmstat** i **free**
9. Detectar problemes de disc amb **iostat** i **iotop**
10. Seguir un **flux de treball metòdic** per diagnosticar problemes de rendiment
11. Documentar incidents i generar informes de monitorització
12. Diferenciar entre problemes de hardware, software i configuració
13. Prendre **decisions basades en mètriques** reals, no intuïcions
14. Prevenir problemes futures configurant alertes i monitorització proactiva

## Comandes d'aquest tema

Les comandes es distribueixen en els següents fitxers:

1. **01_top_htop.md** - Monitorització interactiva general

   - `top` - Monitor de sistema clàssic
   - `htop` - Monitor avançat i visual
   - `atop` - Monitor amb dades històriques

2. **02_cpu_memoria_disc.md** - Anàlisi detallada de CPU, Memòria i Disc

   - `mpstat` - Estadístiques per CPU/nucli
   - `sar` - Dades històriques de rendiment

   - `free` - Resum de memòria
   - `vmstat` - Estadístiques de memòria virtual
   - `slabtop` - Caché del kernel

   - `iostat` - Estadístiques de discos
   - `iotop` - Processos per I/O
   - `lsof` - Fitxers oberts per processos

### Script de test (generació de càrrega)

```bash
# 1. Càrrega de CPU (simple)
yes > /dev/null &
stress-ng --cpu 4 --timeout 60s

# 2. Càrrega de memòria
stress --vm 2 --vm-bytes 1G --timeout 60s
stress-ng --vm 2 --vm-bytes 2G --timeout 30s

# 3. Càrrega de disc
dd if=/dev/zero of=/tmp/testfile bs=1M count=10000
fio --name=seqwrite --rw=write --bs=1M --size=1G --numjobs=1

# 4. Benchmark complet (opcional)
sysbench cpu run
sysbench memory run
```

**Nota**: Aquests comandes són per practicar en servidors de proves, **mai per fer en producció**.
