## 05. Rotació de Registres amb `logrotate`

Els fitxers de registre creixen cada dia. Si no es controlen, poden omplir el disc i aturar els serveis. **Rotar** un registre vol dir canviar-li el nom periòdicament (`syslog` → `syslog.1`), començar-ne un de nou, comprimir els antics i esborrar els més vells.

```
syslog          ← fitxer actual (s'hi escriu ara)
syslog.1        ← setmana anterior (sense comprimir)
syslog.2.gz     ← fa dues setmanes (comprimit)
syslog.3.gz
syslog.4.gz     ← el més antic: a la propera rotació s'esborrarà
```

A Ubuntu, `logrotate` s'executa cada dia amb el temporitzador `logrotate.timer` (fitxer 04).

### Fitxers de Configuració

| Fitxer                    | Contingut                                                          |
| ------------------------- | ------------------------------------------------------------------ |
| `/etc/logrotate.conf`     | Opcions generals i `include /etc/logrotate.d`                      |
| `/etc/logrotate.d/`       | Un fitxer per servei o aplicació (`rsyslog`, `apt`, `nginx`...)    |
| `/var/lib/logrotate/status` | Data de l'última rotació de cada fitxer                          |

Exemple de configuració que ja porta Ubuntu (`/etc/logrotate.d/rsyslog`, resumida):

```
/var/log/syslog
/var/log/auth.log
/var/log/kern.log
{
    rotate 4
    weekly
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

### Directives Principals

| Directiva                       | Significat                                                                  |
| ------------------------------- | --------------------------------------------------------------------------- |
| `daily`, `weekly`, `monthly`    | Freqüència de la rotació                                                    |
| `size 10M`                      | Rota quan el fitxer arriba a aquesta mida (sense tenir en compte la data)   |
| `rotate 4`                      | Nombre de fitxers antics que es conserven                                   |
| `compress`                      | Comprimeix els fitxers antics (`.gz`)                                       |
| `delaycompress`                 | No comprimeix el més recent (`.1`), per si algun programa encara hi escriu  |
| `missingok`                     | No dona error si el fitxer no existeix                                      |
| `notifempty`                    | No rota el fitxer si és buit                                                |
| `create 0640 root adm`          | Crea el fitxer nou amb aquests permisos, propietari i grup                  |
| `copytruncate`                  | Copia el fitxer i el buida, per a programes que no poden reobrir el registre |
| `dateext`                       | Afegeix la data al nom (`syslog-20261105`) en lloc d'un número              |
| `postrotate` ... `endscript`    | Ordres que s'executen després de rotar (per exemple, demanar al servei que reobri el fitxer) |
| `sharedscripts`                 | Executa el `postrotate` una sola vegada per a tots els fitxers del bloc     |
| `su usuari grup`                | Usuari i grup amb què es fa la rotació (cal quan el directori no és de root) |

### Configurar la Rotació dels Registres Propis

Els scripts de les pràctiques escriuen registres a `/var/log/informes/`. Per controlar-ne la mida, es crea `/etc/logrotate.d/informes`:

```
/var/log/informes/*.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root adm
}
```

**Provar la configuració sense fer cap canvi** (mode de depuració):

```bash
sudo logrotate -d /etc/logrotate.d/informes
```

**Forçar una rotació** (per comprovar que funciona sense esperar una setmana):

```bash
sudo logrotate -f /etc/logrotate.d/informes
ls -l /var/log/informes/
```

### Registres del Diari de systemd

Els registres de `journalctl` no els gestiona `logrotate`, sinó el mateix `systemd-journald` (Tema 2, fitxer 11). Els límits es configuren a `/etc/systemd/journald.conf` (`SystemMaxUse=500M`) o es fan neteges puntuals amb `sudo journalctl --vacuum-time=2weeks`.

### Resolució de Problemes

| Problema                                                        | Causa                                                       | Solució                                              |
| --------------------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------- |
| `error: skipping ... because parent directory has insecure permissions` | El directori dels registres no és de root o hi pot escriure el grup | Afegir `su root adm` (o l'usuari i el grup adequats) al bloc |
| `logrotate -d` diu `log does not need rotating`                 | Encara no ha passat el període indicat                      | És normal; forçar-ho amb `-f` per provar             |
| Després de rotar, el servei continua escrivint al fitxer antic  | El programa manté obert el fitxer                           | `postrotate` amb la recàrrega del servei, o `copytruncate` |
| Els fitxers antics no es comprimeixen                           | Falta `compress` (o `delaycompress` deixa sense comprimir el `.1`) | Afegir `compress`; el `.1` es comprimirà a la propera rotació |
| El disc s'omple igualment                                       | Massa fitxers conservats o la rotació depèn de la data i el fitxer creix molt de pressa | Reduir `rotate` o afegir `maxsize 100M`      |
