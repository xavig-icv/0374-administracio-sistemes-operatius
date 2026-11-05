## 03. Gestió d'Impressores, Cues i Treballs

Totes les tasques de la interfície web es poden fer per ordres, i així es poden automatitzar (Temes 3 i 7).

### Consultar l'Estat

| Ordre                 | Funció                                                         |
| --------------------- | -------------------------------------------------------------- |
| `lpstat -t`           | Tot: servidor, impressora per defecte, impressores i treballs  |
| `lpstat -p -d`        | Impressores i quina és la predeterminada                       |
| `lpstat -o`           | Treballs pendents de totes les cues                            |
| `lpstat -v`           | Adreça (dispositiu) de cada impressora                         |
| `lpq -P PDF`          | Cua d'una impressora (ordre de l'estil BSD)                    |

```bash
sudo lpadmin -d PDF                 # PDF com a impressora predeterminada del servidor
lpstat -t
```

```
scheduler is running
system default destination: PDF
device for PDF: cups-pdf:/
PDF accepting requests since Thu 05 Nov 2026 10:12:44 CET
printer PDF is idle.  enabled since Thu 05 Nov 2026 10:12:44 CET
```

### Imprimir

| Ordre                                        | Funció                                           |
| -------------------------------------------- | ------------------------------------------------ |
| `lp -d PDF fitxer.pdf`                       | Imprimir a una impressora concreta               |
| `lp -n 3 fitxer.pdf`                         | 3 còpies a la impressora predeterminada          |
| `lp -o media=A4 -o sides=two-sided-long-edge fitxer.pdf` | Opcions: mida del paper, doble cara  |
| `lp -o page-ranges=1-3 fitxer.pdf`           | Només unes pàgines                               |
| `lpr -P PDF fitxer.pdf`                      | El mateix amb l'ordre de l'estil BSD             |
| `lpoptions -d PDF`                           | Canviar la impressora predeterminada de l'usuari |

```
$ lp -d PDF informe.txt
request id is PDF-12 (1 file(s))
```

`PDF-12` és l'identificador del treball: es fa servir per consultar-lo o cancel·lar-lo.

### Gestionar Treballs

| Ordre                     | Funció                                              |
| ------------------------- | --------------------------------------------------- |
| `cancel PDF-12`           | Cancel·lar un treball                               |
| `cancel -a PDF`           | Cancel·lar tots els treballs d'una impressora       |
| `lp -i PDF-12 -H hold`    | Retenir un treball (no s'imprimeix fins que es deixa anar) |
| `lp -i PDF-12 -H resume`  | Deixar-lo anar                                      |
| `lpmove PDF-12 PDF2`      | Moure un treball a una altra impressora             |

Un usuari normal només pot cancel·lar els seus treballs; els del grup `lpadmin` poden cancel·lar els de tothom.

### Aturar i Reprendre una Impressora

CUPS distingeix entre **acceptar treballs** a la cua i **imprimir-los**:

| Ordre                   | Efecte                                                          | Ús típic                                   |
| ----------------------- | --------------------------------------------------------------- | ------------------------------------------ |
| `cupsdisable PDF`       | La cua accepta treballs però no n'imprimeix cap                 | Canviar el tòner sense perdre els treballs |
| `cupsenable PDF`        | Torna a imprimir els treballs pendents                          |                                            |
| `cupsreject PDF`        | La cua no accepta treballs nous                                 | Retirar una impressora                     |
| `cupsaccept PDF`        | Torna a acceptar-ne                                             |                                            |

### Afegir i Esborrar Impressores: `lpadmin`

**Impressora de xarxa sense controlador (IPP Everywhere):** la majoria d'impressores actuals.

```bash
lpinfo -v                                            # adreces disponibles i impressores detectades
sudo lpadmin -p HP-Aula1 -E -v ipp://192.168.1.60/ipp/print -m everywhere \
     -D "Impressora de l'aula 1" -L "Aula 1"
```

| Opció de `lpadmin`  | Funció                                                         |
| ------------------- | -------------------------------------------------------------- |
| `-p nom`            | Nom de la cua (sense espais)                                   |
| `-E`                | Activa la impressora i fa que accepti treballs                 |
| `-v adreça`         | Dispositiu: `ipp://`, `socket://IP:9100`, `usb://`...          |
| `-m everywhere`     | Sense controlador (IPP Everywhere). Per a impressores antigues: `-m` amb el model de `lpinfo -m` |
| `-D` / `-L`         | Descripció i ubicació                                          |
| `-d nom`            | Impressora predeterminada del servidor                         |
| `-c classe`         | Afegeix la impressora a una classe                             |
| `-x nom`            | Esborra la impressora                                          |

**Classes:** si diverses impressores iguals formen una classe, els treballs enviats a la classe van a la primera que estigui lliure.

```bash
sudo lpadmin -p HP-Aula1 -c aula
sudo lpadmin -p HP-Aula2 -c aula
lp -d aula document.pdf
```

### Resolució de Problemes

| Problema                                              | Causa                                                    | Solució                                              |
| ----------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| `lp: Error - The printer or class does not exist.`    | Nom de la cua incorrecte                                 | `lpstat -p` per veure els noms                       |
| Els treballs es queden a la cua                       | La impressora està desactivada (`disabled`)              | `cupsenable nom`                                     |
| `lp: Destination "PDF" is not accepting jobs.`        | La cua rebutja treballs (`cupsreject`)                   | `cupsaccept nom`                                     |
| `lpadmin: Forbidden`                                  | L'usuari no és administrador de CUPS                     | `sudo` o afegir-lo al grup `lpadmin`                 |
| La impressora s'atura sola després d'un error         | Política d'error per defecte (`stop-printer`)            | Resoldre l'error i `cupsenable`; o canviar-la (fitxer 06) |
