## 03. Samba: Recursos Compartits, Usuaris i Permisos

### Usuaris de Samba

Samba fa servir els **usuaris del sistema Linux**, però té la seva pròpia base de dades de contrasenyes (el format de contrasenya de Windows és diferent del de Linux). Per tant, cada usuari ha d'existir al sistema i, a més, tenir una contrasenya de Samba:

```bash
sudo useradd -m -s /usr/sbin/nologin anna          # usuari del sistema sense accés per terminal
sudo smbpasswd -a anna                              # afegir-lo a Samba amb la seva contrasenya
sudo pdbedit -L                                      # llistar els usuaris de Samba
```

| Ordre                       | Funció                                    |
| --------------------------- | ----------------------------------------- |
| `smbpasswd -a usuari`       | Afegir un usuari a Samba                  |
| `smbpasswd usuari`          | Canviar-ne la contrasenya                 |
| `smbpasswd -d` / `-e usuari`| Desactivar / activar un usuari            |
| `smbpasswd -x usuari`       | Esborrar-lo de Samba                      |
| `pdbedit -L -v`             | Llistar-los amb tots els detalls          |

### Definir un Recurs Compartit

Exemple: una carpeta per al departament d'administració, on tots els membres del grup poden llegir i escriure.

```bash
sudo groupadd administracio
sudo usermod -aG administracio anna
sudo mkdir -p /srv/samba/administracio
sudo chgrp administracio /srv/samba/administracio
sudo chmod 2770 /srv/samba/administracio      # 2 (setgid): els fitxers nous hereten el grup
```

```ini
[administracio]
   comment = Documents del departament d'administració
   path = /srv/samba/administracio
   valid users = @administracio
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 2770
```

| Paràmetre                   | Significat                                                             |
| --------------------------- | ---------------------------------------------------------------------- |
| `path`                      | Directori del servidor que es comparteix                               |
| `valid users`               | Usuaris (o grups amb `@`) que hi poden accedir                         |
| `read only = no`            | Permet escriure (equival a `writable = yes`)                           |
| `write list`                | Usuaris que hi poden escriure encara que el recurs sigui de només lectura |
| `browseable`                | Si apareix a la llista de recursos del servidor                        |
| `create mask` / `directory mask` | Permisos màxims dels fitxers i directoris nous                    |
| `force group`               | Grup que s'assigna a tot el que es crea                                |
| `guest ok`                  | Accés sense contrasenya (no recomanat; Windows 10/11 el bloqueja)      |

**Exemple d'un recurs de només lectura amb editors:**

```ini
[documentacio]
   path = /srv/samba/documentacio
   valid users = @alumnes, @professors
   read only = yes
   write list = @professors
```

### La Doble Capa de Permisos

Un usuari només pot fer una operació si **la permeten Samba i el sistema de fitxers Linux alhora**:

```
Petició de l'usuari ──► Permisos de Samba (smb.conf) ──► Permisos de Linux (propietari, grup, chmod) ──► Fitxer
```

Si `smb.conf` diu `read only = no` però el directori Linux és `755` i de root, l'usuari no hi podrà escriure. Comproveu sempre totes dues capes (`testparm` i `ls -ld`).

### Directoris Personals: `[homes]`

```ini
[homes]
   comment = Directori personal
   browseable = no
   read only = no
   valid users = %S
   create mask = 0600
   directory mask = 0700
```

Cada usuari veu un recurs amb el seu nom (`\\srv0374\anna`) que correspon al seu directori personal. `%S` és el nom del recurs, és a dir, el de l'usuari: ningú no pot entrar al directori d'un altre.

### Paperera de Reciclatge (Opcional)

```ini
   vfs objects = recycle
   recycle:repository = .paperera/%U
   recycle:keeptree = yes
```

Els fitxers esborrats des dels clients es mouen a `.paperera/usuari` en lloc d'esborrar-se.

### Resolució de Problemes

| Problema                                               | Causa                                                    | Solució                                                |
| ------------------------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------ |
| `NT_STATUS_LOGON_FAILURE`                              | L'usuari no és a Samba o la contrasenya és incorrecta    | `sudo smbpasswd -a usuari`                             |
| `NT_STATUS_ACCESS_DENIED` en entrar al recurs          | L'usuari no és a `valid users`                           | Revisar el grup (`id usuari`) i `smb.conf`             |
| Es pot entrar però no escriure                         | `read only = yes` o els permisos de Linux no ho permeten | Revisar les dues capes de permisos                     |
| Els fitxers creats per un usuari no els pot modificar un altre del grup | Els fitxers no hereten el grup o la màscara és massa restrictiva | `chmod 2770` al directori i `create mask = 0660` |
