## 07. Estructures Iteratives (Bucles)

Els bucles repeteixen un bloc d'ordres: per a cada element d'una llista (`for`), mentre es compleix una condició (`while`) o fins que es compleix (`until`).

### `for` amb una Llista

```bash
for variable in element1 element2 element3; do
    ordres amb "$variable"
done
```

```bash
for servei in cron rsyslog slapd; do
    echo "$servei: $(systemctl is-active "$servei")"
done
```

```
cron: active
rsyslog: active
slapd: active
```

La llista pot venir de moltes fonts:

| Llista                     | Exemple                                              |
| -------------------------- | ---------------------------------------------------- |
| Valors escrits             | `for ip in 192.168.1.1 192.168.1.2; do`              |
| Rang de números            | `for i in {1..5}; do` (de l'1 al 5); `{0..100..10}` (de 10 en 10) |
| Sortida d'una ordre        | `for usuari in $(cut -d: -f1 /etc/passwd); do`       |
| Fitxers                    | `for fitxer in /var/log/*.log; do`                   |
| Paràmetres de l'script     | `for parametre in "$@"; do`                          |
| Vector                     | `for element in "${vector[@]}"; do`                  |

**Per recórrer fitxers, feu servir un patró (`*.log`) i no `$(ls)`**: amb `$(ls)`, els noms que tenen espais es parteixen en trossos.

```bash
for fitxer in /var/log/*.log; do
    echo "$fitxer ocupa $(du -h "$fitxer" | cut -f1)"
done
```

### `for` amb Comptador

```bash
for (( i = 1; i <= 5; i++ )); do
    echo "Intent $i"
done
```

El format és `(( inici; condició; increment ))`, com en altres llenguatges de programació.

### `while`: Mentre es Compleixi una Condició

```bash
comptador=1
while (( comptador <= 3 )); do
    echo "Comptador: $comptador"
    ((comptador++))
done
```

**Llegir un fitxer línia a línia** (l'ús més habitual en administració):

```bash
# usuaris.txt conté una línia per usuari: nom;grup
while IFS=';' read -r nom grup; do
    echo "Usuari: $nom - Grup: $grup"
done < usuaris.txt
```

`IFS=';'` indica que els camps de cada línia estan separats per `;` (fitxer 09).

**Bucle infinit** (per a monitors i menús). S'atura amb `Ctrl+C` o amb `break`:

```bash
while true; do
    echo "$(date +%T) - Càrrega: $(cut -d' ' -f1 /proc/loadavg)"
    sleep 10
done
```

### `until`: Fins que es Compleixi una Condició

`until` és el contrari de `while`: repeteix el bloc **fins que** la condició és certa.

```bash
until ping -c 1 -W 1 192.168.1.2 > /dev/null; do
    echo "Esperant que el servidor 0375 respongui..."
    sleep 5
done
echo "El servidor 0375 ja respon"
```

### `break` i `continue`

| Ordre      | Funció                                             |
| ---------- | -------------------------------------------------- |
| `break`    | Surt del bucle                                     |
| `continue` | Salta directament a la iteració següent            |

```bash
# Mostra només els comptes de persones (UID entre 1000 i 65533)
while IFS=: read -r usuari _ uid _; do
    (( uid < 1000 || uid == 65534 )) && continue
    echo "Usuari: $usuari (UID $uid)"
done < /etc/passwd
```

`_` és un nom de variable que, per convenció, indica un camp que no ens interessa.

### Resolució de Problemes

| Problema                                                    | Causa                                                     | Solució                                              |
| ----------------------------------------------------------- | --------------------------------------------------------- | ---------------------------------------------------- |
| `syntax error near unexpected token 'done'`                 | Falta `do` o el `;` abans de `do`                         | `for x in llista; do`                                |
| El bucle no acaba mai                                       | La variable de la condició no canvia                      | Revisar l'increment; aturar-lo amb `Ctrl+C`          |
| Els noms de fitxer amb espais es parteixen                  | Bucle sobre `$(ls)` o variable sense cometes              | Patrons (`*.log`) i `"$fitxer"`                      |
| Les variables modificades dins d'un `cat fitxer \| while read` tornen a estar buides en sortir | Amb una canonada, el `while` s'executa en un shell fill | `while ... done < fitxer`                |
| `{1..$n}` no funciona                                       | Els rangs no admeten variables                            | `for (( i = 1; i <= n; i++ ))` o `$(seq 1 "$n")`     |
