## 10. Monitorització de Recursos

Monitoritzar és **mesurar com es comporten els recursos ara**: CPU, memòria, disc i xarxa. L'objectiu és trobar el **coll d'ampolla**, el recurs que s'ha saturat i fa que tot vagi lent. Abans de canviar res cal saber quin recurs és, perquè cada un té una solució diferent.

| Recurs     | Ordres principals                         | Paquet               |
| ---------- | ----------------------------------------- | -------------------- |
| Visió general | `uptime`, `top`, `htop`                | instal·lades         |
| CPU        | `top`, `vmstat`, `mpstat`                 | `sysstat` (`mpstat`) |
| Memòria    | `free`, `vmstat`                          | instal·lades         |
| Disc (E/S) | `iostat`, `iotop`, `vmstat`               | `sysstat`, `iotop`   |
| Xarxa      | `ip -s link`, `ss -s`, `nload`            | `nload`              |

### Càrrega del Sistema: `uptime`

```bash
uptime
```

```
 10:15:02 up  1:46,  2 users,  load average: 2.10, 1.35, 0.62
```

La **càrrega mitjana** (*load average*) és el nombre mitjà de processos que volen CPU (estat `R`) o que esperen el disc (estat `D`) durant l'últim **minut**, els últims **5 minuts** i els últims **15 minuts**.

**Com s'interpreta?** Sempre en relació amb el **nombre de CPU** (`nproc`):

| Càrrega ÷ nombre de CPU | Situació                                                         |
| ----------------------- | ---------------------------------------------------------------- |
| Menys de 0,7            | Folgada: hi ha CPU lliure                                        |
| Al voltant d'1          | Plena: totes les CPU treballen, sense cua                         |
| Més d'1                 | Saturada: hi ha processos esperant torn                          |

Al servidor `srv0374` (2 CPU), una càrrega de `2.10` vol dir que les dues CPU treballen al màxim i comença a haver-hi cua. La mateixa càrrega en un servidor de 8 CPU seria baixa.

**La tendència també informa:** `2.10, 1.35, 0.62` (1 minut > 15 minuts) indica que la càrrega **està pujant**: el problema és recent. Si fos `0.60, 1.40, 2.10`, la càrrega estaria baixant.

`Nota:` La càrrega no és un percentatge d'ús de la CPU. Com que també compta els processos en estat `D`, un disc lent pot donar una càrrega alta amb la CPU gairebé aturada.

### Estats de la CPU

`top` i `vmstat` divideixen el temps de la CPU en categories:

| Codi | Nom                       | Significat                                                 | Si és alt...                                         |
| ---- | ------------------------- | ---------------------------------------------------------- | ---------------------------------------------------- |
| `us` | *user*                    | Programes d'usuari (aplicacions, scripts)                  | Algun programa consumeix molta CPU                   |
| `sy` | *system*                  | Kernel (crides al sistema, controladors)                   | Molta activitat de xarxa, disc o creació de processos |
| `ni` | *nice*                    | Programes amb *nice* positiu                               | Tasques de baixa prioritat treballant (normal)       |
| `id` | *idle*                    | CPU lliure                                                 | Bo: hi ha capacitat de sobres                        |
| `wa` | *I/O wait*                | CPU esperant el disc                                       | **Coll d'ampolla al disc**                           |
| `hi` / `si` | *interrupts*       | Interrupcions de maquinari / de programari                 | Molt de trànsit de xarxa                             |
| `st` | *steal*                   | Temps que l'hipervisor ha donat a altres màquines virtuals | L'amfitrió està sobrecarregat (només en VM)          |

### `top`: Monitor Interactiu

```bash
top
```

```
top - 10:15:02 up  1:46,  2 users,  load average: 2.10, 1.35, 0.62
Tasks: 148 total,   3 running, 145 sleeping,   0 stopped,   0 zombie
%Cpu(s): 97.5 us,  2.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :   3916.0 total,   2631.7 free,    498.2 used,    786.1 buff/cache
MiB Swap:   3916.0 total,   3916.0 free,      0.0 used.   3417.8 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   2410 xavi      20   0   48836  12544   4352 R  99.7   0.3   0:42.18 stress-ng-cpu
   2411 xavi      20   0   48836  12544   4352 R  99.3   0.3   0:42.05 stress-ng-cpu
   2433 xavi      20   0   11780   5376   3328 R   0.3   0.1   0:00.05 top
      1 root      20   0   22540  13312   9344 S   0.0   0.3   0:02.11 systemd
```

