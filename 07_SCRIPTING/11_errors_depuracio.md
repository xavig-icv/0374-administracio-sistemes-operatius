## 11. Codis de Sortida, Errors i Depuració

### Codis de Sortida

Totes les ordres acaben amb un **codi de sortida** (*exit status*): `0` vol dir que han funcionat i qualsevol altre valor (de 1 a 255), que hi ha hagut un error. El codi de l'última ordre es troba a `$?`:

```bash
ls /etc/hostname
echo $?                         # 0
ls /no/existeix
echo $?                         # 2
```

| Codi  | Significat habitual                       |
| ----- | ----------------------------------------- |
| 0     | Correcte                                  |
| 1     | Error general                             |
| 2     | Ús incorrecte (per conveni)               |
| 126   | L'ordre no es pot executar (permisos)     |
| 127   | Ordre no trobada                          |
| 130   | Interromput amb `Ctrl+C`                  |

### Acabar un Script: `exit`

`exit N` acaba l'script amb el codi `N`. Els vostres scripts també han de retornar `0` si tot ha anat bé i un altre valor si hi ha hagut un error: així altres scripts, `cron` o systemd poden saber si han funcionat.

```bash
#!/bin/bash
# alta_usuari.sh - Ús: sudo ./alta_usuari.sh nom_usuari
if [[ $# -ne 1 ]]; then
    echo "Ús: $0 nom_usuari" >&2
    exit 2
fi
if id "$1" &> /dev/null; then
    echo "L'usuari $1 ja existeix" >&2
    exit 1
fi
useradd -m -s /bin/bash "$1" && echo "Usuari $1 creat"
```

Els missatges d'error s'envien a la sortida d'errors (`>&2`) perquè no es barregin amb les dades quan la sortida es redirigeix a un fitxer.

### Encadenar Ordres segons el Resultat

| Forma                  | La segona ordre s'executa...          |
| ---------------------- | ------------------------------------- |
| `ordre1 && ordre2`     | Només si la primera ha funcionat      |
| `ordre1 \|\| ordre2`   | Només si la primera ha fallat         |
| `ordre1 ; ordre2`      | Sempre                                |

```bash
mkdir -p /var/backups/web && echo "Directori preparat"
cd /var/backups/web || exit 1          # si no s'hi pot entrar, millor no continuar
```

### Mode Estricte: `set -euo pipefail`

Per defecte, si una ordre falla, l'script continua com si res. En un script d'administració això és perillós. Aquestes opcions, al principi de l'script, fan que s'aturi davant dels errors:

```bash
#!/bin/bash
set -euo pipefail
```

| Opció          | Efecte                                                                          |
| -------------- | ------------------------------------------------------------------------------- |
| `-e`           | Atura l'script quan una ordre falla                                             |
| `-u`           | Atura l'script si es fa servir una variable que no existeix (detecta errors d'escriptura) |
| `-o pipefail`  | Una canonada falla si falla qualsevol de les seves ordres, no només l'última    |

**Per què és important `-u`?** Si el nom d'una variable està mal escrit, sense `-u` queda buida i l'ordre `rm -rf "$directori_copies"/*` esborraria `/*`. Amb `-u`, l'script s'atura abans. Una protecció addicional és `${variable:?}`, que atura l'script si la variable és buida: `rm -rf "${directori_copies:?}"/*`.

`Nota:` Amb `-e`, una ordre que falla dins de la condició d'un `if` o abans d'un `&&` o d'un `||` no atura l'script, perquè es considera que el resultat ja es controla.

### Netejar en Acabar: `trap`

`trap` executa una ordre quan l'script acaba o rep un senyal (Tema 2, fitxer 08). És útil per esborrar fitxers temporals:

```bash
temporal=$(mktemp)                     # fitxer temporal amb un nom únic
trap 'rm -f "$temporal"' EXIT          # s'esborra en acabar, encara que hi hagi un error
```

### Esperar Processos: `wait`

Un script pot llançar diverses tasques en segon pla (`&`) perquè s'executin alhora i, després, esperar-les amb `wait`:

```bash
#!/bin/bash
# copies_paralleles.sh - Executeu-lo amb sudo
tar czf /tmp/etc.tar.gz -C / etc &
tar czf /tmp/home.tar.gz -C / home &
wait                                   # espera que acabin totes les tasques en segon pla
echo "Totes les còpies han acabat"
```

`wait PID` espera un sol procés i en retorna el codi de sortida.

### Depuració

| Eina                    | Funció                                                                    |
| ----------------------- | ------------------------------------------------------------------------- |
| `bash -n script.sh`     | Comprova la sintaxi sense executar res                                    |
| `bash -x script.sh`     | Mostra cada ordre, amb les variables ja substituïdes, abans d'executar-la |
| `set -x` / `set +x`     | Activa / desactiva la traça només en una part de l'script                 |
| `shellcheck script.sh`  | Analitza l'script i avisa dels errors habituals (paquet `shellcheck`)     |

```
$ bash -x hola.sh
+ echo 'Hola, món!'
Hola, món!
++ hostname
+ echo 'Sóc l'\''usuari xavi i estic al servidor srv0374'
Sóc l'usuari xavi i estic al servidor srv0374
```

Les línies que comencen per `+` són les ordres que s'executen (`++` indica una ordre dins d'una altra). `shellcheck` detecta, per exemple, variables sense cometes o comparacions mal escrites, i explica com corregir-les. Passeu-lo sempre abans de lliurar o posar en marxa un script.

### Bones Pràctiques

1. Capçalera amb comentaris: què fa l'script, com s'utilitza, autor i data.
2. `#!/bin/bash` i `set -euo pipefail` al principi.
3. Variables entre cometes dobles: `"$variable"`.
4. Comprovar els paràmetres i mostrar com s'utilitza l'script si són incorrectes.
5. Missatges d'error a `>&2` i `exit` amb un codi diferent de 0.
6. Rutes absolutes als fitxers importants, sobretot si l'script s'ha de programar amb `cron` (Tema 3).
7. Funcions per a les parts que es repeteixen, amb variables `local`.
8. Provar-lo amb `bash -n`, `bash -x` i `shellcheck`.

### Resolució de Problemes

| Problema                                                       | Causa                                                      | Solució                                                |
| -------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------ |
| L'script continua després d'un error greu                      | No té `set -e` ni comprovacions                            | `set -euo pipefail` o `ordre \|\| exit 1`              |
| `unbound variable`                                             | `set -u` ha detectat una variable no definida              | Definir-la o donar-li un valor per defecte: `${var:-}` |
| L'script acaba sense cap missatge                              | Amb `set -e`, ha fallat una ordre                          | `bash -x script.sh` per veure quina                    |
| L'script s'atura just després de `((comptador++))`             | Amb `set -e`, `((0++))` retorna error perquè l'expressió val 0 | `comptador=$((comptador + 1))`                     |
| A mà funciona però programat amb `cron` falla                  | Entorn diferent (`PATH`, directori de treball)             | Rutes absolutes (Tema 3)                               |
