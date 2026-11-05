## 05. Estructures Condicionals

### `if`, `elif` i `else`

```bash
if condició; then
    ordres si la condició és certa
elif altra_condició; then
    ordres si la segona condició és certa
else
    ordres si no es compleix cap condició
fi
```

A bash, **una condició és una ordre**: si acaba correctament (codi de sortida 0) es considera **certa**; si acaba amb error (codi diferent de 0), **falsa**. Per això a l'`if` s'hi pot posar qualsevol ordre:

```bash
if ping -c 1 -W 2 192.168.1.2 > /dev/null; then
    echo "El servidor 0375 respon"
else
    echo "El servidor 0375 no respon"
fi

if grep -q "^asixprofe:" /etc/passwd; then      # -q: no mostra res, només retorna el codi
    echo "L'usuari asixprofe existeix"
fi
```

Per fer **comparacions** es fa servir l'ordre `test`, que normalment s'escriu entre claudàtors: `[ condició ]`. Bash té també la forma ampliada `[[ condició ]]`, més segura i amb més possibilitats. **Els espais dins dels claudàtors són obligatoris.**

| Forma                  | Quan es fa servir                                                             |
| ---------------------- | ----------------------------------------------------------------------------- |
| `if ordre; then`       | Comprovar si una ordre ha funcionat                                           |
| `if [ ... ]; then`     | Comparacions compatibles amb qualsevol shell (POSIX)                          |
| `if [[ ... ]]; then`   | Comparacions a bash: patrons, `&&`, `\|\|`, expressions regulars. **Recomanada** |
| `if (( ... )); then`   | Comparacions numèriques amb `<`, `>`, `==`                                    |

### Comparar Números

| Operador | Significat        | Exemple                  |
| -------- | ----------------- | ------------------------ |
| `-eq`    | Igual             | `[[ "$a" -eq "$b" ]]`    |
| `-ne`    | Diferent          | `[[ "$a" -ne "$b" ]]`    |
| `-gt`    | Més gran          | `[[ "$a" -gt "$b" ]]`    |
| `-ge`    | Més gran o igual  | `[[ "$a" -ge "$b" ]]`    |
| `-lt`    | Més petit         | `[[ "$a" -lt "$b" ]]`    |
| `-le`    | Més petit o igual | `[[ "$a" -le "$b" ]]`    |

Amb `(( ))` s'escriuen com en matemàtiques: `(( a > b ))`, `(( a >= b ))`, `(( a == b ))`.

### Comparar Cadenes

| Operador              | Significat                                          | Exemple                          |
| --------------------- | --------------------------------------------------- | -------------------------------- |
| `==` (o `=`)          | Iguals                                              | `[[ "$usuari" == "root" ]]`      |
| `!=`                  | Diferents                                           | `[[ "$resposta" != "s" ]]`       |
| `-z`                  | La cadena és buida                                  | `[[ -z "$1" ]]`                  |
| `-n`                  | La cadena no és buida                               | `[[ -n "$nom" ]]`                |
| `==` amb un patró     | Coincideix amb un patró (només amb `[[ ]]`)         | `[[ "$fitxer" == *.log ]]`       |
| `=~`                  | Coincideix amb una expressió regular (només `[[ ]]`) | `[[ "$ip" =~ ^192\.168\. ]]`    |

El patró de la dreta (`*.log`) s'escriu **sense cometes**: entre cometes es compararia amb el text literal `*.log`.

**Poseu les variables entre cometes.** Amb `[ -n $nom ]` i la variable buida, bash avalua `[ -n ]`, que és **cert**: l'script creu que la variable té valor. Amb `[ -n "$nom" ]` funciona correctament. Dins de `[[ ]]` aquest problema no es produeix, però les cometes no fan cap mal.

### Comprovar Fitxers i Directoris

| Operador                | Cert si...                                      | Exemple                         |
| ----------------------- | ----------------------------------------------- | ------------------------------- |
| `-e`                    | Existeix                                        | `[[ -e "$ruta" ]]`              |
| `-f`                    | És un fitxer normal                             | `[[ -f /etc/passwd ]]`          |
| `-d`                    | És un directori                                 | `[[ -d "$HOME/scripts" ]]`      |
| `-s`                    | Existeix i no és buit                           | `[[ -s errors.log ]]`           |
| `-r`, `-w`, `-x`        | Hi tenim permís de lectura, escriptura o execució | `[[ -r /etc/shadow ]]`        |
| `-L`                    | És un enllaç simbòlic                           | `[[ -L /sbin/init ]]`           |
| `fitxer1 -nt fitxer2`   | `fitxer1` és més nou que `fitxer2`              | `[[ copia.tar -nt dades.txt ]]` |

```bash
#!/bin/bash
# comprovar.sh
if [[ -d "$HOME/scripts" ]]; then
    echo "El directori scripts existeix"
fi

if [[ -f /etc/passwd ]]; then
    echo "/etc/passwd és un fitxer"
fi

if [[ -r /etc/shadow ]]; then
    echo "Pots llegir /etc/shadow"
else
    echo "No tens permís per llegir /etc/shadow (prova-ho amb sudo)"
fi
```

### Combinar Condicions

| Operador | Significat                    | Exemple                                    |
| -------- | ----------------------------- | ------------------------------------------ |
| `&&`     | I: totes dues han de ser certes | `[[ "$a" -gt 0 && "$a" -lt 10 ]]`        |
| `\|\|`   | O: n'hi ha prou que una sigui certa | `[[ "$r" == "s" \|\| "$r" == "S" ]]` |
| `!`      | No: inverteix la condició     | `[[ ! -d "$directori" ]]`                  |

Dins de `[ ]` no es poden fer servir `&&` ni `||`: cal escriure dues condicions, `[ ... ] && [ ... ]`.

**Forma curta en una línia:**

```bash
[[ -d /var/backups ]] || mkdir -p /var/backups       # si no existeix, el crea
systemctl is-active --quiet cron && echo "cron està actiu"
```

### Exemple: Comprovar que l'Script s'Executa com a root

```bash
if [[ $EUID -ne 0 ]]; then
    echo "Aquest script s'ha d'executar amb sudo" >&2
    exit 1
fi
```

`$EUID` és l'identificador de l'usuari efectiu (0 = root). `>&2` envia el missatge a la sortida d'errors i `exit 1` acaba l'script amb un codi d'error (fitxer 11).

### Resolució de Problemes

| Problema                                                    | Causa                                                    | Solució                                       |
| ----------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------- |
| `[: missing ']'`                                            | Falta l'espai abans de `]`                               | `[ "$a" -eq 1 ]`                              |
| `unary operator expected` o `too many arguments`            | Variable buida o amb espais i sense cometes              | `"$variable"` o `[[ ]]`                       |
| `[[: not found`                                             | L'script s'executa amb `sh`                              | Executar-lo amb bash                          |
| `integer expression expected`                               | S'ha fet servir `-eq` o `-gt` amb un text                | Números amb `-eq`, textos amb `==`            |
| El patró `*.log` no coincideix                              | Està entre cometes o dins de `[ ]`                       | `[[ "$f" == *.log ]]`, sense cometes al patró |
| `[ $a > $b ]` crea un fitxer amb el nom del valor de `$b`   | Dins de `[ ]`, `>` és una redirecció                     | `-gt` o `(( a > b ))`                         |
