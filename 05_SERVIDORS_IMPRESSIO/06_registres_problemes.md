## 06. Registres i Resolució de Problemes

### Fitxers de Registre de CUPS

| Fitxer                        | Contingut                                                             |
| ----------------------------- | --------------------------------------------------------------------- |
| `/var/log/cups/access_log`    | Cada petició al servidor (qui s'hi connecta, des d'on i què demana)   |
| `/var/log/cups/error_log`     | Errors i missatges del servei i dels treballs                         |
| `/var/log/cups/page_log`      | Una línia per pàgina o treball imprès: impressora, usuari, treball, data i pàgines |

```bash
sudo tail -f /var/log/cups/error_log
sudo tail -n 5 /var/log/cups/page_log
```

```
PDF xavi 12 [05/Nov/2026:10:31:02 +0100] total 1 - localhost informe.txt - -
PDF anonymous 13 [05/Nov/2026:10:35:47 +0100] total 3 - 192.168.1.50 document.pdf - -
```

Amb les eines del Tema 2 es poden fer estadístiques del `page_log`, per exemple les pàgines impreses per cada usuari:

```bash
sudo awk '{pagines[$2] += $7} END {for (u in pagines) print u, pagines[u]}' /var/log/cups/page_log
```

**Registre detallat:** per diagnosticar un problema, `sudo cupsctl --debug-logging` activa el nivell de detall màxim a `error_log`; `sudo cupsctl --no-debug-logging` el torna a desactivar.

### Política d'Error de les Impressores

Quan un treball falla (per exemple, perquè la impressora no respon), la política d'error indica què fa CUPS:

| Política              | Efecte                                                        |
| --------------------- | ------------------------------------------------------------- |
| `stop-printer`        | Atura la impressora (per defecte). Cal reactivar-la a mà      |
| `retry-job`           | Torna a intentar el treball més tard                          |
| `retry-current-job`   | Torna a intentar el treball immediatament                     |
| `abort-job`           | Cancel·la el treball i continua amb el següent                |

```bash
sudo lpadmin -p HP-Aula1 -o printer-error-policy=retry-job
```

### Problemes Habituals

| Problema                                              | Com detectar-lo                                      | Solució                                                      |
| ----------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| Els treballs no surten i la impressora diu `stopped`  | `lpstat -p` (`disabled`)                             | Resoldre la causa (`error_log`) i `cupsenable`               |
| `Filter failed`                                       | `error_log` indica quin filtre ha fallat             | Controlador incorrecte: provar `-m everywhere` o el model adequat |
| La impressora de xarxa no respon                      | `ping` a la impressora; `lpstat -v` per veure l'adreça | Revisar l'adreça IP i la xarxa                             |
| Un client no veu les impressores compartides          | `access_log` (arriba la petició?)                    | Tallafoc, `cupsctl --share-printers`, `printer-is-shared`    |
| La cua s'omple de treballs que no s'imprimeixen       | `lpstat -o`                                          | `cancel -a nom` i revisar la impressora                      |
| El disc s'omple                                       | `du -sh /var/spool/cups /var/spool/cups-pdf`         | Esborrar els treballs i PDF antics (programar-ho, Tema 3)    |
