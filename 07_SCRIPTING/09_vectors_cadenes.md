## 09. Vectors i Cadenes de Text

### Vectors (*arrays*)

Un **vector** guarda una llista d'elements identificats per un índex, que comença per 0.

```bash
serveis=("ssh" "cron" "slapd")       # declaració
serveis+=("rsyslog")                 # afegeix un element al final
echo "${serveis[0]}"                 # primer element: ssh
echo "${serveis[@]}"                 # tots els elements
echo "${#serveis[@]}"                # nombre d'elements: 4
echo "${!serveis[@]}"                # índexs: 0 1 2 3
serveis[1]="cron.service"            # modifica un element
unset 'serveis[2]'                   # esborra un element
```

| Expressió             | Resultat                                   |
| --------------------- | ------------------------------------------ |
| `${vector[i]}`        | Element de la posició `i`                  |
| `"${vector[@]}"`      | Tots els elements (cadascun per separat)   |
| `${#vector[@]}`       | Nombre d'elements                          |
| `${!vector[@]}`       | Índexs                                     |
| `${vector[@]:1:2}`    | 2 elements a partir de la posició 1        |
| `vector+=(x)`         | Afegeix un element                         |

**Recórrer un vector:**

```bash
for servei in "${serveis[@]}"; do
    echo "Comprovant $servei"
done

for i in "${!serveis[@]}"; do
    echo "$i: ${serveis[$i]}"
done
```

**Omplir un vector amb la sortida d'una ordre** (una línia per element):

```bash
mapfile -t usuaris < <(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd)
echo "Hi ha ${#usuaris[@]} usuaris: ${usuaris[*]}"
```

**Vectors associatius:** l'índex és un text, com en un diccionari.

```bash
declare -A ports=( [ssh]=22 [http]=80 [ldap]=389 )
echo "LDAP fa servir el port ${ports[ldap]}"

for servei in "${!ports[@]}"; do
    echo "$servei → ${ports[$servei]}"
done
```

En un vector associatiu, l'ordre en què es recorren els elements no està garantit.

### Cadenes de Text

Exemples amb `text="administracio_sistemes.log"`:

| Operació                               | Sintaxi                    | Exemple            | Resultat                         |
| -------------------------------------- | -------------------------- | ------------------ | -------------------------------- |
| Longitud                               | `${#text}`                 | `${#text}`         | `26`                             |
| Subcadena                              | `${text:inici:longitud}`   | `${text:0:13}`     | `administracio`                  |
| Des d'una posició fins al final        | `${text:inici}`            | `${text:14}`       | `sistemes.log`                   |
| Treure el principi (coincidència més curta) | `${text#patró}`       | `${text#*_}`       | `sistemes.log`                   |
| Treure el principi (coincidència més llarga) | `${text##patró}`     | `${text##*.}`      | `log`                            |
| Treure el final (coincidència més curta)   | `${text%patró}`        | `${text%.*}`       | `administracio_sistemes`         |
| Treure el final (coincidència més llarga)  | `${text%%patró}`       | `${text%%_*}`      | `administracio`                  |
| Substituir la primera coincidència     | `${text/patró/nou}`        | `${text/log/txt}`  | `administracio_sistemes.txt`     |
| Substituir totes les coincidències     | `${text//patró/nou}`       | `${text//_/-}`     | `administracio-sistemes.log`     |
| Majúscules / minúscules                | `${text^^}` / `${text,,}`  | `${text^^}`        | `ADMINISTRACIO_SISTEMES.LOG`     |
| Concatenar                             | `"$a$b"` o `"${a}_${b}"`   | `"${text%.*}_2026.log"` | `administracio_sistemes_2026.log` |

Usos habituals en administració:

```bash
fitxer="/var/backups/copia_2026-11-05.tar.gz"
echo "${fitxer##*/}"          # copia_2026-11-05.tar.gz  (nom sense el directori, com basename)
echo "${fitxer%/*}"           # /var/backups             (el directori, com dirname)
```

### Dividir una Cadena: `IFS`

`IFS` (*Internal Field Separator*) és el caràcter que bash fa servir per separar paraules. Canviant-lo es pot dividir una línia en parts:

```bash
linia="anna;Anna Puig;alumnes"
IFS=';' read -r usuari nom grup <<< "$linia"
echo "$usuari - $nom - $grup"                # anna - Anna Puig - alumnes

IFS='.' read -ra octets <<< "192.168.1.50"
echo "Últim octet: ${octets[3]}"             # Últim octet: 50
```

`<<<` (*here-string*) passa el contingut d'una variable com a entrada d'una ordre. Posant `IFS=...` just davant de `read`, el canvi només afecta aquella ordre.

### Resolució de Problemes

| Problema                                                      | Causa                                              | Solució                                         |
| ------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------- |
| `$vector` només mostra el primer element                      | `$vector` equival a `${vector[0]}`                 | `"${vector[@]}"`                                |
| Els elements amb espais es parteixen en recórrer el vector    | Falten les cometes                                 | `"${vector[@]}"`                                |
| `declare: -A: invalid option`                                 | Bash molt antic o l'script s'executa amb `sh`      | Executar-lo amb bash 4 o posterior              |
| Després de canviar `IFS`, la resta de l'script es comporta de manera estranya | S'ha canviat `IFS` per a tot l'script | Posar-lo només davant de l'ordre: `IFS=';' read ...` |
