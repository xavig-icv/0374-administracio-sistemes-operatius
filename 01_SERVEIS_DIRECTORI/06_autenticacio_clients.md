## 06. Autenticació de Clients amb LDAP

Un servidor LDAP per si sol no autentica usuaris als sistemes Linux. Cal configurar els clients perquè consultin el directori LDAP en el moment del login. Això es fa mitjançant dos serveis: **NSS** (Name Service Switch) i **PAM** (Pluggable Authentication Modules).

```
USUARI intenta iniciar sessió
   ↓
PAM (Pluggable Authentication Modules)
   → Verifica les credencials consultant LDAP
   ↓
NSS (Name Service Switch)
   → Obté la informació de l'usuari (home, shell, UID...) de LDAP
   ↓
ACCÉS CONCEDIT (o denegat)
```

### Instal·lar els Paquets al Client (Ubuntu Client)

```bash
sudo apt update

# Paquets principals per autenticació LDAP
sudo apt install libnss-ldapd libpam-ldapd ldap-utils nslcd

# Durant la instal·lació demanarà:
#   - URI del servidor LDAP: ldap://192.168.1.1
#   - Base DN: dc=educem,dc=cat
#   - Serveis a configurar per NSS: passwd, group, shadow
```

El paquet `nslcd` (Name Service LDAP Connection Daemon) és el dimoni que gestiona les connexions entre el client i el servidor LDAP per a NSS i PAM.

### Configurar nslcd

El fitxer `/etc/nslcd.conf` defineix com `nslcd` es connecta al servidor LDAP:

```bash
sudo nano /etc/nslcd.conf
```

**Contingut:**

```bash
# /etc/nslcd.conf
# Configuració del daemon de connexió LDAP per NSS

# Usuari i grup del dimoni
uid nslcd
gid nslcd

# Servidor LDAP
uri ldap://192.168.1.1

# Base DN
base dc=educem,dc=cat

# Bases específiques per cada tipus d'informació (opcional)
base passwd ou=usuaris,dc=educem,dc=cat
base group  ou=grups,dc=educem,dc=cat
base shadow ou=usuaris,dc=educem,dc=cat

# Credencials per connectar. Mentre el servidor permeti consultes anònimes no calen.
# Quan les ACL les prohibeixin (fitxer 07), cal un compte de servei de NOMÉS LECTURA
# (mai el compte d'administrador):
# binddn cn=servei-lectura,dc=educem,dc=cat
# bindpw contrasenya_del_servei

# Versió del protocol
ldap_version 3

# Temps d'espera de connexió
bind_timelimit 30
timelimit 30
```

Amb `uri ldap://` la connexió no va xifrada. La configuració de TLS (`tls_cacertfile` i `tls_reqcert`) es fa al fitxer 09.

```bash
# Reiniciar nslcd per aplicar canvis
sudo systemctl restart nslcd
sudo systemctl enable nslcd
sudo systemctl status nslcd
```

### Configurar NSS (Name Service Switch)

NSS determina l'ordre en que el sistema busca informació d'usuaris: primer als fitxers locals (`/etc/passwd`), i si no els troba, al servidor LDAP.

```bash
sudo nano /etc/nsswitch.conf
```

**Modificar les línies de passwd, group i shadow:**

```bash
# /etc/nsswitch.conf
# Ordre de resolució: primer fitxers locals, després LDAP

passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files ldap
gshadow:        files

# La resta queden igual
hosts:          files dns
networks:       files
protocols:      db files
services:       db files
ethers:         db files
rpc:            db files
netgroup:       nis
```

**Verificar que NSS troba els usuaris LDAP:**

```bash
# Llistar tots els usuaris (locals + LDAP)
getent passwd

# Cercar un usuari LDAP específic
getent passwd pep

# Llistar tots els grups
getent group

# Ha de mostrar l'usuari pep amb les seves dades:
# pep:x:10001:10001:Pep Guardiola:/home/pep:/bin/bash
```

### Configurar PAM (Pluggable Authentication Modules)

PAM gestiona l'autenticació pròpiament dita (verificació de contrasenya). El paquet `libpam-ldapd` configura automàticament els fitxers PAM, però es pot revisar la configuració:

```bash
# Verificar que PAM té LDAP configurat
grep -r "ldap\|pam_ldapd" /etc/pam.d/
```

**Habilitar la creació automàtica del directori home:**

Quan un usuari LDAP inicia sessió per primera vegada, el seu directori `/home` no existeix al client. Cal habilitar `pam_mkhomedir`:

Utilitzar la comanda:

```bash
sudo pam-auth-update --enable mkhomedir
```

També podries editar el sgüent fitxer i afegir la configuració al final:

```bash
sudo nano /etc/pam.d/common-session
```

```
# Crear el directori home automàticament si no existeix
session required  pam_mkhomedir.so skel=/etc/skel/ umask=0022
```

### Provar l'Autenticació

```bash
# Verificar que al client es pot autenticar un usuari LDAP (sense iniciar sessió)
su - pep

# Comprovar la identitat
whoami
id pep

# Verificar d'on ve la informació de l'usuari
getent passwd pep

# Provar el login complet (en un altre terminal i via SSH)
ssh pep@localhost
```

### Troubleshooting

```bash
# Veure logs de nslcd
sudo journalctl -u nslcd -f

# Veure logs d'autenticació PAM
sudo tail -f /var/log/auth.log

# Verificar connectivitat amb el servidor LDAP
ldapsearch -x -H ldap://192.168.1.1 -b "dc=educem,dc=cat" "(uid=pep)" cn uid

# Depurar nslcd en mode verbose
sudo systemctl stop nslcd
sudo nslcd -d   # Executa en primer pla amb sortida de debug

# Verificar que PAM troba el mòdul LDAP
pamtester login pep authenticate
```

**Errors comuns:**

| Error                              | Causa probable           | Solució                        |
| ---------------------------------- | ------------------------ | ------------------------------ |
| `getent passwd pep` no retorna res | nslcd no connecta        | Revisar URI i firewall         |
| Login falla però `getent` funciona | PAM no configurat        | Revisar `/etc/pam.d/`          |
| Home directory no es crea          | pam_mkhomedir no actiu   | Afegir a `common-session`      |
| Connexió lenta                     | DNS no resol el servidor | Usar IP en lloc de hostname    |
| `getent` deixa de funcionar després d'aplicar ACL | Les ACL no permeten consultes anònimes | `binddn` i `bindpw` amb un compte de servei (fitxer 07) |
| `TLS handshake failure`            | Certificat no vàlid o sense la IP al SAN | Revisar `tls_cacertfile` i el certificat (fitxer 09) |
