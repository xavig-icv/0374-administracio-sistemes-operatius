## 01. Introducció als Sistemes d'Impressió

### Per què un Servidor d'Impressió?

En una empresa, les impressores són recursos compartits: molts usuaris imprimeixen a poques impressores. Un **servidor d'impressió** centralitza la gestió: rep els treballs de tots els equips, els posa en una **cua**, els envia a la impressora quan està lliure i permet controlar qui imprimeix, quant i a quina impressora.

| Sense servidor d'impressió                                      | Amb servidor d'impressió                                      |
| --------------------------------------------------------------- | ------------------------------------------------------------- |
| Cada equip ha de tenir el controlador de cada impressora        | Els controladors s'instal·len una vegada, al servidor         |
| Cada equip envia directament a la impressora                    | Una cua central ordena els treballs                           |
| No hi ha control de qui imprimeix ni quant                      | Permisos per usuari o grup, quotes i registre de pàgines      |
| Canviar una impressora obliga a reconfigurar tots els equips    | Els equips continuen imprimint a la mateixa cua del servidor  |

### Conceptes Bàsics

| Concepte                 | Significat                                                                       |
| ------------------------ | -------------------------------------------------------------------------------- |
| **Treball** (*job*)      | Un document enviat a imprimir                                                    |
| **Cua** (*queue*)        | Llista de treballs pendents d'una impressora. A CUPS, cada impressora definida és una cua |
| **Controlador** (*driver*) | Converteix el document al llenguatge de la impressora (PostScript, PCL, PWG Raster...) |
| **Impressió sense controlador** (*driverless*) | La impressora anuncia les seves capacitats i accepta formats estàndard (IPP Everywhere, AirPrint) |
| **Classe**               | Grup d'impressores: el treball va a la primera que estigui lliure                |

### Protocols d'Impressió

| Protocol                    | Port     | Característiques                                                         |
| --------------------------- | -------- | ------------------------------------------------------------------------ |
| **IPP** (*Internet Printing Protocol*) | 631 | Protocol estàndard actual, sobre HTTP. Permet consultar l'estat i les capacitats |
| IPPS                        | 631      | IPP xifrat amb TLS                                                       |
| LPD / LPR                   | 515      | Protocol antic dels sistemes UNIX                                        |
| RAW / JetDirect             | 9100     | Envia les dades directament a la impressora, sense control               |
| SMB                         | 445      | Impressores compartides des de Windows o Samba                           |

### CUPS

**CUPS** (*Common UNIX Printing System*) és el sistema d'impressió de Linux i macOS. Fa servir IPP i té una interfície web d'administració.

```
Aplicació ──► cupsd (cua) ──► filtres (conversió al format de la impressora) ──► backend (ipp, usb, socket...) ──► Impressora
```

| Component         | Funció                                                                          |
| ----------------- | ------------------------------------------------------------------------------- |
| `cupsd`           | El servei: rep els treballs, gestiona les cues i la interfície web (port 631)   |
| Filtres           | Converteixen el document (PDF, text, imatge) al format que entén la impressora  |
| *Backends*        | Envien el resultat a la impressora: `ipp://`, `socket://` (port 9100), `usb://`, `smb://`... |
| PPD               | Fitxer que descriu una impressora concreta (en desús amb la impressió sense controlador) |

### La Impressió a l'Escenari del Laboratori

Al laboratori no hi ha impressores físiques connectades a les màquines virtuals. Es fa servir una **impressora virtual PDF** (`cups-pdf`): els treballs es converteixen en fitxers PDF al servidor. Tot el que es fa amb ella (cues, permisos, quotes, compartir-la amb Windows i Linux) és igual que amb una impressora real.

```
Ordinador Windows (xarxa del centre) ──IPP──┐
                                             ├──► srv0374: CUPS + impressora PDF ──► fitxers .pdf
Client Ubuntu (192.168.1.50)        ──IPP──┘
```

### Continguts del Tema

| Fitxer                                   | Contingut                                                    |
| ---------------------------------------- | ------------------------------------------------------------ |
| [02](./02_install_cups.md)               | Instal·lació de CUPS i de la impressora PDF                  |
| [03](./03_gestio_impressores.md)         | Gestió d'impressores, cues i treballs per ordres             |
| [04](./04_compartir_impressores.md)      | Compartir impressores amb clients Linux i Windows            |
| [05](./05_control_acces_quotes.md)       | Control d'accés i quotes d'impressió                         |
| [06](./06_registres_problemes.md)        | Registres i resolució de problemes                           |
