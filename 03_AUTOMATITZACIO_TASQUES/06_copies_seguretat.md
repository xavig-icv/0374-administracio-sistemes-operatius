## 06. Còpies de Seguretat Automàtiques

Les còpies de seguretat són la tasca automatitzada més important d'un servidor: si es perden les dades (error humà, disc avariat, programari maliciós), només es poden recuperar si hi ha una còpia recent **i** se sap restaurar.

### Conceptes

| Tipus de còpia   | Què copia                                         | Avantatges i inconvenients                              |
| ---------------- | ------------------------------------------------- | ------------------------------------------------------- |
| **Completa**     | Totes les dades                                   | Restauració simple; ocupa molt i triga                  |
| **Incremental**  | Només el que ha canviat des de l'última còpia (de qualsevol tipus) | Ràpida i petita; per restaurar calen la completa i totes les incrementals |
| **Diferencial**  | El que ha canviat des de l'última còpia completa  | Entremig: per restaurar calen la completa i l'última diferencial |

**Regla 3-2-1:** tenir **3** còpies de les dades, en **2** suports diferents, i **1** fora del lloc on hi ha el servidor. Al laboratori, com a mínim, la còpia ha d'anar a l'altre servidor (`192.168.1.2`): si el servidor 0374 s'espatlla, la còpia continua disponible.

**Una còpia que no s'ha provat de restaurar no és una còpia fiable.** Cal comprovar periòdicament que es pot recuperar.

### `tar`: Fitxers Comprimits

| Ordre                                                 | Funció                                                  |
| ----------------------------------------------------- | ------------------------------------------------------- |
| `tar czf copia.tar.gz -C / etc`                       | Crea (`c`) un fitxer comprimit amb gzip (`z`) de `/etc` |
| `tar tzf copia.tar.gz`                                | Llista (`t`) el contingut                               |
| `tar xzf copia.tar.gz -C /tmp/restaura`               | Extreu (`x`) el contingut en un directori               |
| `tar xzf copia.tar.gz -C /tmp/restaura etc/hosts`     | Extreu només un fitxer                                  |
| `tar czf web.tar.gz --exclude='*.log' -C / var/www`   | Exclou fitxers                                          |

`-C /` fa que dins de la còpia les rutes siguin relatives (`etc/hosts` en lloc de `/etc/hosts`): en restaurar, no se sobreescriuen els fitxers del sistema per error. `j` (bzip2) i `J` (xz) comprimeixen més que `z`, però són més lents.

**Còpies incrementals amb `tar`:**

```bash
# La primera vegada fa una còpia completa i crea el fitxer d'estat .snar
sudo tar czf /var/backups/home_0.tar.gz --listed-incremental=/var/backups/home.snar -C / home
# Les següents només copien el que ha canviat
sudo tar czf /var/backups/home_1.tar.gz --listed-incremental=/var/backups/home.snar -C / home
```

### `rsync`: Sincronitzar Directoris

`rsync` només copia les diferències entre l'origen i el destí: després de la primera còpia, les següents són molt ràpides. Funciona en local i entre equips per SSH.

```bash
rsync -av /var/www/ /var/backups/www/                       # còpia local
rsync -avz /var/www/ copies@192.168.1.2:/srv/copies/www/    # còpia a un altre servidor per SSH
rsync -avz --delete --dry-run /var/www/ copies@192.168.1.2:/srv/copies/www/   # simulació
```

| Opció         | Funció                                                                         |
| ------------- | ------------------------------------------------------------------------------ |
| `-a`          | Mode arxiu: recursiu i conserva permisos, propietaris i dates                  |
| `-v`          | Mostra els fitxers que copia                                                   |
| `-z`          | Comprimeix les dades durant la transferència                                   |
| `--delete`    | Esborra al destí el que ja no existeix a l'origen (còpia exacta, *mirall*)     |
| `-n`, `--dry-run` | Simula: mostra què faria sense fer res. **Proveu-ho sempre abans de `--delete`** |
| `--exclude`   | Exclou fitxers o directoris                                                    |
| `-e "ssh -i clau"` | Fa servir una clau SSH concreta                                           |

