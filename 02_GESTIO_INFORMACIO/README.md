# Tema 2 - Administració de Processos i Monitorització del Sistema

## Objectiu del Tema

En aquest tema s'introdueixen els fonaments de l'administració de processos i anàlisi de sistemes Linux. L'alumnat aprendrà a entendre el funcionament dels processos, monitoritzar l'estat del sistema i fer un diagnòstic d'incidències reals. Es proposaran situacions reals en entorns empresarials per desenvolupar les competències i poder donar suport o solució a problemàtiques en entorns de producció.

L'objectiu principal és dominar la recollida d'informació del sistema, saber filtrar-la i processar-la per poder després realitzar tasques d'automatització.

**Durada total**: 26 hores  
**Sistema operatiu**: Ubuntu Server 24.04 LTS  
**Infraestructura**: Server 0374 + Client Ubuntu (xarxa privada)

## Temes i distribució d'hores

| Tema                                 | Hores | Descripció                                                                              |
| ------------------------------------ | ----- | --------------------------------------------------------------------------------------- |
| **01. Informació del Sistema**       | 4     | Comandes per obtenir informació estàtica: hardware, CPU, memòria, disc, xarxa, kernel.  |
| **02. Gestió de Processos**          | 2     | Teoria: processos, estats, cicle de vida, threads, prioritats, interrupcions.           |
| **02. Gestió de Processos**          | 4     | Senyals, kill, nice/renice. Control de processos en foreground/background.              |
| **03. Systemd i Serveis**            | 4     | systemctl, journalctl, creació i gestió de units, dependències i arrencada del sistema. |
| **04. Monitorització en Temps Real** | 4     | Monitorització dinàmica de processos i recursos: CPU, RAM, disc, swap, xarxa, I/O.      |
| **05. Activitats Pràctiques**        | 8     | Casos pràctics: problemes de rendiment, processos bloquejats, recursos al límit.        |

## Objectius d'aprenentatge

Al finalitzar aquest RA, seràs capaç de:

1. Entendre què són els processos i el seu funcionament en un sistema Linux
2. Recollir informació completa del sistema (hardware, recursos, xarxa, processos)
3. Monitoritzar l'estat del sistema en temps real i identificar problemes de rendiment
4. Filtrar i processar informació amb comandes de text processing
5. Gestionar processos amb senyals i ajustar prioritats de manera efectiva
6. Controlar serveis amb systemd i analitzar logs amb journalctl
7. Identificar i resoldre problemes reals relacionats amb processos i rendiment
8. Preparar dades per a futura automatització i creació de dashboards

## Eines que utilitzarem

### 1. Informació del Sistema (Recollida d'informació)

#### Sistema i Hardware

- `uname` - Informació del kernel i sistema operatiu
- `hostnamectl` - Informació completa del sistema (hostname, OS, kernel, arquitectura)
- `lscpu` - Detall complet de la CPU (nuclis, arquitectura, MHz)
- `lspci` - Dispositius PCI connectats (targetes gràfiques, xarxa, etc.)
- `lsusb` - Dispositius USB connectats
- `lsblk` - Dispositius de bloc i particions (discs i unitats)
- `dmidecode` - Informació detallada del hardware (BIOS, placa base, RAM)
- `hwinfo` - Resum complet del hardware del sistema

#### Estructura de Directoris

- `tree` - Visualització jeràrquica en arbre de directoris i fitxers
- `find` - Cerca avançada de fitxers i directoris amb criteris múltiples
- `locate` - Cerca ràpida de fitxers utilitzant una base de dades indexada
- `which` - Localitza l'executable d'una comanda al PATH
- `whereis` - Localitza binaris, codi font i pàgines de manual d'una comanda

#### CPU i Càrrega

- `uptime` - Temps d'activitat del sistema i càrrega mitjana
- `nproc` - Nombre de nuclis de processador disponibles

#### Memòria RAM i Swap

- `free` - Estat de la memòria RAM i swap (lliure, usada, disponible)
- `lsmem` - Detall dels mòduls de memòria instal·lats
- `cat /proc/meminfo` - Informació detallada de la memòria del sistema

#### Discos i Sistemes de Fitxers

- `df` - Espai lliure i ocupat per sistema de fitxers
- `du` - Ús d'espai per directoris i fitxers específics
- `blkid` - UUIDs i tipus de sistema de fitxers de les particions
- `fdisk -l` - Informació de particions dels discs

#### Gestió de la Xarxa

- `ip a` - Configuració i estat de les interfícies de xarxa
- `ip route` - Taula de rutes del sistema
- `nmcli` - Gestió i informació de connexions de xarxa (NetworkManager)
- `ss` - Connexions de xarxa actives (substitut de netstat)
- `netstat` - Estadístiques de xarxa i connexions (antic)
- `ethtool` - Informació i configuració avançada d'interfícies
- `cat /proc/net/dev` - Estadístiques de les interfícies de xarxa

#### Processos (Informació Estàtica)

