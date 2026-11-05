## 02. Variables i Paràmetres

### Variables

Una **variable** és un nom que guarda un valor per fer-lo servir més endavant.

```bash
nom="Anna"                  # assignació: SENSE espais al voltant de =
edat=19
echo "Hola, $nom"           # $nom: valor de la variable
echo "Tens ${edat} anys"    # ${...}: delimita on acaba el nom de la variable
```

| Regla                                            | Correcte              | Incorrecte                                                     |
| ------------------------------------------------ | --------------------- | -------------------------------------------------------------- |
| Sense espais al voltant del `=`                  | `servidor="srv0374"`  | `servidor = "srv0374"` (bash intenta executar l'ordre `servidor`) |
| El nom comença per una lletra o per `_`          | `ip_servidor`         | `1ip`                                                          |
| Majúscules i minúscules són diferents            | `$nom` i `$NOM` són dues variables | —                                                 |
| Dins dels scripts, noms en minúscules            | `fitxer_log`          | `PATH` (sobreescriuria una variable del sistema)               |

**Tipus de dades.** Per a bash, totes les variables són text. Segons on es facin servir, es tracten com a números (en una operació aritmètica) o com a llistes (en un vector):

```bash
numero=42                   # enter
decimal=3.14                # bash no fa operacions amb decimals (fitxer 04)
cadena="Administració de Sistemes"
vector=(1 2 3)              # vector (fitxer 09)
echo "${vector[@]}"         # 1 2 3
```

### Cometes

| Cometes            | Comportament                                                | Exemple                  | Resultat           |
| ------------------ | ----------------------------------------------------------- | ------------------------ | ------------------ |
| Dobles `"..."`     | Substitueixen les variables i les ordres                    | `echo "Usuari: $USER"`   | `Usuari: xavi`     |
| Simples `'...'`    | Tot és literal: no substitueixen res                        | `echo 'Usuari: $USER'`   | `Usuari: $USER`    |
| Sense cometes      | Substitueixen, però separen per espais i expandeixen `*`    | `echo $cadena`           | Resultats inesperats |

**Regla pràctica: poseu sempre les variables entre cometes dobles** (`"$fitxer"`). Si el valor té espais o és buit, sense cometes l'script falla o fa una cosa diferent de la prevista.

### Substitució d'Ordres

`$(ordre)` executa l'ordre i en guarda la sortida:

```bash
directori=$(pwd)
sessions=$(who | wc -l)
echo "Estàs a $directori i hi ha $sessions sessions obertes"
```

La forma antiga amb accents greus (`` `ordre` ``) fa el mateix, però és més difícil de llegir.

### Variables Locals i Variables d'Entorn

- Una variable normal només existeix **al shell on s'ha creat**.
- Amb `export`, la variable passa a ser una **variable d'entorn**: la reben tots els programes i scripts que s'executin **des d'aquest shell** (els processos fills).

```bash
nom="Anna"
bash -c 'echo "Nom: $nom"'       # Nom:        (el shell fill no la veu)
export nom
bash -c 'echo "Nom: $nom"'       # Nom: Anna   (ara sí)
```

Un procés fill **no pot** modificar les variables del pare. Per això, un script que crea variables o fa `cd` no canvia el terminal des del qual s'executa (excepte si s'executa amb `source`).

| Ordre                  | Funció                                                          |
| ---------------------- | --------------------------------------------------------------- |
| `env` o `printenv`     | Mostra les variables d'entorn                                   |
| `printenv HOME`        | Mostra una variable d'entorn concreta                           |
| `export NOM="valor"`   | Crea una variable d'entorn                                      |
| `unset NOM`            | Esborra una variable                                            |
| `env NOM=valor ordre`  | Executa **una sola ordre** amb una variable d'entorn afegida    |

Variables d'entorn habituals: `USER`, `HOME`, `SHELL`, `PATH`, `PWD`, `LANG` i `HOSTNAME`. Per fer permanent una variable d'entorn per a un usuari, afegiu-ne l'`export` al final del fitxer `~/.bashrc`.

### Paràmetres d'un Script

Els valors que s'escriuen després del nom de l'script arriben com a **paràmetres posicionals**:

```bash
#!/bin/bash
# parametres.sh - Mostra els paràmetres rebuts
echo "Nom de l'script: $0"
echo "Primer paràmetre: $1"
echo "Segon paràmetre: $2"
echo "Tercer paràmetre: $3"
echo "Nombre de paràmetres: $#"
echo "Tots els paràmetres: $@"
```

```bash
./parametres.sh srv0374 192.168.1.1 "Servidor 0374"
```

```
Nom de l'script: ./parametres.sh
Primer paràmetre: srv0374
Segon paràmetre: 192.168.1.1
Tercer paràmetre: Servidor 0374
Nombre de paràmetres: 3
Tots els paràmetres: srv0374 192.168.1.1 Servidor 0374
```

`"Servidor 0374"` arriba com un sol paràmetre perquè està entre cometes.

| Variable              | Contingut                                                       |
| --------------------- | --------------------------------------------------------------- |
| `$0`                  | Nom de l'script                                                 |
| `$1` ... `$9`, `${10}` | Paràmetres posicionals (a partir del 10, amb claus)            |
| `$#`                  | Nombre de paràmetres                                            |
| `"$@"`                | Tots els paràmetres, cadascun per separat (el més habitual)     |
| `"$*"`                | Tots els paràmetres en una sola cadena                          |
| `$?`                  | Codi de sortida de l'última ordre (0 = correcte, fitxer 11)     |
| `$$`                  | PID de l'script                                                 |
| `$!`                  | PID de l'últim procés llançat en segon pla                      |

`shift` desplaça els paràmetres una posició (`$2` passa a ser `$1`...). És útil per processar-los un a un.

**Valor per defecte:** `${1:-valor}` fa servir `valor` si no s'ha passat el paràmetre:

```bash
servidor=${1:-localhost}           # si no hi ha primer paràmetre, localhost
```

### Resolució de Problemes

| Problema                                                    | Causa                                                  | Solució                                        |
| ----------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------- |
| `servidor: command not found` en assignar una variable      | Espais al voltant del `=`                              | `servidor="srv0374"`                           |
| La variable surt buida                                      | Nom mal escrit (majúscules) o no exportada a un altre script | Revisar el nom; `export` si l'ha de rebre un altre programa |
| `too many arguments` o errors amb noms que tenen espais     | Variable sense cometes                                 | `"$variable"`                                  |
| Les variables de l'script no queden al terminal             | L'script s'executa en un shell fill                    | `source script.sh`, si és el que es vol        |
| `$10` mostra el primer paràmetre seguit d'un 0              | A partir del 10 calen claus                            | `${10}`                                        |
