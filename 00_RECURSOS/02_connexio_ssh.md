## 02. Connexió Inicial per SSH

Aquesta guia es fa **a l'inici del curs**: a partir d'aquí, el servidor 0374 s'administra des de l'ordinador de l'aula per SSH, sense fer servir la finestra de VirtualBox. L'administració remota completa (túnels, bastionat avançat, escriptori remot) es treballa al [Tema 4](../04_ADMINISTRACIO_REMOTA/README.md).

### Què és SSH?

**SSH** (*Secure Shell*) és el protocol que permet obrir una sessió en un equip remot a través d'un canal **xifrat**: tot el que s'envia (usuari, contrasenya, ordres i resultats) viatja protegit. Fa servir el **port 22/TCP**. SSH té el seu propi protocol de xifrat (no fa servir TLS, com HTTPS).

Hi ha dues maneres d'autenticar-se:

| Mètode             | Com funciona                                                        | Seguretat                                         |
| ------------------ | ------------------------------------------------------------------- | ------------------------------------------------- |
| **Contrasenya**    | L'usuari escriu la contrasenya del compte del servidor              | Vulnerable a atacs de força bruta (Tema 2, fitxer 05) |
| **Clau pública**   | El client demostra que té la **clau privada** que correspon a una **clau pública** autoritzada al servidor | Molt més segura: la clau privada no surt mai de l'equip client |

L'objectiu d'aquesta guia és acabar amb l'accés **només amb claus**.

### Pas 1. Comprovar la Xarxa

Segons l'[escenari del laboratori](./01_escenari_laboratori.md), la interfície `enp0s3` del servidor està en mode **adaptador pont**: rep una IP de la xarxa del centre, la mateixa que l'ordinador de l'aula.

**1.** A l'ordinador (Windows), obriu un terminal i consulteu-ne la IP:

```powershell
ipconfig
```

**2.** A VirtualBox, comproveu que l'adaptador 1 de la màquina virtual és **Adaptador pont** i que fa servir la targeta de xarxa física correcta (la de cable o la Wi-Fi, segons com estigui connectat l'ordinador).

**3.** Al servidor, consulteu la IP d'`enp0s3`:

```bash
ip -br a
```

**4.** Des de l'ordinador, comproveu que hi arriba:

```powershell
ping 10.0.1.37
```

`Nota:` A casa, algunes targetes Wi-Fi no funcionen bé en mode pont. L'alternativa és posar l'adaptador en mode **NAT** amb una regla de reenviament de ports (VirtualBox → Xarxa → Avançat → Reenviament de ports: port amfitrió `2222` → port convidat `22`) i connectar-se a `127.0.0.1`, port `2222`.

### Pas 2. El Servei SSH al Servidor

L'instal·lador d'Ubuntu Server ofereix instal·lar el servidor OpenSSH. Si no es va marcar l'opció:

```bash
sudo apt install openssh-server
```

A Ubuntu 24.04, l'SSH s'activa **per socket**: `systemd` escolta el port 22 (`ssh.socket`) i engega el servei `ssh` quan arriba la primera connexió (Tema 2, fitxer 09).

```bash
systemctl status ssh.socket          # ha d'estar active (listening)
sudo ss -tlnp | grep :22             # el port 22 en escolta
```

### Pas 3. Primera Connexió

**Des del terminal de Windows** (Windows 10 i 11 porten el client OpenSSH):

```powershell
ssh xavi@10.0.1.37
```

**Des de MobaXterm:** *Session* → *SSH* → *Remote host*: la IP del servidor, *Specify username*: el vostre usuari → *OK*. No deseu la contrasenya: després l'accés serà amb clau.

La primera vegada, el client mostra l'**empremta digital** (*fingerprint*) de la clau del servidor i pregunta si és de confiança:

```
The authenticity of host '10.0.1.37 (10.0.1.37)' can't be established.
ED25519 key fingerprint is SHA256:Xq3ZP0v9cR1h0W8lYk2c7fWJm6d0Qe4v5T1sN2bA7uE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.1.37' (ED25519) to the list of known hosts.
```

En respondre `yes`, l'empremta es desa al fitxer `known_hosts` del client. A partir d'aquí, si l'empremta canvia, el client avisarà: pot voler dir que algú suplanta el servidor. Per comprovar-la abans d'acceptar-la, al servidor:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

### Pas 4. Autenticació amb Claus

**Com funciona:**

1. Es genera un **parell de claus** a l'equip **client**: la privada (secreta) i la pública.
2. La clau **pública** es copia al servidor, al fitxer `~/.ssh/authorized_keys` de l'usuari.
3. En connectar-se, el servidor demana al client que signi unes dades amb la clau privada. Amb la clau pública, el servidor comprova la signatura: si és correcta, el client té la clau privada i se li permet l'accés.

**La clau privada no surt mai de l'equip client.** No s'ha de generar al servidor ni copiar-se d'un equip a un altre: qualsevol persona que l'aconseguís podria entrar al servidor.

**4.1. Generar les claus a l'ordinador (terminal de Windows):**

```powershell
ssh-keygen -t ed25519 -C "nom.cognom@aula"
```