| Zona          | Interpretació en aquest exemple                                                       |
| ------------- | ------------------------------------------------------------------------------------- |
| Línia 1       | La mateixa informació que `uptime`: càrrega pujant (2,10 amb 2 CPU)                   |
| `Tasks`       | 3 processos en `R` i cap zombi                                                        |
| `%Cpu(s)`     | `us` 97,5% i `id` 0%: CPU saturada per programes d'usuari                             |
| `MiB Mem`     | Memòria total, lliure, usada i en memòria cau                                         |
| `avail Mem`   | Memòria **disponible** (és la xifra que cal mirar)                                    |
| Processos     | Dos `stress-ng-cpu` ocupen una CPU cadascun (99%)                                     |

**Columnes dels processos:** `PR` i `NI` (prioritat i *nice*, [08](./08_senyals_prioritats.md)), `VIRT` (memòria virtual), `RES` (memòria física), `SHR` (compartida), `S` (estat), `TIME+` (temps de CPU acumulat).

| Tecla | Funció                                          | Tecla | Funció                                     |
| ----- | ----------------------------------------------- | ----- | ------------------------------------------ |
| `P`   | Ordenar per CPU                                 | `k`   | Enviar un senyal a un PID                  |
| `M`   | Ordenar per memòria                             | `r`   | Canviar el *nice* d'un PID                 |
| `T`   | Ordenar per temps de CPU                        | `u`   | Mostrar només un usuari                    |
| `1`   | Mostrar cada CPU per separat                    | `c`   | Mostrar l'ordre completa                   |
| `V`   | Vista en arbre                                  | `H`   | Mostrar els fils                           |
| `E`   | Canviar les unitats de la memòria (MiB, GiB)    | `q`   | Sortir                                     |

Per fer-lo servir en scripts, el **mode no interactiu** fa una sola captura:

```bash
top -b -n 1 | head -n 12                       # capçalera i primers processos
top -b -n 1 -o %MEM | head -n 12               # ordenat per memòria
```

**`htop`** mostra la mateixa informació amb colors i barres per CPU, i permet fer servir el ratolí: `F5` vista en arbre, `F6` ordenar, `F9` enviar un senyal, `F7`/`F8` canviar el *nice*, `F10` sortir.

### Memòria: `free`

```bash
free -h
```

```
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       921Mi       412Mi       1.2Mi       2.5Gi       2.8Gi
Swap:          3.8Gi        12Mi       3.8Gi
```

| Columna       | Significat                                                                          |
| ------------- | ----------------------------------------------------------------------------------- |
| `used`        | Memòria que fan servir els processos                                                |
| `free`        | Memòria que no fa servir **ningú**                                                  |
| `buff/cache`  | Memòria cau de disc: fitxers llegits recentment. El kernel l'allibera quan cal      |
| `available`   | Memòria que poden fer servir els programes nous (lliure + memòria cau alliberable)  |

Linux aprofita la memòria lliure com a memòria cau de disc perquè els accessos siguin més ràpids. Per això, al cap d'un temps `free` sempre és baix. En aquest exemple, 412 MiB lliures **no és un problema**: n'hi ha 2,8 GiB disponibles. La memòria és un problema quan **`available` és baix** i la swap no para de créixer.

**Swap:** és una zona del disc on el kernel mou la memòria que no es fa servir quan la RAM s'esgota. Que hi hagi una mica de swap ocupada és normal. El problema és quan el sistema **escriu i llegeix de la swap contínuament** (columnes `si`/`so` de `vmstat`): el disc és milers de vegades més lent que la RAM i tot s'alenteix.

**Quan la memòria s'esgota del tot**, el kernel activa l'***OOM killer*** (*Out Of Memory*) i mata el procés que considera més prescindible:

```bash
sudo dmesg -T | grep -i "out of memory"
```

```
[Thu Nov  5 11:40:22 2026] Out of memory: Killed process 2512 (stress-ng-vm) total-vm:1052036kB, anon-rss:948800kB, ...
```

### Visió de Conjunt: `vmstat`

