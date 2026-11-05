## 04. Compartir Impressores amb Clients Linux i Windows

### Compartir una Impressora al Servidor

Cal que CUPS comparteixi impressores (`cupsctl --share-printers`, fitxer 02) i que la impressora concreta estigui marcada com a compartida:

```bash
sudo lpadmin -p PDF -o printer-is-shared=true
lpoptions -p PDF | grep -o "printer-is-shared=[a-z]*"     # comprovar-ho
```

Cada impressora compartida té una adreça IPP:

```
ipp://192.168.1.1:631/printers/PDF           (xarxa interna)
http://10.0.1.37:631/printers/PDF            (xarxa del centre, format que fa servir Windows)
```

### Clients Linux

**Per ordres:** el client crea una cua local que envia els treballs a la cua compartida del servidor. Una cua de CUPS compartida es comporta com una impressora sense controlador, per això es fa servir `-m everywhere`:

```bash
sudo apt install cups-client                 # al client Ubuntu Desktop ja hi és
sudo lpadmin -p PDF-srv0374 -E -v ipp://192.168.1.1:631/printers/PDF -m everywhere
lp -d PDF-srv0374 /etc/hostname
```

**Escriptori d'Ubuntu:** *Configuració* → *Impressores* → *Afegeix una impressora* i escriure l'adreça del servidor (`192.168.1.1`).

El treball arriba al servidor amb l'usuari del client (`alumne`). Com que aquest usuari no existeix al servidor, el PDF es desa a `/var/spool/cups-pdf/ANONYMOUS/` (fitxer 02).

### Clients Windows

*Configuració* → *Bluetooth i dispositius* → *Impressores i escàners* → *Afegeix un dispositiu* → *La impressora que vull no apareix a la llista* → *Selecciona una impressora compartida pel nom*:

```
http://10.0.1.37:631/printers/PDF
```

Windows fa servir el seu controlador genèric d'IPP (*Microsoft IPP Class Driver*). Per comprovar-ho, imprimiu una pàgina de prova des de les propietats de la impressora.

### Compartir per Samba (Opcional)

Si el servidor ja té Samba (Tema 6), les impressores de CUPS també es poden compartir per SMB, com a `\\srv0374\PDF`:

```ini
[global]
   printing = cups
   printcap name = cups
   load printers = yes

[printers]
   comment = Impressores del servidor
   path = /var/spool/samba
   printable = yes
   browseable = yes
   guest ok = no
```

Actualment es recomana IPP: Windows el fa servir sense controladors addicionals i no depèn de Samba.

### Resolució de Problemes

| Problema                                                      | Causa                                                      | Solució                                                     |
| ------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------- |
| El client no troba la impressora                              | La impressora o el servidor no comparteixen                | `cupsctl --share-printers` i `printer-is-shared=true`       |
| `Unable to connect` des del client                            | Tallafoc o CUPS només escolta a localhost                  | `ufw allow 631/tcp` i `cupsctl --remote-any`                |
| Windows diu que no pot connectar-se a la impressora           | Adreça incorrecta (cal `http://` i el nom exacte de la cua) | `http://IP:631/printers/NOM`; el nom distingeix majúscules |
| Els treballs arriben al servidor però no es veu el PDF        | L'usuari del client no existeix al servidor                | Mirar a `/var/spool/cups-pdf/ANONYMOUS/`                    |
