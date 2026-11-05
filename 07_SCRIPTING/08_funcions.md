## 08. Funcions

Una **funció** és un bloc de codi amb nom que es pot cridar tantes vegades com calgui. Serveix per no repetir codi i per dividir un script llarg en parts petites i entenedores.

### Definir i Cridar una Funció

```bash
nom_funcio() {
    ordres
}

nom_funcio                       # crida sense paràmetres: només cal el nom
nom_funcio param1 param2         # crida amb paràmetres
```

També es pot escriure `function nom_funcio { ... }`, però la primera forma és la més habitual i és compatible amb qualsevol shell.

**Una funció s'ha de definir abans de cridar-la.** Per això les funcions es posen al principi de l'script i el programa principal, al final.

### Paràmetres

Dins de la funció, els paràmetres es reben igual que en un script: `$1`, `$2`, `$#`, `"$@"`.

```bash
saludar() {
    echo "Hola, $1! Tens $2 anys."
}

saludar "Anna" 19                # Hola, Anna! Tens 19 anys.
```

### Retornar Valors

Una funció pot retornar dues coses diferents:

| Mecanisme            | Què retorna                                              | Com es recull                     |
| -------------------- | -------------------------------------------------------- | --------------------------------- |
| `echo` (sortida)     | Qualsevol dada: text o números                           | `resultat=$(nom_funcio)`          |
| `return N`           | Només un codi d'estat, de 0 a 255 (0 = correcte)         | `$?` o directament en un `if`     |

En programació es distingeix entre **funció** (retorna un resultat) i **procediment** (fa una tasca i no retorna res). A bash tots dos s'escriuen igual.

```bash
#!/bin/bash
# funcions.sh

# Funció: retorna per la sortida el percentatge d'ús d'un sistema de fitxers
us_disc() {
    df --output=pcent "$1" | tail -n 1 | tr -d ' %'
}

# Funció: retorna un codi (0 si el servei està actiu)
servei_actiu() {
    systemctl is-active --quiet "$1"
}

# Procediment: mostra un missatge amb la data i l'hora
registre() {
    echo "[$(date '+%F %T')] $*"
}

# --- Programa principal ---
percentatge=$(us_disc /)
registre "Disc arrel ocupat al ${percentatge}%"

for servei in cron rsyslog; do
    if servei_actiu "$servei"; then
        registre "$servei: actiu"
    else
        registre "$servei: ATURAT"
    fi
done
```

```
[2026-11-05 10:30:12] Disc arrel ocupat al 47%
[2026-11-05 10:30:12] cron: actiu
[2026-11-05 10:30:12] rsyslog: actiu
```

`servei_actiu` no té `return`: una funció retorna el codi de l'última ordre que executa.

### Variables Locals

Per defecte, les variables d'una funció són visibles a tot l'script. Amb `local` només existeixen dins de la funció i no trepitgen les del programa principal:

```bash
comptador=10
provar() {
    local comptador=0            # variable diferent de la de fora
    ((comptador++))
    echo "Dins: $comptador"      # Dins: 1
}
provar
echo "Fora: $comptador"          # Fora: 10
```

Bona pràctica: declarar amb `local` totes les variables internes de les funcions.

### Biblioteques de Funcions

Les funcions que es fan servir en molts scripts es poden desar en un fitxer i carregar-les amb `source`:

```bash
# ~/scripts/lib/comuns.sh - Funcions compartides
registre() { echo "[$(date '+%F %T')] $*"; }
error()    { echo "ERROR: $*" >&2; exit 1; }
```

```bash
#!/bin/bash
source "$(dirname "$0")/lib/comuns.sh"     # ruta relativa a la ubicació de l'script

registre "Comença la còpia"
[[ -d /var/backups ]] || error "No existeix /var/backups"
```

Amb `"$(dirname "$0")"` la biblioteca es troba encara que l'script s'executi des d'un altre directori o des de `cron` (Tema 3).

### Resolució de Problemes

| Problema                                                       | Causa                                                    | Solució                                                |
| -------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------ |
| `nom_funcio: command not found`                                | Es crida abans de definir-la                             | Definir les funcions al principi de l'script           |
| La funció "retorna" un número però `$?` en mostra un altre     | `return` només admet codis de 0 a 255                    | Retornar les dades amb `echo` i recollir-les amb `$(...)` |
| `$(funcio)` recull també missatges que no s'esperaven          | Tot el que la funció escriu per la sortida forma part del resultat | Enviar els missatges informatius a `>&2`     |
| Una funció canvia una variable del programa principal          | La variable no és `local`                                | Declarar-la amb `local`                                |
| `source: No such file or directory`                            | Ruta relativa al directori de treball                    | `source "$(dirname "$0")/lib/fitxer.sh"`               |
