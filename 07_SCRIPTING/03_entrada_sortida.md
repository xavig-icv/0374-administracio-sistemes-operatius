## 03. Entrada i Sortida de Dades

Un script pot rebre dades de tres maneres: com a **paràmetres** (fitxer 02), demanant-les a l'usuari mentre s'executa (`read`) o llegint-les d'un fitxer. I pot mostrar els resultats per pantalla o desar-los en fitxers.

### Llegir Dades: `read`

```bash
#!/bin/bash
# entrada.sh - Demana dades a l'usuari
read -rp "Nom del domini: " domini
echo "Has escrit: $domini"
```

```
Nom del domini: educem.cat
Has escrit: educem.cat
```

| Opció        | Funció                                                        |
| ------------ | ------------------------------------------------------------- |
| `-p "text"`  | Mostra un missatge abans de llegir                            |
| `-r`         | No interpreta les barres inverses `\` (**recomanat sempre**)  |
| `-s`         | No mostra el que s'escriu (contrasenyes)                      |
| `-t 10`      | Temps màxim d'espera en segons                                |
| `-n 1`       | Llegeix un sol caràcter, sense esperar l'Enter                |
| `-a vector`  | Desa cada paraula en un element d'un vector                   |

Si no s'indica cap variable, el valor queda a `$REPLY`.

```bash
read -rsp "Contrasenya: " contrasenya; echo     # echo: salt de línia després de la contrasenya
read -rp "Usuari i grup: " usuari grup          # cada paraula va a una variable
```

**Llegir dades d'un fitxer:**

```bash
read -r nom_servidor < /etc/hostname            # primera línia del fitxer
contingut=$(cat /etc/hostname)                  # tot el contingut

while read -r linia; do                         # recórrer un fitxer línia a línia (fitxer 07)
    echo "Línia: $linia"
done < usuaris.txt
```

### Mostrar Dades: `echo` i `printf`

```bash
echo "Text normal"
echo -e "Línia 1\nLínia 2\tamb tabulador"    # -e interpreta \n (salt de línia) i \t (tabulador)
echo -n "Sense salt de línia final"
printf "%-10s %6s\n" "Usuari" "UID"
printf "%-10s %6d\n" "xavi" 1000
```

```
Usuari        UID
xavi         1000
```

| Format de `printf` | Significat                                    |
| ------------------ | --------------------------------------------- |
| `%s`               | Text                                          |
| `%d`               | Nombre enter                                  |
| `%.2f`             | Nombre decimal amb 2 xifres                   |
| `%-10s`            | Text alineat a l'esquerra en 10 caràcters     |
| `%6d`              | Enter alineat a la dreta en 6 caràcters       |
| `\n`, `\t`         | Salt de línia, tabulador                      |

`printf` és més precís que `echo` per fer informes amb columnes ben alineades.

### Colors i Estils al Terminal

**Amb seqüències d'escapament ANSI** i `echo -e`:

```bash
echo -e "\e[32mOK\e[0m - Servei actiu"
echo -e "\e[31mERROR\e[0m - El servei no respon"
echo -e "\e[1;33mAvís en negreta i groc\e[0m"
```

| Codi       | Color   | Codi        | Estil                         |
| ---------- | ------- | ----------- | ----------------------------- |
| `\e[31m`   | Vermell | `\e[1m`     | Negreta                       |
| `\e[32m`   | Verd    | `\e[3m`     | Cursiva                       |
| `\e[33m`   | Groc    | `\e[4m`     | Subratllat                    |
| `\e[34m`   | Blau    | `\e[7m`     | Colors invertits              |
| `\e[35m`   | Lila    | `\e[41m`... | Fons de color (del 40 al 47)  |
| `\e[36m`   | Cian    | `\e[0m`     | **Tornar al format normal**   |

Els codis es combinen separant-los amb `;` (`\e[1;31m`: negreta i vermell). Cal acabar sempre amb `\e[0m`: si no, la resta del terminal es queda amb aquell format.

**Amb `tput`**, que funciona amb qualsevol tipus de terminal:

```bash
vermell=$(tput setaf 1); verd=$(tput setaf 2); negreta=$(tput bold); normal=$(tput sgr0)

echo "${verd}OK${normal} - Servei actiu"
echo "${negreta}${vermell}ERROR${normal} - Servei aturat"
```

| `tput`                       | Efecte                                                            |
| ---------------------------- | ----------------------------------------------------------------- |
| `setaf N` / `setab N`        | Color del text / del fons (0 negre, 1 vermell, 2 verd, 3 groc, 4 blau, 5 lila, 6 cian, 7 blanc) |
| `bold`, `smul`, `rev`        | Negreta, subratllat, colors invertits                             |
| `sgr0`                       | Torna al format normal                                            |

Desar els colors en variables al principi de l'script fa que el codi sigui molt més llegible.

### Redireccions

Tot programa té tres canals: l'entrada estàndard (`0`, el teclat), la sortida estàndard (`1`, la pantalla) i la sortida d'errors (`2`, també la pantalla).

| Operador          | Funció                                                                 |
| ----------------- | ---------------------------------------------------------------------- |
| `> fitxer`        | Envia la sortida a un fitxer (el sobreescriu)                          |
| `>> fitxer`       | Afegeix la sortida al final del fitxer                                 |
| `2> fitxer`       | Envia els errors a un fitxer                                           |
| `2> /dev/null`    | Descarta els errors                                                    |
| `> fitxer 2>&1`   | Sortida i errors al mateix fitxer (`2>&1`: els errors van on va la sortida) |
| `&> fitxer`       | El mateix, en forma abreujada                                          |
| `< fitxer`        | Llegeix l'entrada d'un fitxer                                          |

```bash
find / -name "*.sh" 2> /dev/null                    # només els resultats, sense "Permission denied"
find / -name "*.sh" > trobats.txt 2> errors.txt     # resultats i errors en fitxers separats
find / -name "*.sh" > tot.txt 2>&1                  # tot en un sol fitxer
```

**L'ordre de les redireccions importa:** `> fitxer 2>&1` envia-ho tot al fitxer, però `2>&1 > fitxer` deixa els errors a la pantalla, perquè els redirigeix on anava la sortida **abans** de canviar-la.

**Escriure diverses línies en un fitxer** (*here-document*):

```bash
cat > /tmp/informe.txt << EOF
Servidor: $(hostname)
Data: $(date '+%d/%m/%Y')
Usuari: $USER
EOF
```

Amb `<< 'EOF'` (entre cometes) el text es copia literalment, sense substituir les variables. És molt útil per crear fitxers de configuració des d'un script.

Per mostrar la sortida per pantalla i desar-la alhora en un fitxer: `ordre | tee -a registre.log` (Tema 2, fitxer 02).

### Resolució de Problemes

| Problema                                           | Causa                                           | Solució                                         |
| -------------------------------------------------- | ----------------------------------------------- | ----------------------------------------------- |
| `echo` mostra `\e[31m` literalment                 | Falta `-e` o l'script s'executa amb `sh`        | `echo -e` amb bash, o fer servir `printf`/`tput` |
| Tot el terminal es queda de color                  | Falta `\e[0m` o `tput sgr0` al final            | Tancar sempre el format                         |
| El fitxer de sortida només té l'última línia       | `>` sobreescriu el fitxer cada vegada           | `>>` per afegir-hi línies                       |
| Els errors continuen sortint per pantalla          | Només s'ha redirigit la sortida estàndard       | Afegir `2>` o `2>&1`                            |
| `read` elimina les barres inverses del text        | Falta l'opció `-r`                              | `read -r`                                       |
