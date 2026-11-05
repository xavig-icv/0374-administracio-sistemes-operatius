## 09. Serveis amb systemd

Un **servei** és un programa que s'executa en segon pla de manera permanent per oferir una funció: `sshd` accepta connexions remotes, `cron` executa tasques programades, `slapd` respon les consultes LDAP. A Ubuntu, **systemd** (el procés amb PID 1) arrenca, atura, vigila i reinicia tots els serveis.

### Unitats de systemd

systemd gestiona **unitats** (*units*). Cada unitat es defineix en un fitxer de text i el seu tipus s'indica amb l'extensió:

| Tipus       | Funció                                                            | Exemple                    |
| ----------- | ----------------------------------------------------------------- | -------------------------- |
| `.service`  | Un servei (dimoni o script)                                       | `ssh.service`, `cron.service` |
| `.socket`   | Un port o socket que engega un servei quan rep una connexió       | `ssh.socket`               |
| `.timer`    | Executa un servei en un moment determinat (Tema 3)                | `logrotate.timer`          |
| `.target`   | Agrupa unitats per formar un estat del sistema                    | `multi-user.target`        |
| `.mount`    | Un sistema de fitxers muntat                                      | `boot.mount`               |

**On són els fitxers de les unitats?**

| Directori                    | Contingut                                                             |
| ---------------------------- | --------------------------------------------------------------------- |
| `/usr/lib/systemd/system/`   | Unitats que instal·len els paquets. No s'han de modificar             |
| `/etc/systemd/system/`       | Unitats i modificacions de l'administrador. **Tenen prioritat**       |
| `/run/systemd/system/`       | Unitats temporals, generades durant l'execució                        |

### `systemctl`: Gestionar Serveis

| Ordre                                   | Funció                                                       |
| --------------------------------------- | ------------------------------------------------------------ |
| `systemctl status ssh`                  | Estat del servei, processos i últims missatges               |
| `sudo systemctl start ssh`              | Engega el servei ara                                         |
| `sudo systemctl stop ssh`               | Atura el servei ara                                          |
| `sudo systemctl restart ssh`            | Atura i torna a engegar el servei                            |
| `sudo systemctl reload ssh`             | Torna a llegir la configuració sense aturar el servei        |
| `sudo systemctl enable ssh`             | El servei s'engegarà a cada arrencada                        |
| `sudo systemctl disable ssh`            | El servei no s'engegarà a l'arrencada                        |
| `sudo systemctl enable --now ssh`       | Habilita i engega en un sol pas                              |
| `systemctl is-active ssh`               | Respon `active` o `inactive` (útil en scripts)               |
| `systemctl is-enabled ssh`              | Respon `enabled` o `disabled`                                |
| `sudo systemctl mask servei`            | Impedeix que el servei es pugui engegar (ni a mà)            |
| `systemctl cat ssh`                     | Mostra el fitxer de la unitat                                |

**`start` i `enable` són coses diferents:** `start` engega el servei **ara**, però no fa que s'engegui en reiniciar; `enable` fa que s'engegui **a cada arrencada**, però no l'engega ara. Per això s'acostuma a fer servir `enable --now`.

**`restart` o `reload`?** `restart` atura el servei i tanca les connexions actives; `reload` només li fa llegir la configuració de nou (si el servei ho permet). Sempre que es pugui, `reload`.

### Estat d'un Servei

```bash
systemctl status cron
```

```
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-05 08:29:46 CET; 1h 16min ago
       Docs: man:cron(8)
   Main PID: 780 (cron)
      Tasks: 1 (limit: 4551)
     Memory: 1.1M (peak: 1.6M)
        CPU: 35ms
     CGroup: /system.slice/cron.service
             └─780 /usr/sbin/cron -f -P

Nov 05 09:17:01 srv0374 CRON[1612]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
Nov 05 09:17:01 srv0374 CRON[1613]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
```

