## 04. Accés als Recursos Compartits des de Windows i des de Linux

### Des de Windows

**Explorador de fitxers:** escriure a la barra d'adreces `\\10.0.1.37` (la IP del servidor a la xarxa del centre) per veure'n els recursos, o directament `\\10.0.1.37\administracio`. Windows demana l'usuari i la contrasenya de Samba.

**Connectar una unitat de xarxa** (lletra fixa, com `Z:`):

```powershell
net use Z: \\10.0.1.37\administracio /user:anna /persistent:yes
net use                                  # llistar les unitats connectades
net use Z: /delete                       # desconnectar-la
```

També es pot fer des de l'Explorador: *Aquest PC* → *Connecta una unitat de xarxa*.

**Credencials desades:** Windows guarda les contrasenyes a l'*Administrador de credencials* (Tauler de control). Si una contrasenya canvia al servidor, cal esborrar-hi l'entrada antiga.

`Nota:` Windows 10 i 11 bloquegen l'accés de **convidat** (sense contrasenya) als recursos SMB i no fan servir SMB1. Configureu sempre els recursos amb usuaris autenticats.

### Des de Linux: `smbclient`

```bash
sudo apt install smbclient cifs-utils
smbclient -L //192.168.1.1 -U anna                 # llistar els recursos del servidor
smbclient //192.168.1.1/administracio -U anna      # sessió interactiva, semblant a sftp
```

```
smb: \> ls
smb: \> put informe.odt
smb: \> get pressupost.ods
smb: \> exit
```

### Des de Linux: Muntar el Recurs (`mount -t cifs`)

**1. Fitxer de credencials** (perquè la contrasenya no quedi a l'historial ni a `/etc/fstab`):

```bash
sudo nano /root/.smbcredencials
```

```
username=anna
password=contrasenya_d_anna
```

```bash
sudo chmod 600 /root/.smbcredencials
```

**2. Muntar-lo a mà:**

```bash
sudo mkdir -p /mnt/administracio
sudo mount -t cifs //192.168.1.1/administracio /mnt/administracio -o credentials=/root/.smbcredencials,uid=alumne,gid=alumne
```

Les opcions `uid` i `gid` indiquen de quin usuari local semblaran els fitxers muntats.

**3. Muntar-lo en arrencar** (`/etc/fstab`):

```
//192.168.1.1/administracio  /mnt/administracio  cifs  credentials=/root/.smbcredencials,uid=alumne,gid=alumne,_netdev  0  0
```

`_netdev` indica que és un recurs de xarxa: no es munta fins que la xarxa està a punt. Comproveu la línia amb `sudo mount -a` abans de reiniciar.

**Escriptori d'Ubuntu:** a *Fitxers* → *Altres ubicacions* → `smb://192.168.1.1/administracio`.

### Resolució de Problemes

| Problema                                                     | Causa                                                        | Solució                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------------------------------- |
| Windows: *no podeu accedir a aquesta carpeta compartida perquè les directives de seguretat bloquegen l'accés de convidat* | El recurs és de convidat | Fer servir usuaris autenticats (`valid users`)              |
| Windows continua fent servir la contrasenya antiga           | Credencials desades                                          | Esborrar-les a l'Administrador de credencials               |
| `mount error(13): Permission denied`                         | Usuari o contrasenya incorrectes al fitxer de credencials    | Revisar el fitxer i `sudo smbpasswd -a usuari` al servidor  |
| `mount: wrong fs type, bad option, bad superblock`           | Falta el paquet `cifs-utils`                                 | `sudo apt install cifs-utils`                               |
| L'equip no arrenca bé després d'editar `/etc/fstab`          | Línia incorrecta o servidor inaccessible sense `_netdev`/`nofail` | Provar sempre amb `sudo mount -a`; afegir `nofail`      |
