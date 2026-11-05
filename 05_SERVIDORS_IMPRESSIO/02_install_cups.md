## 02. Instal·lació de CUPS i de la Impressora PDF

### Instal·lació

```bash
sudo apt install cups printer-driver-cups-pdf
systemctl status cups
sudo ss -tlnp | grep 631            # per defecte, només escolta a localhost
lpstat -p                           # el paquet cups-pdf crea la impressora PDF
```

```
printer PDF is idle.  enabled since Thu 05 Nov 2026 10:12:44 CET
```

| Paquet                      | Funció                                                         |
| --------------------------- | -------------------------------------------------------------- |
| `cups`                      | Servidor d'impressió                                           |
| `printer-driver-cups-pdf`   | Impressora virtual que crea fitxers PDF                        |
| `cups-browsed`              | Descobreix automàticament les impressores de la xarxa (vegeu la nota de seguretat) |

### Administradors de CUPS

Per administrar CUPS (crear impressores, cancel·lar treballs d'altres usuaris), l'usuari ha de pertànyer al grup **`lpadmin`**:

```bash
sudo usermod -aG lpadmin xavi        # cal tornar a iniciar la sessió perquè tingui efecte
```

### Interfície Web i Accés per Xarxa

Per defecte, CUPS només accepta connexions del mateix servidor. Per administrar-lo i compartir les impressores per la xarxa:

```bash
sudo cupsctl --remote-admin --remote-any --share-printers
sudo ss -tlnp | grep 631            # ara escolta a totes les adreces
sudo ufw allow 631/tcp              # si el tallafoc està actiu
```

| Opció de `cupsctl`    | Efecte                                                      |
| --------------------- | ----------------------------------------------------------- |
| `--remote-admin`      | Permet administrar CUPS des d'altres equips                 |
| `--remote-any`        | Permet l'accés des de qualsevol xarxa (no només la local)   |
| `--share-printers`    | Comparteix les impressores marcades com a compartides       |
| `--debug-logging`     | Registre detallat (fitxer 06)                               |

`cupsctl` modifica el fitxer de configuració `/etc/cups/cupsd.conf`. Executat sense opcions, en mostra la configuració actual.

**Interfície web:** `https://192.168.1.1:631` des del client (o `https://10.0.1.37:631` des de l'ordinador de l'aula). El certificat és autosignat. Les pàgines d'administració demanen un usuari del grup `lpadmin`.

| Pestanya          | Funció                                                        |
| ----------------- | ------------------------------------------------------------- |
| *Administration*  | Afegir impressores, configuració del servidor                 |
| *Printers*        | Cues, estat i treballs de cada impressora                     |
| *Jobs*            | Tots els treballs                                             |

### La Impressora PDF

Els treballs enviats a la impressora `PDF` es desen com a fitxers:

| Qui imprimeix                      | On es desa el PDF                            |
| ---------------------------------- | -------------------------------------------- |
| Un usuari del servidor             | `~/PDF/` de l'usuari                         |
| Un client de la xarxa (anònim)     | `/var/spool/cups-pdf/ANONYMOUS/`             |

```bash
echo "Prova d'impressió" | lp -d PDF
ls ~/PDF/
```

### Seguretat: `cups-browsed`

`cups-browsed` escolta la xarxa per afegir automàticament les impressores que s'hi anuncien. El setembre de 2024 s'hi van publicar vulnerabilitats greus que permetien a un atacant afegir impressores falses. En un servidor no cal: comproveu-ne l'estat i, si no el necessiteu, desactiveu-lo.

```bash
systemctl status cups-browsed
sudo systemctl disable --now cups-browsed
```

### Resolució de Problemes

| Problema                                                    | Causa                                                   | Solució                                                  |
| ----------------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| La interfície web no respon des d'un altre equip            | CUPS només escolta a localhost                          | `sudo cupsctl --remote-admin --remote-any`               |
| `Forbidden` en entrar a *Administration*                    | L'usuari no és al grup `lpadmin`                        | `sudo usermod -aG lpadmin usuari` i tornar a iniciar sessió |
| No apareix la impressora `PDF`                              | Falta el paquet `printer-driver-cups-pdf`               | Instal·lar-lo i comprovar `lpstat -p`                    |
| No es troba el PDF imprès                                   | S'ha imprès des d'un altre equip (usuari anònim)        | Mirar a `/var/spool/cups-pdf/ANONYMOUS/`                 |
