## 02. Tasques Puntuals: `at` i `batch`

`at` programa una tasca perquè s'executi **una sola vegada** en un moment concret: reiniciar un servei aquesta nit, fer una còpia abans d'una actualització, apagar un equip d'aquí a una hora.

### Instal·lació

```bash
sudo apt install at
systemctl status atd                 # el dimoni que executa les tasques
```

### Programar una Tasca

`at` llegeix les ordres de l'entrada estàndard. S'escriuen una per línia i s'acaba amb `Ctrl+D`:

```
$ at now + 5 minutes
warning: commands will be executed using /bin/sh
at> /usr/local/bin/copia_etc.sh >> /tmp/copia.log 2>&1
at> <EOT>
job 3 at Thu Nov  5 10:40:00 2026
```

`<EOT>` apareix en prémer `Ctrl+D`. Les ordres s'executen amb `/bin/sh`, des del directori i amb les variables d'entorn que hi havia en programar-les.

**Altres maneres de passar-hi les ordres:**

```bash
echo "systemctl restart slapd" | sudo at 23:00             # des d'una canonada
at -f /usr/local/bin/informe-srv0374.sh 07:00 tomorrow     # des d'un fitxer
```

**Formats de temps:**

| Format                          | Significat                               |
| ------------------------------- | ---------------------------------------- |
| `now + 30 minutes`              | D'aquí a 30 minuts (`hours`, `days`, `weeks`) |
| `22:00`                         | Avui a les 22:00 (o demà, si ja han passat) |
| `10:00 tomorrow`                | Demà a les 10:00                         |
| `09:00 2026-11-20`              | Un dia concret                           |
| `midnight`, `noon`, `teatime`   | 00:00, 12:00, 16:00                      |

### Gestionar les Tasques

| Ordre        | Funció                                           |
| ------------ | ------------------------------------------------ |
| `atq`        | Llista les tasques pendents (`at -l`)            |
| `at -c 3`    | Mostra el contingut de la tasca 3                |
| `atrm 3`     | Esborra la tasca 3 (`at -r 3`)                   |

```
$ atq
3	Thu Nov  5 10:40:00 2026 a xavi
```

Cada línia mostra el número de la tasca, quan s'executarà, la cua (`a`) i l'usuari. Un usuari normal només veu les seves tasques; root les veu totes.

### `batch`: Quan el Sistema Estigui Lliure

`batch` funciona com `at`, però no indica cap hora: executa la tasca quan la càrrega del sistema és baixa. És útil per a tasques pesades que no tenen pressa.

```bash
echo "tar czf /var/backups/home.tar.gz -C / home" | sudo batch
```

### Sortida de les Tasques

El que una tasca escriu per pantalla s'envia per correu a l'usuari. Si el servidor no té cap servidor de correu configurat, es perd. Per això convé **redirigir sempre la sortida a un fitxer** o fer servir `logger` dins de l'script.

### Control d'Accés

| Fitxer          | Efecte                                                         |
| --------------- | -------------------------------------------------------------- |
| `/etc/at.allow` | Si existeix, **només** els usuaris que hi apareixen poden fer servir `at` |
| `/etc/at.deny`  | Si no existeix `at.allow`, els usuaris que hi apareixen no poden fer-lo servir |

### Alternativa amb systemd

`systemd-run` crea un temporitzador temporal que executa una ordre una sola vegada i en desa el resultat al registre:

```bash
sudo systemd-run --on-active=30min /usr/local/bin/copia_etc.sh
sudo systemd-run --on-calendar="2026-11-20 09:00" /usr/local/bin/copia_etc.sh
systemctl list-timers                 # el temporitzador temporal apareix a la llista
```

### Resolució de Problemes

| Problema                                       | Causa                                        | Solució                                            |
| ---------------------------------------------- | -------------------------------------------- | -------------------------------------------------- |
| `at: command not found`                        | El paquet no està instal·lat                 | `sudo apt install at`                              |
| `Can't open /var/run/atd.pid to signal atd`    | El dimoni `atd` no està en marxa             | `sudo systemctl enable --now atd`                  |
| La tasca s'executa però no es veu cap resultat | La sortida s'ha enviat per correu i s'ha perdut | Redirigir-la a un fitxer o fer servir `logger`  |
| `You do not have permission to use at.`        | L'usuari és a `at.deny` o no és a `at.allow` | Revisar els fitxers de control d'accés             |
| Una ordre de bash falla dins de la tasca       | Les tasques s'executen amb `/bin/sh`         | Posar les ordres en un script amb `#!/bin/bash`    |