- Accepteu la ruta per defecte (`C:\Users\<usuari>\.ssh\id_ed25519`).
- Escriviu una **frase de pas** (*passphrase*): protegeix la clau privada si algú copia el fitxer.

Es creen dos fitxers: `id_ed25519` (clau **privada**) i `id_ed25519.pub` (clau **pública**). Ed25519 és el tipus de clau recomanat actualment: més segur i més curt que RSA.

**4.2. Copiar la clau pública al servidor:**

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh xavi@10.0.1.37 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Aquesta ordre afegeix la clau pública al final d'`authorized_keys` i deixa els permisos que exigeix SSH: el directori `~/.ssh` amb `700` i el fitxer amb `600`. Des d'un client Linux, `ssh-copy-id xavi@10.0.1.37` fa el mateix.

**4.3. Comprovar la connexió amb la clau:**

```powershell
ssh xavi@10.0.1.37
```

Ara el client demana la **frase de pas de la clau**, no la contrasenya de l'usuari del servidor.

**Amb MobaXterm:** el generador *Tools* → *MobaKeyGen* crea claus Ed25519 (fitxer `.ppk`) i mostra la clau pública en format OpenSSH per enganxar-la a `authorized_keys`. A la sessió, *Advanced SSH settings* → *Use private key* → el fitxer de la clau privada.

### Pas 5. Desactivar l'Accés amb Contrasenya

Quan l'accés amb clau funciona, es desactiva l'accés amb contrasenya i el de root.

**Important:** no tanqueu la sessió SSH oberta fins que hàgiu comprovat, amb una connexió nova, que podeu entrar. Si us quedeu fora, sempre podeu entrar per la consola de VirtualBox.

**5.1. Crear un fitxer de configuració propi.** A Ubuntu 24.04, el fitxer principal `/etc/ssh/sshd_config` comença llegint els fitxers de `/etc/ssh/sshd_config.d/`, i **per a cada paràmetre val el primer valor que es llegeix**. L'instal·lador pot haver creat `50-cloud-init.conf` amb `PasswordAuthentication yes`, que anul·laria un canvi fet al fitxer principal. Per això es crea un fitxer amb un número més baix:

```bash
sudo nano /etc/ssh/sshd_config.d/10-seguretat.conf
```

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Els fitxers de configuració s'editen sempre amb `sudo` i un editor (`nano`): no se n'han de canviar mai els permisos per editar-los des d'un altre programa.

**5.2. Comprovar la configuració i aplicar-la:**

```bash
sudo sshd -t                                                          # sense sortida = sintaxi correcta
sudo sshd -T | grep -E "^(passwordauthentication|permitrootlogin)"    # valors que s'aplicaran
sudo systemctl restart ssh
```

```
permitrootlogin no
passwordauthentication no
```

**5.3. Comprovar que la contrasenya ja no funciona.** Des d'un altre terminal, forceu una connexió amb contrasenya:

```powershell
ssh -o PubkeyAuthentication=no xavi@10.0.1.37
```

```
xavi@10.0.1.37: Permission denied (publickey).
```

El servidor ja només accepta claus. Les altres mesures (canviar el port, limitar els usuaris amb `AllowUsers`, tallafoc, `fail2ban`) es veuen al Tema 4.

### Pas 6. Copiar Fitxers

- **MobaXterm:** el panell lateral de la sessió (SFTP) permet arrossegar fitxers entre l'ordinador i el servidor.
- **Terminal:** `scp` i `sftp` fan servir la mateixa connexió i les mateixes claus:

```powershell
scp .\script.sh xavi@10.0.1.37:~/scripts/          # ordinador → servidor
scp xavi@10.0.1.37:/tmp/informe.txt .               # servidor → ordinador
```

### Resolució de Problemes

| Problema                                                    | Causa probable                                                  | Solució                                                          |
| ----------------------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------- |
| `Connection timed out`                                      | IP incorrecta o l'adaptador no està en mode pont                | Revisar `ip -br a` al servidor i la configuració de VirtualBox   |
| `Connection refused`                                        | El servidor SSH no està instal·lat o el port està tancat        | `sudo apt install openssh-server`; revisar el tallafoc           |
| `Permission denied (publickey)` amb la clau                 | La clau pública no és a `authorized_keys`, permisos incorrectes o usuari equivocat | Revisar el fitxer i els permisos (`700` i `600`)  |
| Després del pas 5, la contrasenya continua funcionant       | Un fitxer de `sshd_config.d` amb número més baix diu `yes`      | `sudo sshd -T \| grep passwordauthentication` i revisar `/etc/ssh/sshd_config.d/` |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`          | El servidor s'ha reinstal·lat i té claus noves (o algú el suplanta) | Si és un servidor reinstal·lat: `ssh-keygen -R 10.0.1.37` al client |
| `sshd -t` diu `Missing privilege separation directory: /run/sshd` | El servei està aturat i el directori temporal no existeix | `sudo mkdir -p /run/sshd`                                       |
| Windows diu `UNPROTECTED PRIVATE KEY FILE`                  | La clau privada té permisos per a altres usuaris de Windows     | Deixar-ne només els permisos del vostre usuari (Propietats → Seguretat) |
