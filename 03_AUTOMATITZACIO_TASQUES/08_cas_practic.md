## 08. Cas Pràctic: Automatitzar l'Informe Diari del Servidor

**Objectiu:** cada dia a les 7:00, el servidor `srv0374` ha de generar un informe del seu estat, desar-lo a `/var/log/informes/`, deixar-ne constància al registre del sistema i conservar només els informes de les últimes 4 setmanes. Aquest cas reuneix el que s'ha vist al tema i és la base de la pràctica de programació de tasques.

### Pas 1. Preparar l'Script per a l'Execució Desatesa

Es parteix de l'script d'informe del Tema 2 i s'adapta seguint les bones pràctiques del Tema 7: rutes absolutes, cap pregunta a l'usuari, missatge al registre i codi de sortida correcte.

```bash
#!/bin/bash
# /usr/local/bin/informe-srv0374.sh - Informe diari de l'estat del servidor
set -euo pipefail

directori="/var/log/informes"
fitxer="$directori/informe-$(date +%F).txt"
mkdir -p "$directori"

{
    echo "=== INFORME DEL SERVIDOR $(hostname) - $(date '+%d/%m/%Y %H:%M') ==="
    echo "Temps engegat: $(uptime -p)"
    echo "Càrrega (1, 5, 15 min): $(cut -d' ' -f1-3 /proc/loadavg)"
    echo "Memòria disponible: $(free -h | awk '/^Mem:/ {print $7}')"
    echo "Disc arrel ocupat: $(df -h / | awk 'NR==2 {print $5}')"
    echo "Unitats amb errors: $(systemctl --failed --no-legend | wc -l)"
    echo "Contrasenyes SSH incorrectes (24 h): $(journalctl -u ssh --since yesterday | grep -c 'Failed password' || true)"
} > "$fitxer"

# Una línia de resum per dia (aquest fitxer el rota logrotate)
echo "$(date +%F) càrrega=$(cut -d' ' -f1 /proc/loadavg) disc=$(df --output=pcent / | tail -n 1 | tr -d ' ')" >> "$directori/resum.log"

# Conserva només els informes de les últimes 4 setmanes
find "$directori" -name "informe-*.txt" -mtime +28 -delete

logger -t informe-srv0374 "Informe generat: $fitxer"
```

```bash
sudo install -m 755 informe-srv0374.sh /usr/local/bin/       # copia l'script amb permisos 755
sudo /usr/local/bin/informe-srv0374.sh                       # prova manual
cat /var/log/informes/informe-$(date +%F).txt
```

```
=== INFORME DEL SERVIDOR srv0374 - 05/11/2026 10:45 ===
Temps engegat: up 2 hours, 16 minutes
Càrrega (1, 5, 15 min): 0.08 0.05 0.01
Memòria disponible: 3.3Gi
Disc arrel ocupat: 47%
Unitats amb errors: 0
Contrasenyes SSH incorrectes (24 h): 17
```

### Pas 2. Programar-lo

**Opció A: temporitzador de systemd.** Es fan servir les unitats `informe-srv0374.service` i `informe-srv0374.timer` del [fitxer 04](./04_systemd_timers.md) (cada dia a les 7:00, amb `Persistent=true`):

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now informe-srv0374.timer
```

**Opció B: `cron`.** Un fitxer a `/etc/cron.d/` (el nom, sense punts):

```
# /etc/cron.d/informe-srv0374
0 7 * * *  root  /usr/local/bin/informe-srv0374.sh
```

Les dues opcions fan el mateix. Amb el temporitzador, si el servidor està apagat a les 7:00 l'informe es genera en engegar-lo, i la sortida de l'script queda al diari del servei.

### Pas 3. Rotar el Resum

El fitxer `resum.log` creix una línia cada dia. La configuració `/etc/logrotate.d/informes` del [fitxer 05](./05_logrotate.md) el rota cada setmana i en conserva 4 còpies. Els informes diaris (`informe-*.txt`) no cal rotar-los: l'script mateix esborra els de més de 28 dies.

### Pas 4. Enviar-lo a l'Altre Servidor (Opcional)

Amb la clau SSH de còpies del [fitxer 06](./06_copies_seguretat.md), n'hi ha prou d'afegir una línia al final de l'script:

```bash
rsync -az -e "ssh -i /root/.ssh/id_copies" "$fitxer" copies@192.168.1.2:/srv/copies/informes/
```

### Pas 5. Comprovar que Tot Funciona

| Comprovació                                  | Ordre                                                    |
| -------------------------------------------- | -------------------------------------------------------- |
| El temporitzador està programat              | `systemctl list-timers informe-srv0374.timer`            |
| L'última execució ha acabat bé               | `systemctl status informe-srv0374.service` (`status=0/SUCCESS`) |
| Missatges de l'script                        | `journalctl -t informe-srv0374`                          |
| S'han generat els informes                   | `ls -l /var/log/informes/`                               |
| La rotació està ben configurada              | `sudo logrotate -d /etc/logrotate.d/informes`            |

### Resolució de Problemes

| Problema                                                  | Causa                                                   | Solució                                                  |
| --------------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| L'script funciona a mà però el servei falla               | Error que només passa amb l'entorn del servei           | `journalctl -u informe-srv0374.service` mostra el motiu  |
| `status=203/EXEC`                                         | L'script no és executable o no té *shebang*             | `sudo chmod 755` i `#!/bin/bash` a la primera línia      |
| L'informe no té dades d'algun apartat                     | Una ordre necessita permisos o no existeix              | Executar l'script amb `bash -x` per veure quina falla    |
| Amb `cron`, l'informe no es genera                        | Fitxer de `/etc/cron.d/` amb punt al nom o sense usuari | Nom sense punts i camp de l'usuari                       |
