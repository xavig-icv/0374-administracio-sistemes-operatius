## 05. NFS: Compartir Fitxers entre Sistemes Linux

**NFS** (*Network File System*) és el protocol natiu dels sistemes UNIX i Linux per compartir directoris. És més senzill i ràpid que SMB entre equips Linux, però no té usuaris propis: els permisos es basen en els **números d'usuari i de grup (UID i GID)**. Per això funciona millor quan tots els equips fan servir els mateixos usuaris, per exemple els del directori LDAP (Tema 1).

### Servidor NFS (srv0374)

```bash
sudo apt install nfs-kernel-server
sudo mkdir -p /srv/nfs/dades
sudo nano /etc/exports
```

```
# directori        clients autoritzats (opcions)
/srv/nfs/dades     192.168.1.0/24(rw,sync,no_subtree_check)
```

```bash
sudo exportfs -ra                 # aplicar /etc/exports
sudo exportfs -v                  # directoris exportats i opcions
```

| Opció               | Significat                                                                        |
| ------------------- | --------------------------------------------------------------------------------- |
| `rw` / `ro`         | Lectura i escriptura / només lectura                                              |
| `sync`              | Confirma les escriptures quan ja són al disc (més segur que `async`)              |
| `no_subtree_check`  | Desactiva una comprovació que dona problemes quan es canvien noms de fitxers      |
| `root_squash`       | (Per defecte) El root del client es tracta com l'usuari `nobody`: no és root al servidor |
| `no_root_squash`    | El root del client és root al servidor. **Perillós**: només per a casos molt concrets |

**Tallafoc.** El 0374 té `ufw` actiu ([escenari](../00_RECURSOS/01_escenari_laboratori.md)). NFS fa servir el port 2049. L'ordre `showmount` del client necessita, a més, `rpcbind` (port 111) i el servei `mountd`, que per defecte canvia de port a cada arrencada. Fixeu-lo al 20048: a la secció `[mountd]` de `/etc/nfs.conf`, descomenteu la línia `port` i deixeu-la així:

```
[mountd]
port=20048
```

```bash
sudo systemctl restart nfs-server
rpcinfo -p | grep mountd                  # mountd escolta al port 20048
sudo ufw allow from 192.168.1.0/24 to any port 111,2049,20048 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 111,20048 proto udp
```

### Client NFS (client Ubuntu)

```bash
sudo apt install nfs-common
showmount -e 192.168.1.1                          # directoris que exporta el servidor
sudo mkdir -p /mnt/dades
sudo mount -t nfs 192.168.1.1:/srv/nfs/dades /mnt/dades
df -h /mnt/dades
```

Per muntar-lo en arrencar, a `/etc/fstab`:

```
192.168.1.1:/srv/nfs/dades  /mnt/dades  nfs  defaults,_netdev  0  0
```

### Directoris Personals Compartits per als Usuaris LDAP

La integració més útil: els usuaris del directori LDAP tenen el **mateix directori personal a tots els equips**. En qualsevol client on iniciïn sessió, troben els seus fitxers.

**Al servidor:** exportar el directori on hi haurà els directoris personals.

```bash
sudo mkdir -p /srv/nfs/home
echo "/srv/nfs/home 192.168.1.0/24(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -ra
# Crear el directori de cada usuari LDAP amb el seu UID i GID (Tema 1, fitxer 05)
sudo mkdir /srv/nfs/home/pep
sudo chown 10001:10001 /srv/nfs/home/pep
sudo chmod 700 /srv/nfs/home/pep
```

**Al client:** muntar automàticament el directori de cada usuari quan el necessita, amb `autofs`.

```bash
sudo apt install autofs
echo "/home/ldap  /etc/auto.home" | sudo tee -a /etc/auto.master
echo "*  -fstype=nfs,rw  192.168.1.1:/srv/nfs/home/&" | sudo tee /etc/auto.home
sudo systemctl restart autofs
```

`*` i `&` fan que, quan algú accedeix a `/home/ldap/pep`, `autofs` munti `192.168.1.1:/srv/nfs/home/pep`. Cal que el `homeDirectory` dels usuaris al directori LDAP sigui `/home/ldap/<usuari>` (o adaptar les rutes).

### Seguretat de NFS

- NFS **no xifra** les dades i confia en el client per identificar l'usuari (UID). Un equip que no sigui de confiança podria fer-se passar per qualsevol usuari.
- Exporteu només a les xarxes i equips que ho necessiten, mai a tothom (`*`).
- En entorns que necessiten més seguretat s'utilitza NFS amb Kerberos (`sec=krb5p`), que autentica els usuaris i xifra el trànsit.

### Resolució de Problemes

| Problema                                                    | Causa                                                    | Solució                                                     |
| ----------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| `mount.nfs: access denied by server`                        | El client no és a la llista de `/etc/exports`            | Revisar la xarxa autoritzada i `sudo exportfs -ra`          |
| Els fitxers apareixen amb un propietari estrany             | L'UID del servidor no coincideix amb el del client       | Fer servir els mateixos usuaris (LDAP) als dos equips       |
| Root del client no pot escriure al directori muntat         | `root_squash` (comportament per defecte, correcte)       | Escriure-hi amb l'usuari propietari                         |
| `/home/ldap/pep` no apareix                                 | `autofs` munta sota demanda: el directori no existeix fins que s'hi accedeix | `cd /home/ldap/pep`; revisar `journalctl -u autofs` |
| `showmount` no respon, però el muntatge funciona            | `mountd` sense port fix, o el tallafoc no deixa passar el 111 i el 20048 | Apartat del tallafoc; `rpcinfo -p 192.168.1.1` des del client |
| `mount` es queda esperant                                   | El tallafoc del 0374 no deixa passar el port 2049        | `sudo ufw status` al servidor                               |