`vmstat` mostra CPU, memòria, swap i disc en una sola línia cada pocs segons. És ideal per veure l'evolució:

```bash
vmstat 2 5                         # una línia cada 2 segons, 5 vegades
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 2  0      0 2694860  42716 762300    0    0    35    12  210  305  6  1 93  0  0  0
 2  0      0 2694628  42716 762304    0    0     0     0 1020  210 99  1  0  0  0  0
 2  0      0 2694628  42716 762304    0    0     0     4 1018  198 99  1  0  0  0  0
```

| Columna     | Significat                                                         | Alerta si...                        |
| ----------- | ------------------------------------------------------------------ | ----------------------------------- |
| `r`         | Processos que volen CPU                                            | És més gran que el nombre de CPU    |
| `b`         | Processos bloquejats esperant E/S (estat `D`)                      | És més gran que 0 de manera continuada |
| `swpd`      | Swap utilitzada (KiB)                                              | Creix contínuament                  |
| `si` / `so` | Memòria llegida de / escrita a la swap per segon                   | Diferents de 0 de manera continuada |
| `bi` / `bo` | Blocs llegits / escrits al disc per segon                          | Valors alts amb `wa` alt            |
| `us sy id wa st` | Estats de la CPU (en %)                                       | Vegeu la taula d'estats             |
| `gu`        | Temps de CPU de màquines virtuals convidades (les versions antigues no el mostren) | — |

La **primera línia** és la mitjana des de l'arrencada; les següents, el que passa en cada interval. En l'exemple, `r = 2` i `us = 99`: les 2 CPU estan ocupades per programes d'usuari.

### Disc: `iostat` i `iotop`

```bash
sudo apt install sysstat iotop
iostat -dxz 2                      # estadístiques ampliades de cada disc cada 2 segons
```

Sortida simplificada (`iostat` mostra més columnes):

```
Device     r/s     rkB/s     w/s      wkB/s  r_await w_await  aqu-sz  %util
sda       2.00     16.00  410.50  105088.00     0.50   18.40    7.56  98.40
```

