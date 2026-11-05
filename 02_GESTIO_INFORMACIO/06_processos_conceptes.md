## 06. Processos: Conceptes Bàsics

### Per què cal Entendre els Processos?

Gairebé totes les incidències d'un servidor acaben en un procés: un programa que consumeix tota la CPU, un servei que no respon, una còpia de seguretat que s'atura quan es tanca la sessió. Per resoldre-les cal saber què és un procés, en quin estat es troba i com se li poden donar ordres.

| Situació                                                  | Concepte que cal conèixer                       |
| --------------------------------------------------------- | ----------------------------------------------- |
| El servidor va lent i la CPU està al 100%                 | Estats dels processos i prioritats              |
| Un servei no respon i no s'atura                          | Senyals (`SIGTERM`, `SIGKILL`) i l'estat `D`    |
| Apareixen desenes de processos `<defunct>`                | Processos zombi i el procés pare                |
| Una còpia de seguretat llarga es talla en tancar l'SSH    | Tasques en segon pla, `SIGHUP` i `nohup`        |
| Un procés desconegut consumeix CPU                        | Identificar-lo: PID, usuari, executable         |

### Programa i Procés

Un **programa** és un fitxer executable desat al disc (`/usr/bin/bash`). Un **procés** és un programa **en execució**: el kernel li assigna memòria, temps de CPU i un identificador. Un mateix programa pot generar molts processos alhora: si tres usuaris obren una sessió SSH, hi ha tres processos `bash` que executen el mateix fitxer.

Cada procés té:

