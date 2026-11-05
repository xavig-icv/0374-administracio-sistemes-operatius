## 01. Introducció a la Gestió de la Informació del Sistema

### Per què cal Recollir Informació del Sistema?

Un administrador no pot gestionar el que no coneix. Abans de configurar un servei, resoldre una incidència o decidir si cal ampliar un servidor, necessita dades concretes: quin maquinari té, quin sistema executa, quins serveis hi ha en marxa, qui s'hi connecta i com es comporten els recursos.

Situacions habituals en una empresa:

| Situació                                 | Pregunta que cal respondre          | Informació necessària                         |
| ---------------------------------------- | ----------------------------------- | --------------------------------------------- |
| Heretes un servidor sense documentació   | Què és aquesta màquina i què hi corre? | Sistema, maquinari, xarxa, serveis, usuaris |
| Els usuaris diuen que "va lent"          | On és el coll d'ampolla?            | CPU, memòria, disc, processos                 |
| Cal instal·lar una aplicació nova        | Té prou recursos?                   | RAM disponible, espai de disc, nuclis         |
| Auditoria de seguretat                   | Qui hi accedeix i per on?           | Usuaris, sessions, intents fallits, ports oberts |
| Un servei s'ha aturat                    | Per què ha caigut?                  | Estat del servei i registres (*logs*)         |

### Informació Estàtica i Informació Dinàmica

| Tipus                     | Què és                                              | Exemples                                                  | Fitxers del tema |
| ------------------------- | --------------------------------------------------- | --------------------------------------------------------- | ---------------- |
| **Estàtica** (inventari)  | Canvia poc: es consulta un cop i es documenta       | Model de CPU, RAM total, discos, versió del kernel, IP fixa | 03, 04, 05     |
| **Dinàmica** (estat)      | Canvia contínuament: s'ha de mesurar en el moment   | Càrrega de CPU, memòria disponible, processos, connexions | 06 a 11          |

La informació estàtica respon "**què tinc?**" i la dinàmica respon "**què està passant ara?**". Per diagnosticar un problema calen totes dues: saber que el servidor té 2 nuclis permet interpretar que una càrrega de 4 és excessiva.

### D'on Surt la Informació?

A Linux gairebé tota la informació del sistema es pot llegir com si fossin fitxers. Les ordres d'aquest tema llegeixen aquestes fonts i en presenten les dades de manera entenedora:

| Font                                    | Què conté                                                  | Ordres que la llegeixen         |
| --------------------------------------- | ---------------------------------------------------------- | ------------------------------- |
| `/proc`                                 | Sistema de fitxers virtual del kernel: processos, CPU, memòria | `ps`, `top`, `free`, `uptime` |
| `/sys`                                  | Dispositius i controladors que el kernel ha detectat       | `lsblk`, `lspci`, `lscpu`       |
| `/etc`                                  | Configuració del sistema                                   | `hostnamectl`, `getent`, netplan |
| `/run/utmp`, `/var/log/wtmp`, `/var/log/btmp` | Sessions actuals, històric de connexions, intents fallits | `who`, `w`, `last`, `lastb` |
| Diari de systemd (*journal*) i `/var/log` | Registres del sistema i dels serveis                     | `journalctl`, `dmesg`, `tail`   |

```bash
cat /proc/uptime                        # segons que fa que el sistema està engegat
cat /proc/loadavg                       # càrrega mitjana (la mateixa que mostra uptime)
cat /sys/class/net/enp0s8/address       # adreça MAC de la interfície enp0s8
```

### Usuari Normal o `sudo`?

La majoria d'ordres d'informació funcionen amb un usuari normal. Algunes necessiten `sudo` perquè accedeixen a dades sensibles:

