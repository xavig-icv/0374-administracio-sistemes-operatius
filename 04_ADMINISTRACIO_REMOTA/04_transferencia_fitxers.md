## 04. Transferència de Fitxers per SSH

La mateixa connexió SSH serveix per copiar fitxers de manera xifrada, amb les mateixes claus i la mateixa configuració del fitxer `~/.ssh/config`. El servei SFTP per a usuaris (gàbies `chroot`, usuaris només SFTP) es treballa al mòdul 0375, al tema de transferència de fitxers.

| Eina      | Ús                                                                 |
| --------- | ------------------------------------------------------------------ |
| `scp`     | Copiar fitxers o directoris en una sola ordre                      |
| `sftp`    | Sessió interactiva per navegar i copiar fitxers                    |
| `rsync`   | Sincronitzar directoris copiant només les diferències              |
| `sshfs`   | Muntar un directori remot com si fos local                         |
| Gràfiques | MobaXterm (panell SFTP), WinSCP, FileZilla (protocol SFTP)         |

### `scp`

```bash
scp informe.txt srv0374:/tmp/                    # local → remot
scp srv0374:/var/log/syslog .                    # remot → local (directori actual)
scp -r scripts/ srv0374:~/                       # directori sencer (-r)
scp -P 2222 informe.txt xavi@10.0.1.37:/tmp/     # port diferent (-P majúscula)
scp -J srv0374 informe.txt xavi@192.168.1.2:/tmp/   # passant pel 0374 (fitxer 05)
```

Les versions actuals d'OpenSSH fan servir internament el protocol SFTP per a `scp`.

### `sftp`

```
$ sftp srv0374
sftp> pwd
Remote working directory: /home/xavi
sftp> ls
scripts  informes
sftp> get informes/informe-2026-11-05.txt
sftp> put copia.tar.gz /tmp/
sftp> bye
```

| Ordre               | Funció                                    |
| ------------------- | ----------------------------------------- |
| `ls`, `cd`, `pwd`   | Navegar pel servidor                      |
| `lls`, `lcd`, `lpwd`| Navegar per l'equip local (`l` = *local*) |
| `get fitxer`        | Descarregar                               |
| `put fitxer`        | Pujar                                     |
| `get -r` / `put -r` | Directoris sencers                        |
| `mkdir`, `rm`, `rename` | Gestionar fitxers remots              |
| `bye`               | Sortir                                    |

### `rsync`

`rsync` és la millor opció per a còpies grans o repetides: després de la primera vegada, només envia el que ha canviat (Tema 3, fitxer 06).

```bash
rsync -avz scripts/ srv0374:~/scripts/                  # sincronitzar un directori
rsync -avz --delete --dry-run web/ srv0375:/var/www/    # simulació d'una còpia exacta
rsync -avz -e "ssh -p 2222" fitxers/ xavi@10.0.1.37:~/  # port diferent
```

### `sshfs`: Muntar un Directori Remot

```bash
sudo apt install sshfs
mkdir -p ~/remot
sshfs srv0374:/var/log ~/remot          # els registres del servidor, com si fossin locals
ls ~/remot
fusermount -u ~/remot                   # desmuntar
```

És pràctic per editar fitxers remots amb un editor local, però per a còpies grans és més eficient `rsync`.

### Resolució de Problemes

| Problema                                                    | Causa                                                    | Solució                                              |
| ----------------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| `scp: dest open "/var/www/fitxer": Permission denied`       | L'usuari no pot escriure al directori de destí           | Copiar a `/tmp` i moure-ho amb `sudo`, o ajustar els permisos |
| `scp -p 2222` no funciona                                   | A `scp` el port és `-P` (majúscula); `-p` conserva les dates | `scp -P 2222`                                     |
| `rsync: command not found` al servidor                      | `rsync` ha d'estar instal·lat als dos equips             | `sudo apt install rsync`                             |
| `sshfs` diu `Transport endpoint is not connected`           | La connexió s'ha tallat amb el directori muntat          | `fusermount -u ~/remot` i tornar a muntar            |
