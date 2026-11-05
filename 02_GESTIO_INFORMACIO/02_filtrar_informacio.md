## 02. Filtrar i Processar la Informació

Les ordres d'informació sovint mostren molt més del que necessitem: `ps aux` pot llistar 150 processos i `lastb`, milers d'intents fallits. Un administrador ha de saber **extreure només la dada que busca**: un número, una columna, les cinc línies més rellevants. Aquest fitxer presenta les eines de filtratge que fareu servir durant tot el tema i al Tema 7 (Scripting).

### Redireccions i Canonades

Cada ordre té tres canals: l'entrada estàndard (*stdin*), la sortida estàndard (*stdout*) i la sortida d'errors (*stderr*).

| Operador | Funció                                                      | Exemple                              |
| -------- | ----------------------------------------------------------- | ------------------------------------ |
| `>`      | Envia la sortida a un fitxer (el sobreescriu)               | `lscpu > cpu.txt`                    |
| `>>`     | Afegeix la sortida al final d'un fitxer                     | `date >> registre.txt`               |
| `2>`     | Envia els errors a un fitxer                                | `find / -name "*.conf" 2> /dev/null` |
| `&>`     | Envia la sortida i els errors al mateix fitxer              | `sudo apt update &> apt.log`         |
| `\|`     | Canonada (*pipe*): la sortida d'una ordre és l'entrada de la següent | `ps aux \| wc -l`           |
| `tee`    | Mostra per pantalla **i** desa a un fitxer                  | `free -h \| tee memoria.txt`         |

`/dev/null` és un "forat negre": tot el que s'hi envia es descarta. És útil per amagar els errors de permisos (`Permission denied`) quan no ens interessen.

Una canonada es llegeix d'esquerra a dreta i cada ordre fa una sola feina:

```bash
ps aux | grep sshd | wc -l
#  │          │        └── 3. compta les línies
#  │          └── 2. es queda les línies que contenen "sshd"
#  └── 1. llista tots els processos
```

### `grep`: Cercar Línies

| Opció           | Funció                                                    |
| --------------- | --------------------------------------------------------- |
| `-i`            | No distingeix majúscules i minúscules                     |
| `-v`            | Inverteix: mostra les línies que **no** coincideixen      |
| `-c`            | Compta les línies que coincideixen                        |
| `-n`            | Mostra el número de línia                                 |
| `-w`            | Paraula sencera (`-w 22` no troba `2222`)                 |
| `-E`            | Expressions regulars esteses (`a\|b`, `+`, `?`)           |
| `-o`            | Mostra només el tros que coincideix                       |
| `-A n` / `-B n` | Mostra `n` línies després / abans de cada coincidència    |
| `-r`            | Cerca dins de tots els fitxers d'un directori             |

```bash
grep -c "^processor" /proc/cpuinfo                 # nombre de CPU lògiques
grep -E "^(root|xavi):" /etc/passwd                # línies que comencen per root: o xavi:
grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"   # configuració sense comentaris ni línies buides
grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" fitxer.log   # extreu les adreces IP
```

**Expressions regulars bàsiques:**

| Símbol         | Significat                                   | Exemple            |
| -------------- | -------------------------------------------- | ------------------ |
| `^`            | Inici de línia                               | `^root`            |
| `$`            | Final de línia                               | `bash$`            |
| `.`            | Qualsevol caràcter                           | `s.d` (`sed`, `sad`) |
| `*`            | L'element anterior, zero o més vegades       | `ab*` (`a`, `ab`, `abb`) |
| `[...]`        | Un dels caràcters de la llista               | `[0-9]`            |
| `+`, `?`, `\|` | Amb `-E`: una o més vegades, opcional, alternativa | `-E "ssh\|ftp"` |

### `cut`: Extreure Camps amb un Separador Fix

`cut` va bé quan les columnes estan separades per un caràcter fix (`:`, `,`, tabulador):

```bash
cut -d: -f1 /etc/passwd            # camp 1 (nom d'usuari)
cut -d: -f1,3,7 /etc/passwd        # camps 1, 3 i 7 (nom, UID i shell)
cut -c1-10 fitxer.txt              # caràcters de l'1 al 10 de cada línia
```

`Nota:` `cut` no funciona bé amb sortides alineades amb espais (com `ps` o `df`), perquè cada espai compta com un separador. Per a aquests casos feu servir `awk`.

### `awk`: Processar per Columnes

`awk` divideix cada línia en camps (`$1`, `$2`...) separats per un o més espais. `$0` és la línia sencera, `NF` el nombre de camps de la línia i `NR` el número de línia. L'estructura és `awk 'condició {acció}'`: sense condició, l'acció s'aplica a totes les línies; sense acció, es mostra la línia.

```bash
df -h | awk '{print $1, $5}'                       # dispositiu i % d'ús
awk -F: '{print $1, $7}' /etc/passwd               # -F canvia el separador
awk -F: '$3 >= 1000 {print $1}' /etc/passwd        # condició: UID >= 1000
ps aux | awk 'NR > 1 {print $2, $11}'              # salta la capçalera (línia 1)
ps aux | awk '{suma += $4} END {print suma "%"}'   # suma la columna %MEM
df -h | awk '$5+0 > 80 {print $6, $5}'             # sistemes de fitxers per sobre del 80%
free -h | awk '/^Mem:/ {print $2}'                 # condició amb expressió regular: línia que comença per Mem:
```

