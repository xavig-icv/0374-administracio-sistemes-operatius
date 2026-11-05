## 12. Scripts d'Administració

Aquest fitxer reuneix el que s'ha vist al tema en quatre scripts complets d'ús real. Feu-los servir com a model per a les pràctiques i el projecte.

### 1. Crear Usuaris a partir d'un Fitxer

Fitxer `usuaris.csv` (una línia per usuari: `usuari;nom complet;grup`):

```
anna;Anna Puig;alumnes
marc;Marc Soler;alumnes
laia;Laia Font;professors
```

```bash
#!/bin/bash
# alta_usuaris.sh - Crea els usuaris d'un fitxer CSV (usuari;nom complet;grup)
# Ús: sudo ./alta_usuaris.sh usuaris.csv
set -euo pipefail

fitxer=${1:-}
credencials="credencials_$(date +%F).txt"

if [[ $EUID -ne 0 ]]; then
    echo "Executa l'script amb sudo" >&2
    exit 1
fi
if [[ ! -f "$fitxer" ]]; then
    echo "Ús: $0 fitxer.csv" >&2
    exit 2
fi

touch "$credencials" && chmod 600 "$credencials"   # només root el pot llegir
creats=0

while IFS=';' read -r usuari nom grup; do
    [[ -z "$usuari" || "$usuari" == \#* ]] && continue      # salta línies buides i comentaris
    getent group "$grup" > /dev/null || groupadd "$grup"
    if id "$usuari" &> /dev/null; then
        echo "Ja existeix: $usuari"
        continue
    fi
    useradd -m -s /bin/bash -c "$nom" -g "$grup" "$usuari"
    contrasenya=$(openssl rand -base64 12)
    echo "$usuari:$contrasenya" | chpasswd
    passwd -e "$usuari" > /dev/null                         # l'haurà de canviar en el primer inici
    echo "$usuari;$contrasenya" >> "$credencials"
    creats=$((creats + 1))
done < "$fitxer"

echo "Usuaris creats: $creats (contrasenyes inicials a $credencials)"
```

El fitxer de credencials conté contrasenyes: s'ha de lliurar a cada usuari per un canal segur i esborrar-lo després.

### 2. Còpia de Seguretat amb Rotació

```bash
#!/bin/bash
# copia_etc.sh - Còpia comprimida de /etc; conserva les còpies dels últims 7 dies
# Ús: sudo ./copia_etc.sh
set -euo pipefail

desti="/var/backups/etc"
dies=7
fitxer="$desti/etc_$(date +%F_%H%M).tar.gz"

mkdir -p "$desti"
if tar czf "$fitxer" -C / etc 2> /dev/null; then
    logger -t copia_etc "Còpia correcta: $fitxer ($(du -h "$fitxer" | cut -f1))"
else
    logger -t copia_etc -p user.err "ERROR en la còpia de /etc"
    exit 1
fi

# Rotació: esborra les còpies de més de $dies dies
find "$desti" -name "etc_*.tar.gz" -mtime +"$dies" -delete
```

`logger` escriu els missatges al registre del sistema: `journalctl -t copia_etc` mostra el resultat de cada execució (Tema 2, fitxer 11). Al Tema 3, aquest script es programa perquè s'executi cada nit.

### 3. Vigilar Serveis

```bash
#!/bin/bash
# vigila_serveis.sh - Reinicia els serveis aturats i ho registra
# Ús: sudo ./vigila_serveis.sh
serveis=("cron" "rsyslog" "slapd")

for servei in "${serveis[@]}"; do
    if ! systemctl is-active --quiet "$servei"; then
        logger -t vigila_serveis -p user.warning "$servei aturat: es reinicia"
        if systemctl restart "$servei"; then
            logger -t vigila_serveis "$servei reiniciat correctament"
        else
            logger -t vigila_serveis -p user.err "No s'ha pogut reiniciar $servei"
        fi
    fi
done
```

systemd ja pot reiniciar els serveis automàticament amb `Restart=` (Tema 2, fitxer 09). Un script com aquest és útil per a comprovacions més completes: per exemple, que el servei no només estigui actiu sinó que respongui al seu port.

### 4. Mètriques en Format JSON

El format **JSON** és el que fan servir les aplicacions web i les eines de monitorització per intercanviar dades. Construir-lo a mà amb `echo` és fàcil d'espatllar (cometes, comes); l'ordre `jq` (paquet `jq`) el genera sempre ben format:

```bash
#!/bin/bash
# metriques.sh - Desa les mètriques bàsiques del servidor en format JSON
set -euo pipefail

carrega=$(cut -d' ' -f1 /proc/loadavg)
mem_total=$(free -m | awk '/^Mem:/ {print $2}')
mem_disponible=$(free -m | awk '/^Mem:/ {print $7}')
disc=$(df --output=pcent / | tail -n 1 | tr -d ' %')

jq -n \
  --arg host "$(hostname)" \
  --arg data "$(date -Iseconds)" \
  --argjson carrega "$carrega" \
  --argjson mem_total "$mem_total" \
  --argjson mem_disponible "$mem_disponible" \
  --argjson disc "$disc" \
  '{hostname: $host, timestamp: $data, carrega: $carrega,
    memoria: {total_mb: $mem_total, disponible_mb: $mem_disponible},
    disc_arrel_percent: $disc}' > /tmp/metriques.json

cat /tmp/metriques.json
```

```json
{
  "hostname": "srv0374",
  "timestamp": "2026-11-05T10:30:12+01:00",
  "carrega": 0.08,
  "memoria": {
    "total_mb": 3916,
    "disponible_mb": 3412
  },
  "disc_arrel_percent": 47
}
```

`--arg` passa un valor com a text i `--argjson`, com a número. Per llegir un valor d'un fitxer JSON: `jq '.memoria.disponible_mb' /tmp/metriques.json`. Aquest script és el punt de partida del projecte de monitorització.

### Resolució de Problemes

| Problema                                              | Causa                                                    | Solució                                              |
| ----------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| `useradd: group 'x' does not exist`                   | El grup no s'ha creat                                    | `groupadd` abans de `useradd` (com a l'exemple)      |
| `chpasswd` o `useradd` donen `Permission denied`      | L'script no s'executa com a root                         | `sudo ./script.sh`                                   |
| `jq: command not found`                               | El paquet no està instal·lat                             | `sudo apt install jq`                                |
| `jq: invalid JSON text passed to --argjson`           | El valor no és un número vàlid (buit o amb `%`)          | Netejar el valor (`tr -d ' %'`) o passar-lo amb `--arg` |
| `find -delete` no esborra les còpies antigues         | `-mtime +7` compta dies complets des de l'última modificació | Comprovar-ho primer sense `-delete`              |
