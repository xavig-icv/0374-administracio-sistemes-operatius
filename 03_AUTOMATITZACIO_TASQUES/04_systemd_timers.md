## 04. Temporitzadors de systemd

Els **temporitzadors** (*timers*) de systemd són l'alternativa moderna a `cron`. Una tasca programada amb systemd té dues unitats (Tema 2, fitxer 09):

- Un **servei** (`.service`): què s'executa.
- Un **temporitzador** (`.timer`): quan s'executa. Activa el servei que té el mateix nom.

Ubuntu ja fa servir temporitzadors per a moltes tasques del sistema (`logrotate.timer`, `apt-daily.timer`, `fstrim.timer`...).

```bash
systemctl list-timers
```

```
NEXT                        LEFT     LAST                        PASSED  UNIT                     ACTIVATES
Fri 2026-11-06 00:00:00 CET 13h left Thu 2026-11-05 00:00:01 CET 10h ago logrotate.timer          logrotate.service
Fri 2026-11-06 07:00:00 CET 20h left Thu 2026-11-05 07:00:03 CET 3h ago  informe-srv0374.timer    informe-srv0374.service
```

| Columna         | Significat                                    |
| --------------- | --------------------------------------------- |
| `NEXT` / `LEFT` | Pròxima execució i temps que falta            |
| `LAST` / `PASSED` | Última execució i temps que ha passat       |
| `UNIT`          | El temporitzador                              |
| `ACTIVATES`     | El servei que executa                         |

### Crear un Temporitzador

Exemple: executar cada dia a les 7:00 l'script d'informe del Tema 2.

**1. El servei** (`/etc/systemd/system/informe-srv0374.service`):

```ini
[Unit]
Description=Informe diari del servidor srv0374

[Service]
Type=oneshot
ExecStart=/usr/local/bin/informe-srv0374.sh
```

`Type=oneshot` indica que és una tasca que s'executa i acaba (no un dimoni que es queda en marxa).

**2. El temporitzador** (`/etc/systemd/system/informe-srv0374.timer`):

```ini
[Unit]
Description=Executa l'informe del servidor cada dia a les 7:00

[Timer]
OnCalendar=*-*-* 07:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

**3. Activar-lo:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now informe-srv0374.timer     # s'habilita el temporitzador, no el servei
systemctl list-timers informe-srv0374.timer
```

**4. Provar-lo sense esperar** i consultar-ne el resultat:

```bash
sudo systemctl start informe-srv0374.service          # executa la tasca ara
journalctl -u informe-srv0374.service -n 20           # sortida de l'script i resultat
```

Tot el que l'script escriu per pantalla queda al registre del servei: no cal redirigir la sortida com a `cron`.

### Quan s'Executa: `OnCalendar`

| Expressió                  | Significat                                     |
| -------------------------- | ---------------------------------------------- |
| `*-*-* 07:00:00`           | Cada dia a les 7:00 (any-mes-dia hora)          |
| `daily`                    | Cada dia a les 00:00                           |
| `hourly`, `weekly`, `monthly` | Cada hora, cada dilluns a les 00:00, el dia 1 de cada mes |
| `Mon..Fri 08:30`           | De dilluns a divendres a les 8:30              |
| `Sat,Sun 10:00`            | Dissabtes i diumenges a les 10:00              |
| `*-*-01 02:00`             | El dia 1 de cada mes a les 2:00                |
| `*:0/15`                   | Cada 15 minuts (minuts 0, 15, 30 i 45)         |

`systemd-analyze calendar` comprova una expressió i indica quan serà la propera execució:

```bash
systemd-analyze calendar "Mon..Fri 08:30"
```

```
  Original form: Mon..Fri 08:30
Normalized form: Mon..Fri *-*-* 08:30:00
    Next elapse: Fri 2026-11-06 08:30:00 CET
       From now: 22h left
```

**Temporitzadors relatius** (en lloc d'una hora fixa):

| Directiva              | Significat                                                  |
| ---------------------- | ----------------------------------------------------------- |
| `OnBootSec=5min`       | 5 minuts després d'engegar el sistema                       |
| `OnUnitActiveSec=1h`   | 1 hora després de l'última execució (repetició periòdica)   |

### Opcions Útils

| Directiva                 | Efecte                                                                       |
| ------------------------- | ---------------------------------------------------------------------------- |
| `Persistent=true`         | Si l'equip estava apagat a l'hora prevista, la tasca s'executa en engegar-lo (com `anacron`) |
| `RandomizedDelaySec=10min`| Retard aleatori de fins a 10 minuts, perquè molts servidors no facin la tasca alhora |
| `Nice=19` (al servei)     | La tasca s'executa amb la prioritat mínima (Tema 2)                          |
| `User=nom` (al servei)    | La tasca s'executa amb aquest usuari en lloc de root                         |

### `cron` o Temporitzadors de systemd?

| Aspecte                        | `cron`                                  | Temporitzadors de systemd                    |
| ------------------------------ | --------------------------------------- | -------------------------------------------- |
| Configuració                   | Una línia                               | Dos fitxers (servei i temporitzador)         |
| Registre de la sortida         | Cal redirigir-la                        | Automàtic al diari (`journalctl -u`)         |
| Tasques perdudes per apagada   | Es perden (cal `anacron`)               | `Persistent=true`                            |
| Provar la tasca a mà           | Executar l'script                       | `systemctl start servei` (amb el mateix entorn) |
| Límits de recursos i usuari    | Limitats                                | Totes les opcions dels serveis (`Nice=`, `User=`, `CPUQuota=`...) |
| Saber quan s'executarà         | Interpretar la línia                    | `systemctl list-timers`                      |

`cron` és més ràpid de configurar i el trobareu a tots els sistemes UNIX. Els temporitzadors donen més control i un registre millor. Cal conèixer totes dues eines.

### Resolució de Problemes

| Problema                                               | Causa                                                        | Solució                                                   |
| ------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------------- |
| El temporitzador no apareix a `list-timers`            | No s'ha habilitat o no s'ha fet `daemon-reload`              | `sudo systemctl daemon-reload` i `enable --now` del `.timer` |
| S'ha habilitat el `.service` en lloc del `.timer`      | Confusió entre les dues unitats                              | Habilitar el `.timer`                                     |
| `Failed to parse calendar specification`               | Expressió `OnCalendar` incorrecta                            | Comprovar-la amb `systemd-analyze calendar`               |
| El servei falla amb `status=203/EXEC`                  | L'script no existeix, no és executable o no té *shebang*     | Revisar `ExecStart` i `chmod 755` (Tema 2, fitxer 09)     |
| La tasca no s'ha fet perquè el servidor estava apagat  | Falta `Persistent=true`                                      | Afegir-lo al temporitzador                                |