`$5+0` converteix un text com `47%` en el número `47` per poder-lo comparar.

### `sort`, `uniq`, `wc`, `head` i `tail`

| Ordre  | Funció                                       | Opcions habituals                                                                 |
| ------ | -------------------------------------------- | --------------------------------------------------------------------------------- |
| `sort` | Ordena línies                                | `-n` numèric, `-r` invers, `-k3` per la columna 3, `-t:` separador, `-h` mides (`2K`, `1G`), `-u` sense repetits |
| `uniq` | Elimina línies repetides **consecutives**    | `-c` compta les repeticions                                                       |
| `wc`   | Compta                                       | `-l` línies, `-w` paraules, `-c` bytes                                            |
| `head` | Primeres línies                              | `-n 5`                                                                            |
| `tail` | Últimes línies                               | `-n 5`, `-n +2` (des de la línia 2), `-f` segueix el fitxer en temps real         |

```bash
sort -t: -k3 -n /etc/passwd | tail -n 3              # els 3 comptes amb l'UID més alt
sudo du -sh /var/* 2>/dev/null | sort -rh | head -n 5   # els 5 directoris més grans de /var
```

**El patró més útil: rànquing amb `sort | uniq -c | sort -rn`**

```bash
# Quina shell té assignada cada compte del sistema?
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```

```
     26 /usr/sbin/nologin
      3 /bin/false
      2 /bin/bash
      1 /bin/sync
```

`uniq` només ajunta les línies iguals que estan **una al costat de l'altra**. Per això sempre cal `sort` abans. El segon `sort -rn` ordena pel nombre de repeticions, de més a menys.

### Altres Eines Útils

| Ordre       | Funció                                              | Exemple                                                  |
| ----------- | --------------------------------------------------- | -------------------------------------------------------- |
| `tr`        | Substitueix o elimina caràcters                     | `getent group sudo \| cut -d: -f4 \| tr ',' '\n'`        |
| `sed`       | Edita el text en flux                               | `sed 's/enp0s3/WAN/'`, `sed -n '5,10p' fitxer`           |
| `xargs`     | Converteix línies en arguments d'una altra ordre    | `find /var/log -name "*.gz" \| xargs du -ch \| tail -n 1` |
| `column -t` | Alinea la sortida en columnes                       | `mount \| column -t`                                     |

### Variables i Substitució d'Ordres

Per construir informes es desa el resultat d'una ordre en una variable amb `$(...)`:

```bash
NUCLIS=$(nproc)
RAM=$(free -h | awk '/^Mem:/ {print $2}')
echo "El servidor $(hostname) té $NUCLIS nuclis i $RAM de RAM"
```

```
El servidor srv0374 té 2 nuclis i 3.8Gi de RAM
```

### Primer Script d'Informe

Un script és un fitxer de text amb ordres que s'executen una darrere l'altra. Així no cal repetir les mateixes ordres cada vegada:

```bash
#!/bin/bash
# informe-srv0374.sh - Informe bàsic del servidor

echo "=== INFORME DEL SERVIDOR $(hostname) ==="
echo "Data: $(date '+%d/%m/%Y %H:%M')"
echo

echo "--- Sistema ---"
echo "Sistema operatiu: $(grep PRETTY_NAME /etc/os-release | cut -d'"' -f2)"
echo "Kernel: $(uname -r)"
echo

echo "--- Recursos ---"
echo "CPU: $(nproc) nuclis"
echo "RAM total: $(free -h | awk '/^Mem:/ {print $2}')"
echo "Disc arrel: $(df -h / | awk 'NR==2 {print $5}') ocupat"
```

```bash
chmod +x informe-srv0374.sh
./informe-srv0374.sh                               # mostrar l'informe per pantalla
./informe-srv0374.sh > informe-$(date +%F).txt     # desar-lo amb la data al nom (informe-2026-11-05.txt)
```

A les pràctiques ampliareu aquest script amb la informació de cada fitxer del tema. Al **Tema 3** el programareu perquè s'executi sol cada dia i al **Tema 7** hi afegireu condicions, bucles i funcions.

### Resolució de Problemes

| Problema                                              | Causa                                                    | Solució                                         |
| ----------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------- |
| `uniq -c` compta malament                             | Les línies iguals no són consecutives                    | Posar `sort` abans d'`uniq`                     |
| `cut -d' '` retorna camps buits                       | La sortida té diversos espais seguits                    | Fer servir `awk '{print $N}'`                   |
| `ps aux \| grep sshd` mostra també el mateix `grep`   | El procés `grep sshd` també conté el text "sshd"         | Fer servir `pgrep sshd` o `grep '[s]shd'`       |
| `awk` no separa pels `:`                              | No s'ha indicat el separador                             | `awk -F: ...`                                   |
| `./script.sh: Permission denied`                      | El fitxer no té permís d'execució                        | `chmod +x script.sh`                            |
| `/bin/bash^M: bad interpreter`                        | El fitxer s'ha editat a Windows (finals de línia CRLF)   | `sed -i 's/\r$//' script.sh`                    |
