## 02. Samba: Instal·lació i Configuració Bàsica

**Samba** és la implementació lliure del protocol **SMB** (*Server Message Block*), el que fan servir els equips Windows per compartir carpetes i impressores. Un servidor Linux amb Samba apareix a la xarxa com un servidor de fitxers Windows més.

| Component | Funció                                                                 |
| --------- | ---------------------------------------------------------------------- |
| `smbd`    | Servei principal: carpetes i impressores compartides, autenticació (ports 445 i 139) |
| `nmbd`    | Noms NetBIOS i exploració de la xarxa antiga (ports 137 i 138 UDP)     |
| `winbindd`| Integració amb dominis Windows (fitxer 06)                             |

### Instal·lació

```bash
sudo apt install samba
systemctl status smbd nmbd
sudo ss -tlnp | grep smbd                 # ports 139 i 445
```

Si el tallafoc està actiu: `sudo ufw allow 445/tcp` (amb SMB2 o posterior, n'hi ha prou amb el port 445).

### El Fitxer `smb.conf`

La configuració és a `/etc/samba/smb.conf`, dividida en seccions:

| Secció      | Contingut                                                         |
| ----------- | ----------------------------------------------------------------- |
| `[global]`  | Configuració del servidor                                         |
| `[homes]`   | Directori personal de cada usuari (recurs especial)               |
| `[printers]`| Impressores compartides (Tema 5)                                  |
| `[nom]`     | Cada carpeta compartida (*recurs* o *share*), fitxer 03           |

**Paràmetres principals de `[global]`:**

```ini
[global]
   workgroup = EDUCEM
   server string = Servidor de fitxers srv0374
   security = user
   map to guest = never
   interfaces = lo enp0s3 enp0s8
   bind interfaces only = yes
   server min protocol = SMB2
   log file = /var/log/samba/log.%m
   max log size = 1000
```

| Paràmetre                  | Significat                                                           |
| -------------------------- | -------------------------------------------------------------------- |
| `workgroup`                | Grup de treball (o nom NetBIOS del domini) que veuen els equips Windows |
| `server string`            | Descripció del servidor                                              |
| `security = user`          | Cada usuari s'autentica amb el seu nom i contrasenya de Samba        |
| `map to guest = never`     | No converteix els accessos incorrectes en accessos de convidat       |
| `interfaces` + `bind interfaces only` | Interfícies on escolta Samba                              |
| `server min protocol = SMB2` | No accepta SMB1, que és insegur i obsolet                          |
| `log file`                 | Un registre per a cada equip client (`%m` = nom de l'equip)          |

### Comprovar la Configuració i Aplicar-la

```bash
testparm                                  # comprova la sintaxi i mostra la configuració efectiva
sudo systemctl restart smbd
smbstatus                                 # connexions i fitxers oberts ara
```

`testparm` indica els errors i mostra només els paràmetres que no tenen el valor per defecte. Cal executar-lo **sempre** abans de reiniciar el servei.

### Resolució de Problemes

| Problema                                                  | Causa                                                    | Solució                                                |
| --------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------ |
| `testparm` mostra `Unknown parameter encountered`          | Paràmetre mal escrit                                     | Corregir-lo a `smb.conf`                               |
| Els clients no hi arriben                                  | Tallafoc o `interfaces` no inclou la interfície de la xarxa del client | `sudo ufw allow 445/tcp`; revisar `interfaces` |
| Windows no veu el servidor a *Xarxa*                       | Windows 10/11 ja no fa servir l'exploració NetBIOS       | Accedir-hi directament per l'adreça: `\\10.0.1.37`     |
| Els canvis no tenen efecte                                 | El servei no s'ha reiniciat                              | `sudo systemctl restart smbd`                          |