| Columna              | Significat                                                      |
| -------------------- | --------------------------------------------------------------- |
| `r/s`, `w/s`         | Lectures i escriptures per segon                                |
| `rkB/s`, `wkB/s`     | KiB llegits i escrits per segon                                 |
| `r_await`, `w_await` | Temps mitjà de cada operació en ms (inclou l'espera a la cua)   |
| `aqu-sz`             | Operacions a la cua del disc                                    |
| `%util`              | Percentatge de temps que el disc ha estat ocupat                |

En l'exemple, el disc està gairebé sempre ocupat (`%util` 98%) escrivint 100 MiB/s, amb 7,5 operacions en cua. Per saber **quin procés** el fa servir:

```bash
sudo iotop -o                      # només els processos que llegeixen o escriuen ara
```

### Xarxa

```bash
ip -s link show enp0s3             # bytes, paquets, errors i descarts de la interfície
ss -s                              # resum de connexions per estat
nload enp0s3                       # trànsit d'entrada i sortida en temps real (paquet nload)
```

Els comptadors d'**errors** o **descarts** que creixen indiquen problemes de cable, de targeta o de commutador.

### Seguir un Valor: `watch`

`watch` executa una ordre cada pocs segons i en mostra el resultat a pantalla completa:

```bash
watch -n 2 free -h                 # memòria cada 2 segons
watch -d -n 1 "ss -tn state established | wc -l"   # -d ressalta els canvis
```

### Generar Càrrega per Practicar

Per aprendre a interpretar les eines cal veure-les en un sistema carregat. `stress-ng` (paquet `stress-ng`) genera càrrega controlada:

```bash
stress-ng --cpu 2 --timeout 60s                         # 2 processos que consumeixen CPU durant 60 s
stress-ng --vm 1 --vm-bytes 80% --timeout 60s           # ocupa el 80% de la memòria disponible
cd /tmp && stress-ng --hdd 1 --hdd-bytes 1G --timeout 60s   # escriptura intensiva al disc
```

Mentre s'executen, observeu-los des d'un altre terminal amb `top`, `vmstat 2` o `iostat -dxz 2`.

`Nota:` Feu aquestes proves només a les màquines virtuals del laboratori, mai en un servidor en producció.

### Flux de Diagnosi: "El Servidor va Lent"

| Pas | Pregunta                          | Ordres                                     |
| --- | --------------------------------- | ------------------------------------------ |
| 1   | Hi ha un problema? Des de quan?   | `uptime` (càrrega i tendència)             |
| 2   | Quin recurs està saturat?         | `top` (`%Cpu`, `avail Mem`), `vmstat 2`    |
| 3   | Quin procés en té la culpa?       | `top` (`P`, `M`), `sudo iotop -o`, `ps`    |
| 4   | Què fa aquest procés i per què?   | `ps -o ...`, `/proc/PID`, registres ([11](./11_logs_diagnosi.md)) |
| 5   | Quina acció cal fer?              | `renice`, `kill`, `systemctl restart`, reprogramar la tasca |
| 6   | Ha funcionat?                     | Tornar a mesurar i documentar-ho           |

| Recurs saturat | Indicadors                                               | Accions habituals                                         |
| -------------- | -------------------------------------------------------- | --------------------------------------------------------- |
| CPU            | Càrrega > CPU, `us` o `sy` alts, `id` ≈ 0, `r` > CPU     | Baixar la prioritat (`renice`), aturar el procés, més CPU |
| Memòria        | `available` baix, `si`/`so` > 0, missatges de l'OOM killer | Reiniciar o limitar el servei que consumeix, més RAM    |
| Disc           | `wa` alt, processos en `D`, `%util` ≈ 100%               | Moure la tasca a una hora de poca activitat, disc més ràpid |
| Xarxa          | Errors o descarts, trànsit al límit de l'enllaç          | Revisar cable i commutador, identificar qui genera el trànsit |

### Cas Pràctic: "Aquest Matí Tot va Lent"

Són les 9:15 i els usuaris diuen que el servidor (2 CPU) respon molt a poc a poc.

**1. Hi ha un problema?**

```
load average: 4.85, 4.10, 2.30
```

Una càrrega de 4,85 amb 2 CPU és més del doble de la capacitat, i va pujant.

**2. Quin recurs?**

```
%Cpu(s):  8.0 us,  4.0 sy,  0.0 ni, 10.0 id, 78.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

La CPU gairebé no treballa (`us` 8%) però espera el disc el 78% del temps (`wa`). **El coll d'ampolla és el disc**, no la CPU: afegir més CPU no serviria de res.

**3. Quin procés?** `sudo iotop -o` mostra un `tar` que escriu 90 MB/s i `ps` indica que va començar a les 9:00 (`START 09:00`). És una còpia de seguretat que algú ha llançat a mà en hores de feina.

**4. Acció:** si queda poc per acabar, deixar-la acabar; si no, aturar-la (`kill`) i tornar-la a fer a la nit. Programar-la perquè s'executi sola de matinada (Tema 3) evita que es repeteixi.

**5. Verificació:** al cap de pocs minuts la càrrega baixa (`load average: 0.90, 2.40, 2.60`) i `wa` torna a ser proper a 0. Cal documentar l'incident: què ha passat, la causa i la solució.

### Resolució de Problemes

| Problema                                                    | Causa                                                    | Solució                                         |
| ----------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------- |
| `iostat` o `mpstat`: `command not found`                    | Paquet `sysstat` no instal·lat                            | `sudo apt install sysstat`                      |
| Un procés mostra més del 100% de CPU a `top`                | Fa servir diversos fils en diverses CPU (100% = 1 CPU)    | És normal. Amb `I` es divideix pel nombre de CPU |
| `free` mostra molt poca memòria lliure                      | Linux fa servir la memòria lliure com a memòria cau       | Mirar la columna `available`                    |
| Càrrega alta però la CPU està lliure (`id` alt)             | Processos en estat `D` esperant el disc o la xarxa (NFS)  | `vmstat` (columna `b`), `iostat`, `sudo iotop -o` |
| Hi ha swap ocupada tot i haver-hi RAM lliure                | Pàgines que es van moure fa temps i no s'han tornat a fer servir | És normal si `si`/`so` són 0              |
| `st` (*steal*) alt de manera continuada                     | L'ordinador amfitrió està sobrecarregat                   | Revisar les altres VM de l'amfitrió             |
| `iotop` diu que cal `sudo` o no mostra res                  | Necessita privilegis per llegir l'E/S de tots els processos | `sudo iotop -o`                               |