| Línia                          | Significat                                                                  |
| ------------------------------ | --------------------------------------------------------------------------- |
| `●` (punt)                     | Verd: actiu. Blanc: inactiu. Vermell: error                                 |
| `Loaded: ... enabled`          | Fitxer de la unitat i si s'engega a l'arrencada. `preset` és el valor per defecte de la distribució |
| `Active: active (running)`     | Estat actual i des de quan                                                  |
| `Main PID`                     | PID del procés principal                                                    |
| `Tasks`, `Memory`, `CPU`       | Recursos que consumeix el servei (tots els seus processos)                  |
| `CGroup`                       | Processos que pertanyen al servei                                           |
| Línies finals                  | Últims missatges del servei al registre ([11](./11_logs_diagnosi.md))       |

**Estats possibles (`Active:`):**

| Estat                  | Significat                                                                   |
| ---------------------- | ---------------------------------------------------------------------------- |
| `active (running)`     | En marxa, amb almenys un procés                                              |
| `active (exited)`      | Ha fet la seva feina i ha acabat correctament (serveis que s'executen un cop) |
| `active (waiting)`     | Actiu, esperant un esdeveniment (per exemple, un temporitzador)              |
| `inactive (dead)`      | Aturat                                                                       |
| `activating (auto-restart)` | El procés ha mort i systemd el tornarà a engegar                        |
| `failed`               | Ha fallat: no ha pogut arrencar o ha acabat amb error                        |

**Llistar serveis:**

```bash
systemctl list-units --type=service --state=running   # serveis en marxa
systemctl --failed                                    # unitats que han fallat
systemctl list-unit-files --type=service              # tots els serveis instal·lats i si estan habilitats
```

### Arrencada del Sistema: Targets i `systemd-analyze`

Un **target** és un conjunt d'unitats que defineix un estat del sistema:

| Target               | Estat del sistema                                                  |
| -------------------- | ------------------------------------------------------------------ |
| `multi-user.target`  | Multiusuari amb xarxa, sense entorn gràfic (servidors)             |
| `graphical.target`   | `multi-user.target` + entorn gràfic                                |
| `rescue.target`      | Mode de rescat: només root, sense xarxa, per reparar el sistema    |
| `emergency.target`   | Mínim possible: sistema de fitxers arrel en només lectura          |

```bash
systemctl get-default                          # target amb què arrenca el sistema
systemd-analyze                                # temps total d'arrencada
systemd-analyze blame | head -n 5              # unitats que han trigat més
systemd-analyze critical-chain                 # cadena d'unitats que ha marcat el temps d'arrencada
```

```
Startup finished in 3.012s (kernel) + 9.845s (userspace) = 12.857s

5.912s systemd-networkd-wait-online.service
1.203s snapd.seeded.service
 845ms cloud-init-local.service
 612ms dev-mapper-ubuntu\x2d\x2dvg\x2dubuntu\x2d\x2dlv.device
 402ms apparmor.service
```

`systemd-networkd-wait-online` espera que **totes** les interfícies de netplan estiguin configurades. Si una interfície no té enllaç, l'arrencada es pot quedar esperant fins a 2 minuts. Es resol marcant-la com a opcional a netplan (`optional: true`).

### Crear un Servei Propi

Qualsevol script es pot convertir en un servei: systemd l'engegarà a l'arrencada, en recollirà la sortida al registre i el tornarà a engegar si falla.

**1. L'script** (`/usr/local/bin/monitor-carrega.sh`): escriu la càrrega i la memòria disponible cada 30 segons.

```bash
#!/bin/bash
# Registra la càrrega del sistema cada 30 segons
while true; do
    echo "Càrrega: $(cut -d' ' -f1-3 /proc/loadavg) | Memòria disponible: $(free -m | awk '/^Mem:/ {print $7}') MB"
    sleep 30
done
```

```bash
sudo chmod 755 /usr/local/bin/monitor-carrega.sh
```

**2. La unitat** (`/etc/systemd/system/monitor-carrega.service`):

```ini
[Unit]
Description=Monitor de càrrega del servidor

[Service]
Type=simple
ExecStart=/usr/local/bin/monitor-carrega.sh
User=nobody
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

| Directiva                    | Significat                                                                  |
| ---------------------------- | --------------------------------------------------------------------------- |
| `Description`                | Nom descriptiu que mostren `status` i el registre                           |
| `Type=simple`                | El procés d'`ExecStart` és el servei (no es desdobla en segon pla)          |
| `ExecStart`                  | Ordre que s'executa. Ruta absoluta                                          |
| `User=nobody`                | Usuari amb què s'executa. Un servei no ha de funcionar com a root si no ho necessita |
| `Restart=on-failure`         | Torna a engegar el servei si acaba amb error o el mata un senyal com `SIGKILL` |
| `RestartSec=5`               | Espera 5 segons abans de tornar-lo a engegar                                |
| `WantedBy=multi-user.target` | Amb `enable`, el servei s'engega en arribar a l'estat multiusuari           |

**3. Activar-lo i comprovar-lo:**

```bash
sudo systemctl daemon-reload                   # llegir els fitxers d'unitat nous o modificats
sudo systemctl enable --now monitor-carrega
systemctl status monitor-carrega
journalctl -u monitor-carrega -f               # la sortida de l'script va al registre (sortir amb Ctrl+C)
```

```
Nov 05 10:20:01 srv0374 monitor-carrega.sh[2345]: Càrrega: 0.08 0.05 0.01 | Memòria disponible: 3412 MB
Nov 05 10:20:31 srv0374 monitor-carrega.sh[2345]: Càrrega: 0.05 0.04 0.00 | Memòria disponible: 3410 MB
```

**4. Comprovar que es reinicia sol:**

```bash
systemctl show -p MainPID monitor-carrega      # MainPID=2345
sudo kill -9 2345
systemctl status monitor-carrega               # activating (auto-restart) i, al cap de 5 s, un PID nou
journalctl -u monitor-carrega -n 10
```

Fragment del registre:

```
Nov 05 10:21:12 srv0374 systemd[1]: monitor-carrega.service: Main process exited, code=killed, status=9/KILL
Nov 05 10:21:12 srv0374 systemd[1]: monitor-carrega.service: Failed with result 'signal'.
Nov 05 10:21:17 srv0374 systemd[1]: monitor-carrega.service: Scheduled restart job, restart counter is at 1.
Nov 05 10:21:17 srv0374 systemd[1]: Started monitor-carrega.service - Monitor de càrrega del servidor.
```

Amb `Restart=on-failure`, un `kill` normal (`SIGTERM`) es considera una aturada correcta i el servei **no** es reinicia. Amb `kill -9`, sí. Si el servei s'ha de reiniciar sempre, cal `Restart=always`. Per aturar-lo de debò, feu servir `systemctl stop`.

**Modificar un servei del sistema:** no editeu els fitxers de `/usr/lib/systemd/system/`, perquè una actualització els sobreescriuria. `sudo systemctl edit nom` crea un fitxer de modificacions a `/etc/systemd/system/nom.service.d/` que té prioritat sobre l'original.

### Resolució de Problemes

| Problema                                                          | Causa                                                        | Solució                                             |
| ----------------------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| `Unit nom.service not found`                                      | Nom incorrecte o paquet no instal·lat                        | `systemctl list-unit-files \| grep nom`             |
| `Warning: The unit file ... changed on disk`                      | S'ha modificat el fitxer de la unitat                        | `sudo systemctl daemon-reload`                      |
| `status=203/EXEC`                                                 | L'script no existeix, no és executable o no té `#!/bin/bash` | Revisar la ruta d'`ExecStart` i `chmod 755`         |
| `status=217/USER`                                                 | L'usuari de `User=` no existeix                              | Crear l'usuari o corregir el nom                    |
| `The unit files have no installation config`                      | Falta la secció `[Install]`                                  | Afegir `WantedBy=multi-user.target`                 |
| `Start request repeated too quickly`                              | El servei falla en arrencar i systemd ha deixat de provar-ho | Corregir la causa i `sudo systemctl reset-failed nom` |
| El servei funciona però no s'engega en reiniciar                  | Està engegat (`start`) però no habilitat                     | `sudo systemctl enable nom`                         |
| `systemctl is-enabled ssh` diu `disabled`, però l'SSH funciona    | A Ubuntu 24.04 l'SSH s'activa per socket                     | `systemctl status ssh.socket`                       |