- `ps` - Llista de processos actius en un moment determinat
- `pstree` - Vista jeràrquica dels processos (arbre de processos)
- `pgrep` - Cerca processos per nom o altres criteris

#### Usuaris i Sessions

- `who` - Usuaris connectats actualment al sistema
- `w` - Usuaris connectats i què estan fent
- `last` - Històric de connexions d'usuaris
- `lastlog` - Últim login de cada usuari

#### Kernel i Sistema

- `dmesg` - Missatges del buffer del kernel (errors, hardware, drivers)
- `sysctl -a` - Paràmetres de configuració del kernel
- `cat /proc/cpuinfo` - Informació detallada de cada CPU
- `cat /proc/version` - Versió del kernel i informació de compilació

### 2. Gestió de Processos i Serveis

#### Control de Processos

- `kill` - Enviar senyals a processos (SIGTERM, SIGKILL, etc.)
- `killall` - Finalitzar processos per nom
- `pkill` - Finalitzar processos amb criteris de cerca avançats
- `nice` - Executar comanda amb prioritat modificada
- `renice` - Canviar prioritat d'un procés existent
- `nohup` - Executar comanda que ignori el tancament de sessió
- `screen` - Multiplexor de terminal persistent
- `tmux` - Multiplexor de terminal avançat

#### Systemd (Gestió de Serveis)

- `systemctl` - Control i gestió de serveis del sistema
- `journalctl` - Consulta i anàlisi de logs del sistema
- `systemd-analyze` - Anàlisi del temps d'arrencada del sistema

### 3. Monitorització del Sistema (Temps Real)

#### Processos a Temps Real

- `top` - Monitorització interactiva de processos i recursos
- `htop` - Versió millorada de top amb interfície més amigable
- `atop` - Monitor avançat de processos amb històric
- `pidstat` - Estadístiques per procés individual (CPU, memòria, I/O)

#### Recursos en General

- `vmstat` - Estadístiques de memòria virtual, CPU i swap
- `mpstat` - Estadístiques per cada CPU/nucli individual
- `sar` - Recopilació i visualització d'estadístiques del sistema al llarg del temps

#### Memòria RAM

- `watch free -h` - Actualització automàtica cada 2 segons

#### Discos i I/O

- `iostat` - Estadístiques d'entrada/sortida de discs
- `iotop` - Processos ordenats per ús d'I/O de disc
- `ioping` - Latència d'I/O en temps real

#### Gestió de la Xarxa

- `iftop` - Monitorització del tràfic de xarxa en temps real per interfície
- `nethogs` - Tràfic de xarxa per procés
- `vnstat` - Estadístiques històriques de tràfic de xarxa
- `nload` - Visualització gràfica del tràfic de xarxa

#### Sistema de Fitxers `/proc`

- `/proc/[PID]/` - Informació completa de cada procés
- `/proc/cpuinfo` - Informació de CPU
- `/proc/meminfo` - Informació de memòria
- `/proc/net/` - Estadístiques de xarxa
- `/proc/sys/` - Paràmetres configurables del kernel

### 4. Eines de Diagnosi Avançat

#### Debugging

- `strace` - Traça de crides al sistema d'un procés
- `ltrace` - Traça de crides a llibreries
- `lsof` - Llista de fitxers oberts per processos (files, sockets, etc.)
- `fuser` - Identificar processos que utilitzen fitxers o sockets

#### Testing i Benchmarking

- `stress-ng` - Eina per generar càrrega al sistema (CPU, memòria, I/O, xarxa)
- `sysbench` - Benchmark de rendiment del sistema
- `dd` - Test de velocitat de disc

### 5. Processament i Filtratge de Dades (Tema Transversal)

#### Filtratge i Cerca

- `grep` - Cerca de patrons en text
- `egrep` / `grep -E` - Cerca amb expressions regulars avançades
- `ack` / `ag` - Eines de cerca ràpida en fitxers

#### Extracció i Transformació

- `awk` - Processament avançat de text per columnes
- `sed` - Editor de flux per transformar text
- `cut` - Extreure columnes específiques
- `tr` - Transformar o eliminar caràcters

#### Ordenació i Filtratge

- `sort` - Ordenar línies de text
- `uniq` - Eliminar o comptar línies duplicades
- `head` - Mostrar les primeres línies
- `tail` - Mostrar les últimes línies (útil amb `-f` per seguir logs)
- `less` / `more` - Paginadors de text

#### Comptabilitzar i Numerar

- `wc` - Comptar línies, paraules i caràcters
- `nl` - Numerar línies de sortida

#### Formatar la informació

- `column` - Formatar sortida en columnes alineades
- `printf` - Formatar sortida amb control precís

#### Processament de JSON

- `jq` - Processar i filtrar dades JSON (útil per APIs i configuracions)

#### Construcció de Comandes

- `xargs` - Construir i executar comandes des de l'entrada estàndard
- `|` (pipe) - Encadenar comandes passant sortida com entrada
