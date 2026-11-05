## 07. Control d'Accés: ACLs

Per defecte, OpenLDAP permet que qualsevol usuari llegeixi pàcticament tot el directori, incloent atributs sensibles. Les ACLs (Access Control Lists) defineixen els permisos sobre cada part del directori: llegir, escriure, autenticar-se, etc.

Una mala configuració d'ACLs és un dels errors de seguretat més habituals en instal·lacions OpenLDAP. Un servei extern compromès podria llegir totes les contrasenyes en hash del directori si no es restringeix l'accés.

**Estructura d'una ACL:**

```
access to <A QUÈ>
    by <QUI> <PERMÍS>
    by <QUI> <PERMÍS>
```

- **A QUÈ**: A quina part del directori (atribut, entrada, subàrbre) s'aplica la regla.
- **QUI**: Qui fa la petició (usuari anònim, l'usuari propi, un grup, tothom, etc.).
- **PERMÍS**: Nivell d'accés concedit.

**Nivells de permís (de menor a major):**

| Permís     | Descripció                                                                              |
| ---------- | --------------------------------------------------------------------------------------- |
| `none`     | Cap accés (l'usuari no pot veure res)                                                   |
| `disclose` | Pot saber que l'entrada existeix (no pot llegir ni modificar atributs)                  |
| `auth`     | Només per autenticació (bind), no pot llegir cap atribut                                |
| `compare`  | Pot comparar valors (ldap_compare), comprovar si la contrasenya concideix               |
| `read`     | Pot llegir atributs i entrades però no modificar (apps consulta de dades)               |
| `write`    | Pot modificar atributs o afegir-ne de nous (no pot eliminar l'entrada)                  |
| `manage`   | Control total (com a administrador) pot canviar ACL, modificar o eliminar entrada, etc. |

**Qui pot ser el subjecte (`by <QUI>`):**

| Subjecte           | Descripció                                                       |
| ------------------ | ---------------------------------------------------------------- |
| `*`                | Tothom (anònim + autenticat)                                     |
| `anonymous`        | Usuaris no autenticats                                           |
| `users`            | Qualsevol usuari autenticat                                      |
| `self`             | El propi usuari autenticat (sobre la seva entrada)               |
| `dn="..."`         | Un DN concret amb permisos específics                            |
| `group="..."`      | Aplicar permisos a membres d'un grup                             |
| `dn.subtree="..."` | Aplicar permisos a tots els DN dins d'un subàrbre (OU múltiples) |

### On es defineixen les ACLs

Les ACLs es configuren a la base de dades via OLC (Online Configuration), modificant l'atribut `olcAccess` de la base de dades:

```bash
# Veure les ACLs actuals
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "olcDatabase={1}mdb,cn=config" olcAccess
```

### ACLs Bàsiques Recomanades

Aquesta configuració és un punt de partida segur per a un entorn de producció:

```bash
nano acls-basiques.ldif
```

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
# REGLA 1: la contrasenya només la pot canviar el propi usuari i l'administrador.
# Els anònims només la poden fer servir per autenticar-se. Ningú la pot llegir.
olcAccess: to attrs=userPassword
  by self write
  by dn="cn=admin,dc=educem,dc=cat" write
  by anonymous auth
  by * none
# REGLA 2: dades de caducitat de la contrasenya (shadowAccount)
olcAccess: to attrs=shadowLastChange,shadowExpire
  by self write
  by dn="cn=admin,dc=educem,dc=cat" write
  by users read
  by * none
# REGLA 3: la resta del directori la poden llegir els usuaris autenticats
olcAccess: to *
  by dn="cn=admin,dc=educem,dc=cat" write
  by users read
  by * none
```

**Atenció al format LDIF:** una línia en blanc indica el final d'una entrada. Dins d'un canvi (`changetype: modify`) no hi pot haver línies en blanc, i els comentaris han d'ocupar una línia sencera que comenci per `#`. Les línies que comencen per un espai són la continuació de la línia anterior.

**Per què `self` no té `write` a la regla 3?** Si cada usuari pogués modificar la seva pròpia entrada, podria canviar el seu `uidNumber` a 0 i convertir-se en root als clients Linux. Els usuaris només poden canviar la seva contrasenya (regla 1).

```bash
# Aplicar les ACLs
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f acls-basiques.ldif

# Verificar que s'han aplicat
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "olcDatabase={1}mdb,cn=config" olcAccess
```

**Ordre important:** OpenLDAP avalua les ACLs de dalt a baix i s'atura a la primera que coincideix. Les regles més específiques han d'anar **sempre** abans que les generals.

### ACLs per Casos d'Ús Habituals

**Compte de servei de només lectura** (per a aplicacions que consulten el directori):

És una pràctica recomanada crear un compte específic per a cada servei extern (sistema d'incidències, correu, VPN, etc.) amb permisos mínims, en lloc d'utilitzar el compte d'administrador.

```ldif
# Primer creem el compte del servei.
# userPassword: enganxeu-hi el hash que retorna slappasswd (fitxer 05)
dn: cn=glpi-incidencies,dc=educem,dc=cat
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: glpi-incidencies
description: Compte de servei per al servei web glpi
userPassword: {SSHA}z6r+3GqL3gk0y+qz+2K1QmGZk6iXQnLr
```

```ldif
# ACL que permet al servei glpi llegir uid, mail i cn
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: to attrs=userPassword
  by self write
  by dn="cn=admin,dc=educem,dc=cat" write
  by anonymous auth
  by * none
olcAccess: to attrs=uid,mail,cn
  by dn="cn=glpi-incidencies,dc=educem,dc=cat" read
  by dn="cn=admin,dc=educem,dc=cat" write
  by users read
  by * none
olcAccess: to *
  by dn="cn=admin,dc=educem,dc=cat" write
  by users read
  by * none
```

El servei GLPI no necessita cap permís sobre `userPassword`: per validar l'inici de sessió d'un usuari, l'aplicació s'autentica amb el DN i la contrasenya de l'usuari, i això ho permet `by anonymous auth`.

### Clients Linux amb ACL Restrictives

Amb la regla `to * ... by * none`, les consultes anònimes no poden llegir res. Els clients Linux configurats al fitxer 06 consulten el directori de manera anònima, així que **deixen de trobar els usuaris** (`getent passwd pep` no retorna res). La solució és que `nslcd` s'autentiqui amb un compte de servei de només lectura:

**1.** Crear el compte `cn=servei-lectura,dc=educem,dc=cat` igual que el de l'exemple anterior (amb el seu hash).

**2.** Afegir-lo al fitxer `/etc/nslcd.conf` del client:

```
binddn cn=servei-lectura,dc=educem,dc=cat
bindpw contrasenya_del_servei
```

**3.** Reiniciar `nslcd` i comprovar que torna a funcionar:

```bash
sudo systemctl restart nslcd
getent passwd pep
```

El compte de servei és un usuari autenticat: amb `by users read` pot llegir el directori, però no les contrasenyes. Com que `/etc/nslcd.conf` conté la seva contrasenya, només el poden llegir root i el grup `nslcd` (permisos `640`).

**Permetre que els usuaris canviïn la seva pròpia contrasenya:**

Ja inclòs a la regla bàsica amb `by self write` a `userPassword`. Es pot verificar:

```bash
# L'usuari pep canvia la seva pròpia contrasenya
sudo ldappasswd -x -H ldap://192.168.1.1 -D "uid=pep,ou=usuaris,dc=educem,dc=cat" -W -S
```

**Restringir accés per IP o subxarxa** (via `peername`):

```ldif
olcAccess: to *
  by peername.ip=192.168.1.0%255.255.255.0 read
  by * none
```

### Verificar les ACLs en Acció

```bash
# Provar si un usuari anònim pot llegir contrasenyes (no hauria de poder)
sudo ldapsearch -x -H ldap://192.168.1.1 -b "ou=usuaris,dc=educem,dc=cat" "(uid=pep)" userPassword

# Provar si l'usuari pep pot llegir el seu propi perfil
ldapsearch -x -H ldap://192.168.1.1 -D "uid=pep,ou=usuaris,dc=educem,dc=cat" -W -b "uid=pep,ou=usuaris,dc=educem,dc=cat"

# Provar si el pep pot llegir les dades d'un altre usuari
ldapsearch -x -H ldap://192.168.1.1 -D "uid=pep,ou=usuaris,dc=educem,dc=cat" -W -b "uid=julia,ou=usuaris,dc=educem,dc=cat"
```