| Atribut          | Descripció                                                     | Exemple           |
| ---------------- | -------------------------------------------------------------- | ----------------- |
| PID              | Identificador únic del procés                                  | `1520`            |
| PPID             | PID del procés pare (el que l'ha creat)                        | `1519`            |
| Usuari (UID)     | Propietari del procés: determina què pot fer                   | `xavi`            |
| Estat            | Què fa en aquest moment                                        | `S` (esperant)    |
| Prioritat (nice) | Preferència per obtenir temps de CPU                           | `0`               |
| Memòria          | Espai de memòria propi, aïllat de la resta de processos        | 5 MB              |
| Terminal         | Terminal des del qual s'ha llançat, si en té                   | `pts/0`           |
| Fitxers oberts   | Fitxers, sockets de xarxa i canonades que utilitza             | `/var/log/syslog` |

### Com es Creen els Processos

Tot procés el crea un altre procés, el seu **pare**. Quan escriviu `ls` al terminal:

1. `bash` es duplica (crida `fork`): el fill és una còpia de `bash` amb un PID nou.
2. El fill substitueix el seu programa per `/usr/bin/ls` (crida `exec`).
3. `ls` s'executa i acaba. El pare (`bash`) en recull el codi de sortida (`echo $?`).

Així es forma un **arbre de processos**:

```
systemd (PID 1)
 ├── cron
 ├── slapd
 └── sshd
      └── sshd (sessió de xavi)
           └── bash (PID 1520)
                └── ls (PID 1710)
```

A l'arrel hi ha **`systemd` (PID 1)**, el primer procés que arrenca el kernel i que llança tots els serveis. Al seu costat hi ha `kthreadd` (PID 2), el pare dels **fils del kernel**, que `ps` mostra entre claudàtors (`[kworker/0:1]`, `[ksoftirqd/0]`).

**Seqüència d'arrencada del sistema:**

```
Firmware (BIOS/UEFI) → GRUB → kernel → systemd (PID 1) → serveis → inici de sessió
```

L'arrencada dels serveis i el temps que hi dedica cadascun es veuen a [09. Serveis amb systemd](./09_serveis_systemd.md).

### Tipus de Processos

| Tipus                        | Característiques                                                    | Exemple                    |
| ---------------------------- | ------------------------------------------------------------------- | -------------------------- |
| Primer pla (*foreground*)    | Ocupa el terminal fins que acaba                                    | `nano fitxer.txt`          |
| Segon pla (*background*)     | S'executa sense bloquejar el terminal (`ordre &`)                   | `sleep 300 &`              |
| Dimoni (*daemon*) o servei   | S'executa sense terminal (`TTY ?`), normalment des de l'arrencada   | `sshd`, `cron`, `slapd`    |
| Fil del kernel               | Feina interna del kernel; no té cap programa associat               | `[kworker/0:1]`            |

Els noms de molts dimonis acaben en **d** (*daemon*). Dimonis habituals d'Ubuntu Server:

| Dimoni                | Funció                                                  |
| --------------------- | ------------------------------------------------------- |
| `systemd-journald`    | Recull els registres del sistema i dels serveis         |
| `systemd-logind`      | Gestiona les sessions dels usuaris                      |
| `systemd-networkd`    | Configura la xarxa (a partir de netplan)                |
| `systemd-resolved`    | Resolució DNS local                                     |
| `systemd-timesyncd`   | Sincronitza el rellotge per NTP                         |
| `sshd`                | Connexions SSH remotes                                  |
| `cron`                | Executa tasques programades (Tema 3)                    |
| `rsyslogd`            | Escriu els registres de text a `/var/log`               |
| `slapd`               | Servidor LDAP (Tema 1)                                  |

### Estats dels Processos

Un procés no s'executa tota l'estona: la major part del temps espera alguna cosa (que l'usuari escrigui, que arribi un paquet, que el disc retorni dades).

| Codi | Estat                     | Significat                                                                          |
| ---- | ------------------------- | ----------------------------------------------------------------------------------- |
| `R`  | *Running*                 | S'executa o està a punt per executar-se (a la cua de la CPU)                        |
| `S`  | *Sleeping*                | Espera un esdeveniment: teclat, xarxa, temporitzador. És l'estat més habitual        |
| `D`  | *Uninterruptible sleep*   | Espera una operació d'entrada/sortida (disc, NFS). No es pot interrompre, ni amb `kill -9` |
| `T`  | *Stopped*                 | Aturat amb `Ctrl+Z` o amb el senyal `SIGSTOP`                                       |
| `Z`  | *Zombie*                  | Ha acabat, però el pare encara no n'ha recollit el codi de sortida                  |
| `I`  | *Idle*                    | Fil del kernel inactiu                                                              |

**Cicle de vida d'un procés:**

```
                          creat (fork + exec)
                                   │
                                   ▼
 ┌──────────────┐  SIGSTOP  ┌──────────────┐   espera E/S    ┌──────────────┐
 │      T       │ ◄──────── │      R       │ ──────────────► │    S / D     │
 │    aturat    │ ────────► │ executant-se │ ◄────────────── │   esperant   │
 └──────────────┘  SIGCONT  └──────────────┘  esdeveniment   └──────────────┘
                                   │
                                   │ acaba (exit)
                                   ▼
                            ┌──────────────┐  el pare en recull
                            │   Z  zombi   │ ──────────────────► eliminat
                            └──────────────┘  el codi de sortida
```

### Processos Problemàtics

**Processos zombi (`Z`)**

Un zombi és un procés que ja ha acabat, però el seu pare no n'ha recollit el codi de sortida. No consumeix CPU ni memòria, només ocupa una entrada a la taula de processos (un PID). Apareix com a `<defunct>`:

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/'
```

```
   2310    2305 Z    [informe.sh] <defunct>
```

Un zombi **no es pot matar**, perquè ja està mort. La solució és actuar sobre el **pare** (`PPID 2305`): demanar-li que reculli els fills (`kill -CHLD 2305`) o, si no ho fa, aturar-lo o reiniciar el servei. Llavors `systemd` adopta el zombi i l'elimina. Uns quants zombis no són greus; centenars indiquen un programa mal fet que pot esgotar els PID disponibles.

**Processos orfes**

Si el pare acaba abans que el fill, el fill queda **orfe** i l'adopta `systemd` (PID 1). No és cap problema: és el mecanisme amb què es creen els dimonis.

```bash
bash -c 'sleep 300 &'              # el bash intern acaba i deixa el sleep orfe
ps -o pid,ppid,cmd -C sleep        # el PPID del sleep ara és 1
```

**Processos bloquejats (`D`)**

Esperen una operació d'entrada/sortida que no acaba: un disc que falla, un servidor NFS que no respon. No responen ni a `kill -9` fins que l'operació acaba. Molts processos en estat `D` fan pujar la càrrega del sistema sense que la CPU treballi (vegeu [10. Monitorització de Recursos](./10_monitoritzacio.md)).

### Procés, Fil i Tasca

| Concepte         | Què és                                                                          | Identificador          |
| ---------------- | ------------------------------------------------------------------------------- | ---------------------- |
| Procés           | Programa en execució amb memòria pròpia                                         | PID                    |
| Fil (*thread*)   | Línia d'execució dins d'un procés. Tots els fils d'un procés comparteixen la memòria | TID (`ps -eLf`, columna `LWP`) |
| Tasca (*job*)    | Ordre o canonada llançada des d'un shell, que es gestiona amb `jobs`, `fg` i `bg` | `%1`, `%2`            |

Un servidor web o una base de dades fan servir molts fils per atendre diverses peticions alhora sense haver de crear un procés per a cada una. `pstree` mostra els fils entre claus: `{slapd}`.

### Planificació de la CPU

Si hi ha més processos preparats (`R`) que CPU, el **planificador** del kernel els reparteix el temps de CPU a torns molt curts (mil·lisegons). L'usuari té la sensació que tot s'executa alhora.

- El planificador actual de Linux (EEVDF des del kernel 6.6; abans, CFS) reparteix la CPU de manera **proporcional**: si dos processos competeixen per la mateixa CPU, cadascun en rep la meitat.
- El **valor *nice*** (de -20 a 19) modifica aquest repartiment: un procés amb *nice* alt "cedeix" CPU als altres. Es veu a [08. Senyals, Prioritats i Control de Tasques](./08_senyals_prioritats.md).
- Si hi ha més processos `R` que CPU de manera continuada, es forma una cua i la **càrrega** del sistema augmenta (fitxer [10](./10_monitoritzacio.md)).

### Senyals

Un **senyal** és un missatge que el kernel o un altre procés envia a un procés: acaba, atura't, continua, torna a llegir la configuració... `Ctrl+C` envia el senyal `SIGINT` al procés de primer pla i `Ctrl+Z`, `SIGTSTP`. Els senyals i les ordres per enviar-los (`kill`, `killall`, `pkill`) es veuen a [08. Senyals, Prioritats i Control de Tasques](./08_senyals_prioritats.md).