**La barra final de l'origen importa:** `rsync -a /var/www/ desti/` copia el **contingut** de `www` dins de `desti`; `rsync -a /var/www desti/` crea `desti/www`.

### Còpies Desateses per SSH

Una còpia programada no pot demanar contrasenya. Cal autenticació per **claus SSH** sense frase de pas, amb un usuari dedicat a les còpies al servidor de destinació:

```bash
# Al servidor de destinació (192.168.1.2): usuari i directori per a les còpies
sudo useradd -m -s /bin/bash copies
sudo passwd copies                      # contrasenya temporal, només per copiar-hi la clau
sudo mkdir -p /srv/copies && sudo chown copies: /srv/copies

# Al servidor d'origen (srv0374), com a root (l'usuari que executarà la còpia)
sudo ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_copies -C "copies@srv0374"
sudo ssh-copy-id -i /root/.ssh/id_copies.pub copies@192.168.1.2
sudo ssh -i /root/.ssh/id_copies copies@192.168.1.2 "echo Connexió correcta"
```

`ssh-copy-id` demana la contrasenya de l'usuari `copies` una sola vegada. Si l'accés per contrasenya està desactivat al destí, cal copiar la clau pública manualment al fitxer `~/.ssh/authorized_keys` de l'usuari `copies`.

Una clau sense frase de pas dona accés directe al servidor de destinació. Per limitar-ne el risc, al fitxer `authorized_keys` del destí es pot restringir des de quina IP es pot fer servir i desactivar-ne el reenviament de ports i la terminal:

```
from="192.168.1.1",restrict ssh-ed25519 AAAAC3Nza... copies@srv0374
```

### Script de Còpia

```bash
#!/bin/bash
# /usr/local/bin/copia-srv0374.sh - Còpia de /etc i /home al servidor 0375
set -euo pipefail

desti="copies@192.168.1.2:/srv/copies/srv0374"
clau="/root/.ssh/id_copies"
local_dir="/var/backups/srv0374"
data=$(date +%F)

mkdir -p "$local_dir"
tar czf "$local_dir/etc_$data.tar.gz" -C / etc
tar czf "$local_dir/home_$data.tar.gz" -C / home
find "$local_dir" -name "*.tar.gz" -mtime +7 -delete       # conserva 7 dies en local

if rsync -az --delete -e "ssh -i $clau" "$local_dir/" "$desti/"; then
    logger -t copia-srv0374 "Còpia del $data enviada a 192.168.1.2"
else
    logger -t copia-srv0374 -p user.err "ERROR enviant la còpia del $data"
    exit 1
fi
```

Es programa cada nit amb un temporitzador (fitxer 04) o amb `cron` (fitxer 03), i el resultat es consulta amb `journalctl -t copia-srv0374`.

**Comprovar la restauració:**

```bash
mkdir -p /tmp/restaura
tar xzf /var/backups/srv0374/etc_2026-11-05.tar.gz -C /tmp/restaura etc/hosts
diff /etc/hosts /tmp/restaura/etc/hosts && echo "Restauració correcta"
```

### Resolució de Problemes

| Problema                                                     | Causa                                                          | Solució                                                       |
| ------------------------------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------------------- |
| La còpia programada es queda esperant una contrasenya        | La clau SSH té frase de pas o no s'ha copiat al destí          | Clau sense frase de pas i `ssh-copy-id`                       |
| `Host key verification failed` en la còpia programada        | El servidor de destinació no és a `known_hosts` de root        | Connectar-s'hi una vegada a mà com a root per acceptar la clau |
| `rsync: command not found` al destí                          | `rsync` no està instal·lat al servidor de destinació           | `sudo apt install rsync` als dos servidors                    |
| Amb `--delete` s'han esborrat fitxers del destí que no s'esperava | La barra final de l'origen o el destí eren incorrectes     | Provar sempre abans amb `--dry-run`                           |
| `tar: Removing leading '/' from member names`                | S'ha fet servir una ruta absoluta sense `-C`                   | És un avís; per evitar-lo, `-C / etc`                         |
| Les còpies omplen el disc                                    | No hi ha rotació de còpies antigues                            | `find ... -mtime +N -delete` o menys dies de conservació      |
