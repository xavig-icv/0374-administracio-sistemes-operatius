## 01. Introducció al Bash Scripting

### El Terminal, el Shell i les Ordres

- El **terminal** és el programa que mostra la finestra de text on escrivim (MobaXterm, la consola del servidor...).
- El **shell** és l'intèrpret que llegeix el que escrivim, ho interpreta i ho fa executar al sistema operatiu.
- Una **ordre** és un programa (normalment a `/usr/bin`) o una ordre interna del shell (`cd`, `echo`, `read`...). La majoria d'ordres accepten opcions i paràmetres.

| Shell  | Fitxer      | Característiques                                                                 |
| ------ | ----------- | -------------------------------------------------------------------------------- |
| `sh`   | `/bin/sh`   | Shell POSIX bàsic. A Ubuntu és `dash`: molt ràpid, però amb menys funcions        |
| `bash` | `/bin/bash` | Shell per defecte de la majoria de distribucions Linux. **És el que farem servir** |
| `zsh`  | `/bin/zsh`  | Compatible amb bash en gairebé tot i amb més opcions interactives (macOS)        |

```bash
echo $SHELL          # shell per defecte de l'usuari
ps -p $$             # shell que s'està executant ara mateix
cat /etc/shells      # shells instal·lats al sistema
```

### Per què Fer Scripts?

Un **script** (guió) és un fitxer de text amb ordres que s'executen una darrere l'altra. Per a un administrador de sistemes és l'eina bàsica per:

- **Automatitzar** tasques repetitives: crear usuaris, fer còpies de seguretat, generar informes.
- Fer les tasques **sempre igual**, sense errors humans.
- **Programar-les** perquè s'executin soles a una hora determinada (Tema 3).

A més de les ordres del sistema, bash és un llenguatge de programació: té variables, condicions, bucles i funcions, que es veuen als fitxers següents.

### El Primer Script

**1. Crear el fitxer.** Al servidor, amb `nano` (o `vim`); des de l'ordinador, amb l'editor de MobaXterm o amb VS Code connectat per SSH.

```bash
mkdir -p ~/scripts && cd ~/scripts
nano hola.sh
```

```bash
#!/bin/bash
# hola.sh - El meu primer script
# Autor: nom i cognoms    Data: 05/11/2026

echo "Hola, món!"
echo "Sóc l'usuari $USER i estic al servidor $(hostname)"
```

- La primera línia, `#!/bin/bash` (*shebang*), indica quin intèrpret ha d'executar el fitxer.
- Les línies que comencen per `#` són **comentaris**: no s'executen. Una capçalera amb el nom de l'script, què fa, l'autor i la data és una bona pràctica.

**2. Donar-li permís d'execució:**

```bash
chmod +x hola.sh
ls -l hola.sh
```

```
-rwxrwxr-x 1 xavi xavi 148 Nov  5 10:20 hola.sh
```

**3. Executar-lo:**

```bash
./hola.sh
```

```
Hola, món!
Sóc l'usuari xavi i estic al servidor srv0374
```

| Forma d'execució                  | Què passa                                                                       |
| --------------------------------- | ------------------------------------------------------------------------------- |
| `./hola.sh`                       | El sistema llegeix el *shebang* i l'executa amb bash. Cal el permís d'execució  |
| `bash hola.sh`                    | S'executa amb bash encara que no tingui permís d'execució ni *shebang*          |
| `sh hola.sh`                      | S'executa amb `sh` (`dash`): les funcions pròpies de bash poden fallar          |
| `source hola.sh` (o `. hola.sh`)  | S'executa dins del shell actual: les variables que crea es queden al terminal   |

**Per què cal escriure `./`?** El shell només busca les ordres als directoris de la variable `PATH` (`echo $PATH`), i el directori actual no hi és per seguretat. Per executar els vostres scripts des de qualsevol lloc, deseu-los a `~/bin` (Ubuntu l'afegeix al `PATH` quan existeix, a partir del següent inici de sessió) o, si són per a tots els usuaris, a `/usr/local/bin`.

### Bones Pràctiques des del Primer Dia

- Noms de fitxer amb l'extensió `.sh`, sense espais ni accents: `copia_seguretat.sh`.
- Capçalera de comentaris: què fa l'script, com s'utilitza, autor i data.
- Una ordre per línia i sagnat de 4 espais dins de les condicions i els bucles.
- Proveu els scripts en una màquina de proves, sobretot si esborren o modifiquen fitxers.

### Resolució de Problemes

| Problema                                                  | Causa                                        | Solució                                                      |
| --------------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| `./hola.sh: Permission denied`                            | El fitxer no té permís d'execució            | `chmod +x hola.sh`                                           |
| `hola.sh: command not found`                              | S'ha escrit el nom sense `./`                | `./hola.sh`                                                  |
| `/bin/bash^M: bad interpreter`                            | El fitxer s'ha editat a Windows (finals CRLF) | `sed -i 's/\r$//' hola.sh` o desar-lo amb finals de línia LF |
| Funciona amb `bash script.sh` però no amb `sh script.sh`  | `sh` és `dash`, no bash                      | Executar-lo amb `bash` o amb `./`                            |
| Els accents surten malament                               | El fitxer no està desat en UTF-8             | Desar-lo en UTF-8                                            |