| Ordre        | Per què necessita `sudo`                                                                 |
| ------------ | ---------------------------------------------------------------------------------------- |
| `dmidecode`  | Llegeix les taules del firmware (BIOS/UEFI)                                              |
| `lastb`      | `/var/log/btmp` només el pot llegir root: hi poden quedar contrasenyes escrites per error al camp d'usuari |
| `dmesg`      | A Ubuntu està restringit als usuaris normals (`kernel.dmesg_restrict = 1`)               |
| `ss -p`      | Sense `sudo` només mostra els processos del vostre usuari                                |
| `journalctl` | Els membres dels grups `adm` o `systemd-journal` ho poden llegir tot; la resta, només els seus registres |

`Nota:` L'usuari que es crea en instal·lar Ubuntu ja pertany als grups `sudo` i `adm`. Per això pot llegir els registres del sistema sense `sudo`.

### Ordres que no Estan Instal·lades

Ubuntu Server porta la majoria d'eines, però algunes s'han d'instal·lar. Si una ordre no existeix, el sistema suggereix el paquet:

```
$ iostat
Command 'iostat' not found, but can be installed with:
sudo apt install sysstat
```

| Ordre                                 | Paquet       |
| ------------------------------------- | ------------ |
| `lspci`                               | `pciutils`   |
| `lsusb`                               | `usbutils`   |
| `iostat`, `mpstat`, `pidstat`, `sar`  | `sysstat`    |
| `hwinfo`                              | `hwinfo`     |
| `tree`                                | `tree`       |
| `stress-ng`                           | `stress-ng`  |
| `netstat`, `ifconfig` (antigues)      | `net-tools`  |

### Com Trobar Ajuda

| Ordre                        | Ús                                                              |
| ---------------------------- | --------------------------------------------------------------- |
| `man ordre`                  | Manual complet (`/paraula` per cercar, `n` següent, `q` sortir) |
| `ordre --help`               | Resum ràpid de les opcions                                      |
| `apropos paraula`            | Cerca ordres relacionades amb una paraula (`apropos memory`)    |
| `type ordre`                 | Indica si és una ordre interna del shell o on és l'executable   |

### Flux de Treball: Primer Dia amb un Servidor

Quan rebeu un servidor desconegut, seguiu sempre el mateix ordre. Cada pas es desenvolupa en un fitxer del tema:

| Pas | Pregunta                | Ordres                                       | Fitxer |
| --- | ----------------------- | -------------------------------------------- | ------ |
| 1   | Quin sistema és?        | `hostnamectl`, `uname -r`                    | [03](./03_sistema_hardware.md) |
| 2   | Quins recursos té?      | `lscpu`, `free -h`, `lsblk`, `df -h`         | [03](./03_sistema_hardware.md) |
| 3   | Com està connectat?     | `ip -br a`, `ip route`, `ss -tulnp`          | [04](./04_info_xarxa.md) |
| 4   | Qui l'utilitza?         | `who`, `last`, `getent passwd`               | [05](./05_usuaris_sessions.md) |
| 5   | Què hi corre?           | `ps aux`, `systemctl list-units --type=service` | [07](./07_veure_processos.md), [09](./09_serveis_systemd.md) |
| 6   | Com es troba ara?       | `uptime`, `top`, `vmstat`                    | [10](./10_monitoritzacio.md) |
| 7   | Hi ha errors?           | `journalctl -p err -b`, `sudo dmesg`         | [11](./11_logs_diagnosi.md) |

El resultat d'aquest recorregut és l'**informe del servidor**: un document, o millor un script que el genera, amb les dades bàsiques de la màquina. A les pràctiques el construireu pas a pas amb un script de Bash (vegeu [02. Filtrar i Processar la Informació](./02_filtrar_informacio.md)).

### Relació amb Altres Temes

- **Tema 1 (Serveis de directori):** si el servidor és client LDAP, `getent passwd` també mostra els usuaris del directori.
- **Tema 3 (Automatització de tasques):** programar els scripts d'informe perquè s'executin sols (`cron`, temporitzadors de systemd).
- **Tema 4 (Administració remota):** obtenir la mateixa informació de servidors remots per SSH.
- **Tema 7 (Scripting):** scripts més complets (condicions, bucles, funcions) a partir de les ordres d'aquest tema.
