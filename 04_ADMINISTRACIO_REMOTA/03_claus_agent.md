## 03. Gestió de Claus i Agent SSH

### Tipus de Claus

| Tipus       | Ordre                              | Característiques                                             |
| ----------- | ---------------------------------- | ------------------------------------------------------------ |
| **Ed25519** | `ssh-keygen -t ed25519`            | Recomanada: segura, ràpida i curta                           |
| RSA         | `ssh-keygen -t rsa -b 4096`        | Compatible amb equips antics. Per a claus noves no es recomanen mides inferiors a 3072 bits |
| ECDSA       | `ssh-keygen -t ecdsa -b 521`       | Alternativa basada en corbes el·líptiques                    |

Cada clau privada s'ha de protegir amb una **frase de pas**: si algú copia el fitxer, no la podrà fer servir sense la frase. Per canviar-la o afegir-la a una clau existent: `ssh-keygen -p -f ~/.ssh/id_ed25519`.

### L'Agent SSH: Escriure la Frase de Pas una Sola Vegada

Amb frase de pas, cada connexió la demana. L'**agent SSH** guarda la clau desxifrada a la memòria durant la sessió de treball:

**A Linux:**

```bash
eval "$(ssh-agent)"                    # engegar l'agent (els escriptoris ja en tenen un)
ssh-add ~/.ssh/id_ed25519              # afegir la clau: demana la frase de pas una vegada
ssh-add -l                             # llistar les claus carregades
ssh-add -D                             # treure-les totes
```

**A Windows** (PowerShell com a administrador, una sola vegada):

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

MobaXterm té el seu propi agent (*Settings* → *SSH* → *SSH agents* → *Use internal SSH agent "MobAgent"*).

**Reenviament de l'agent** (`ssh -A` o `ForwardAgent yes`): permet fer servir les claus de l'ordinador des d'un servidor intermedi. És arriscat: qualsevol amb accés de root a aquell servidor podria fer servir les vostres claus mentre esteu connectats. Per saltar entre servidors és millor `ProxyJump` (fitxer 05), que no ho necessita.

### Diverses Claus

És una bona pràctica tenir claus diferents per a usos diferents (feina, laboratori, GitHub, còpies automàtiques) i indicar-les al fitxer `~/.ssh/config` amb `IdentityFile` (fitxer 02). Si una clau es veu compromesa, només cal revocar-la a ella.

### Opcions al Fitxer `authorized_keys`

Davant de cada clau pública es poden posar opcions que en limiten l'ús:

```
from="10.0.1.0/24",restrict ssh-ed25519 AAAAC3Nza... nom.cognom@aula
command="/usr/local/bin/informe.sh",restrict ssh-ed25519 AAAAC3Nza... informes@srv0375
```

| Opció                  | Efecte                                                                   |
| ---------------------- | ------------------------------------------------------------------------ |
| `from="xarxa o IP"`    | La clau només val des d'aquestes adreces                                 |
| `command="ordre"`      | Amb aquesta clau només es pot executar aquesta ordre                     |
| `restrict`             | Desactiva túnels, reenviament de l'agent, X11 i terminal                 |
| `no-port-forwarding`   | Desactiva només els túnels                                               |

Són especialment útils per a les claus sense frase de pas de les tasques automàtiques (Tema 3, fitxer 06).

**Revocar una clau:** esborrar-ne la línia del fitxer `authorized_keys` de cada servidor on estigui autoritzada. Per això convé que el comentari final de cada clau (`nom.cognom@aula`) identifiqui clarament de qui és.

### Claus dels Servidors i `known_hosts`

Cada servidor té les seves pròpies claus (`/etc/ssh/ssh_host_*_key`), que el client desa a `~/.ssh/known_hosts` en la primera connexió. Si la clau d'un servidor canvia, el client es nega a connectar-se: pot ser un atac de suplantació (*man-in-the-middle*) o una reinstal·lació del servidor.

| Ordre (al client)                          | Funció                                                    |
| ------------------------------------------ | --------------------------------------------------------- |
| `ssh-keygen -F 10.0.1.37`                  | Comprova si un servidor és a `known_hosts`                |
| `ssh-keygen -R 10.0.1.37`                  | Esborra la clau antiga d'un servidor (després d'una reinstal·lació verificada) |
| `ssh-keyscan 10.0.1.37`                    | Obté la clau pública d'un servidor                        |

| Ordre (al servidor)                                      | Funció                                       |
| -------------------------------------------------------- | -------------------------------------------- |
| `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`       | Empremta de la clau del servidor             |
| `sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server` | Regenera les claus (per exemple, en clonar una màquina virtual) |

Les màquines virtuals **clonades** comparteixen les claus del servidor original. Si se'n clonen diverses, cal regenerar-les a cada còpia.

### Resolució de Problemes

| Problema                                                        | Causa                                                         | Solució                                                      |
| --------------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------ |
| La clau no funciona i el servidor demana contrasenya            | Permisos massa oberts a `~/.ssh` o `authorized_keys` (el servidor els ignora) | `chmod 700 ~/.ssh`, `chmod 600 ~/.ssh/authorized_keys`, i el directori personal sense permís d'escriptura per al grup |
| `Could not open a connection to your authentication agent`      | L'agent no està en marxa                                      | `eval "$(ssh-agent)"` (Linux) o engegar el servei `ssh-agent` (Windows) |
| `REMOTE HOST IDENTIFICATION HAS CHANGED`                        | El servidor té claus noves (reinstal·lació o clonació) o hi ha una suplantació | Verificar l'empremta i, si és correcta, `ssh-keygen -R`     |
| Dues VM clonades tenen la mateixa empremta                      | Comparteixen les claus del servidor                           | Regenerar-les amb `dpkg-reconfigure openssh-server`          |
| Una clau amb `command=` no permet obrir una sessió normal       | És el comportament esperat de l'opció                         | Fer servir una altra clau per a l'administració              |
