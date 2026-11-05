## 10. Dates i Temps

Les dates són imprescindibles en els scripts d'administració: per posar nom als fitxers de les còpies, per anotar quan ha passat cada cosa als registres i per mesurar quant triga una tasca.

### `date`

```bash
date '+%d/%m/%Y %H:%M'          # 05/11/2026 10:30
date +%F                        # 2026-11-05
date '+%F_%H%M'                 # 2026-11-05_1030 (per a noms de fitxer)
date -Iseconds                  # 2026-11-05T10:30:12+01:00 (format ISO 8601)
```

| Format      | Significat                                   | Exemple       |
| ----------- | -------------------------------------------- | ------------- |
| `%Y` / `%y` | Any (4 xifres / 2 xifres)                    | 2026 / 26     |
| `%m`        | Mes                                          | 11            |
| `%d`        | Dia                                          | 05            |
| `%H:%M:%S`  | Hora, minuts i segons                        | 10:30:12      |
| `%F`        | Data ISO (`%Y-%m-%d`)                        | 2026-11-05    |
| `%T`        | Hora (`%H:%M:%S`)                            | 10:30:12      |
| `%A` / `%a` | Dia de la setmana (nom / abreujat)           | Thursday / Thu |
| `%u`        | Dia de la setmana en número (1 = dilluns)    | 4             |
| `%j`        | Dia de l'any                                 | 309           |
| `%s`        | Segons des de l'1/1/1970 (*timestamp*)       | 1793871012    |

**Dates relatives** (opció `-d` de la versió GNU de `date`):

```bash
date -d yesterday +%F           # ahir
date -d "7 days ago" +%F        # fa 7 dies
date -d "+1 month" +%F          # d'aquí a un mes
date -d @1793871012             # converteix un timestamp en data
```

### Usos Habituals en Scripts

```bash
# Nom de fitxer amb la data
copia="/var/backups/etc_$(date +%F).tar.gz"

# Línia de registre amb la data i l'hora
echo "[$(date '+%F %T')] Còpia feta" >> /var/log/copies.log

# Una tasca només en dies feiners (1-5 = de dilluns a divendres)
if (( $(date +%u) <= 5 )); then
    echo "Avui és dia feiner"
fi
```

### Mesurar el Temps d'Execució

```bash
#!/bin/bash
# temps.sh - Calcula quant triga un conjunt de tasques
inici=$(date +%s)
sleep 2                         # simulació de tasques
sleep 3
final=$(date +%s)
echo "L'script ha trigat $(( final - inici )) segons"
```

```
L'script ha trigat 5 segons
```

Bash també té la variable `$SECONDS`, que compta els segons des que ha començat l'script, i l'ordre `time`, que mesura una sola ordre: `time tar czf copia.tar.gz /etc`.

### Esperar: `sleep`

`sleep` atura l'script el temps indicat: `sleep 5` (segons), `sleep 0.5`, `sleep 2m` (minuts), `sleep 1h` (hores).

### Resolució de Problemes

| Problema                                                  | Causa                                    | Solució                                                    |
| --------------------------------------------------------- | ---------------------------------------- | ---------------------------------------------------------- |
| Els mesos o dies `08` i `09` donen error en una operació  | Bash els llegeix en octal                | `$((10#$(date +%m)))`                                      |
| El nom del dia surt en anglès                             | Idioma del sistema                       | `LC_TIME=ca_ES.UTF-8 date +%A` (si l'idioma està instal·lat) |
| L'hora no és correcta                                     | La zona horària és UTC                   | `sudo timedatectl set-timezone Europe/Madrid`              |
| `date -d` no funciona en un altre sistema                 | És una opció de la versió GNU (Linux)    | A macOS i BSD la sintaxi és diferent                       |
