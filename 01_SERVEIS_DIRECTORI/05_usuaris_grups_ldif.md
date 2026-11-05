## 05. Gestió d'Usuaris i Grups amb LDIF

La gestió d'usuaris i grups és l'acció més habitual que es realitza a un servidor LDAP. En aquest apartat veurem com crear, modificar i eliminar usuaris i grups, com gestionar contrasenyes i com realitzar operacions en bloc.

### Crear Usuaris

Cada usuari, al directori LDAP, ha de tenir com a mínim les classes d'objecte `inetOrgPerson` i `posixAccount` (per poder integrar-se amb els sistemes Linux).

**Fitxer LDIF per crear un usuari:**

**Abans generem una contrasenya per l'usuari**
**Generar un hash de contrasenya manualment:**

```bash
# Generar el hash SSHA de la contrasenya (sense -s, la demana dues vegades sense mostrar-la)
slappasswd -h {SSHA}
New password:
Re-enter new password:
{SSHA}z6r+3GqL3gk0y+qz+2K1QmGZk6iXQnLr
```

El resultat és el que es posa a l'atribut `userPassword`. Cada execució dona un hash diferent (inclou un valor aleatori, la *sal*), encara que la contrasenya sigui la mateixa. Amb `-s "contrasenya"` no la demana, però la contrasenya queda a l'historial del terminal.

```bash
nano usuari-pep.ldif
```

```ldif
dn: uid=pep,ou=usuaris,dc=educem,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Pep Guardiola
sn: Guardiola
givenName: Pep
uid: pep
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/pep
loginShell: /bin/bash
mail: pep@educem.cat
userPassword: {SSHA}z6r+3GqL3gk0y+qz+2K1QmGZk6iXQnLr
```

```bash
# Afegir l'usuari al directori
sudo ldapadd -x -D "cn=admin,dc=educem,dc=cat" -W -f usuari-pep.ldif
```

**Crear múltiples usuaris en un sol fitxer LDIF** (separats per un salt de línia i una línia en blanc):

```ldif
dn: uid=julia,ou=usuaris,dc=educem,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Júlia López
sn: López
givenName: Júlia
uid: julia
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/julia
loginShell: /bin/bash
mail: julia@educem.cat

dn: uid=pere,ou=usuaris,dc=educem,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Pere Mas
sn: Mas
givenName: Pere
uid: pere
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/pere
loginShell: /bin/bash
mail: pere@educem.cat
```

Aquests dos usuaris no tenen `userPassword`: no podran iniciar sessió fins que se'ls assigni una contrasenya amb `ldappasswd` (apartat *Gestió de Contrasenyes*). **No escriviu mai `{SSHA}` seguit d'una contrasenya en text pla**: el servidor l'interpretaria com un hash i la contrasenya no funcionaria.

**Nota sobre uidNumber i gidNumber:**
Han de ser únics al directori i no coincidir amb els dels comptes locals (a Ubuntu, els usuaris locals comencen a 1000). Per això els usuaris LDAP comencen a 10000. El `gidNumber` de cada usuari és el seu **grup principal**, que ha d'existir al directori (apartat següent).

### Crear Grups

```bash
nano grups.ldif
```

Cada usuari té un **grup personal** amb el mateix nom i el mateix número que l'usuari (com fa Ubuntu amb els comptes locals). Els grups de treball fan servir el rang 20000-29999 perquè no coincideixin amb els dels usuaris.

```ldif
# Grups personals (grup principal de cada usuari)
dn: cn=pep,ou=grups,dc=educem,dc=cat
objectClass: posixGroup
cn: pep
gidNumber: 10001

dn: cn=julia,ou=grups,dc=educem,dc=cat
objectClass: posixGroup
cn: julia
gidNumber: 10002

dn: cn=pere,ou=grups,dc=educem,dc=cat
objectClass: posixGroup
cn: pere
gidNumber: 10003

# Grup d'Administradors
dn: cn=admins,ou=grups,dc=educem,dc=cat
objectClass: posixGroup
cn: admins
gidNumber: 20001
memberUid: pep
memberUid: julia

# Grup de Desenvolupadors
dn: cn=developers,ou=grups,dc=educem,dc=cat
objectClass: posixGroup
cn: developers
gidNumber: 20002
memberUid: julia
memberUid: pere
```

Si el grup principal d'un usuari no existeix, en un client `id pep` mostra l'avís `cannot find name for group ID 10001`.

```bash
sudo ldapadd -x -D "cn=admin,dc=educem,dc=cat" -W -f grups.ldif
```

### Gestió de Contrasenyes

**Establir o canviar la contrasenya d'un usuari:**

```bash
# Canviar contrasenya (demanarà la contrasenya nova interactivament)
sudo ldappasswd -x -D "cn=admin,dc=educem,dc=cat" -W -S "uid=pep,ou=usuaris,dc=educem,dc=cat"

# -S → Demanar la nova contrasenya interactivament (és més segur, sinó s'hauria de posar al terminal)
```

### Modificar Entrades Existents

El `changetype: modify` permet afegir, substituir o eliminar atributs d'una entrada existent.

```bash
nano modificar-usuari.ldif
```

**Exemples de modificacions:**

```ldif
# Canviar el correu electrònic d'un usuari
dn: uid=pep,ou=usuaris,dc=educem,dc=cat
changetype: modify
replace: mail
mail: pguardiola@educem.cat

# Afegir un número de telèfon
dn: uid=pep,ou=usuaris,dc=educem,dc=cat
changetype: modify
add: telephoneNumber
telephoneNumber: +34 617 123 456

# Eliminar un atribut
dn: uid=pep,ou=usuaris,dc=educem,dc=cat
changetype: modify
delete: telephoneNumber
```

```bash
ldapmodify -x -D "cn=admin,dc=educem,dc=cat" -W -f modificar-usuari.ldif
```

**Afegir un membre a un grup:**

```ldif
dn: cn=admins,ou=grups,dc=educem,dc=cat
changetype: modify
add: memberUid
memberUid: pere
```

### Eliminar Entrades

```bash
# Eliminar un usuari
ldapdelete -x -D "cn=admin,dc=educem,dc=cat" -W "uid=pep,ou=usuaris,dc=educem,dc=cat"

# Eliminar un grup
ldapdelete -x -D "cn=admin,dc=educem,dc=cat" -W "cn=developers,ou=grups,dc=educem,dc=cat"
```

### Cercar i Consultar el Directori

```bash
# Llistar tots els usuaris
ldapsearch -x -b "ou=usuaris,dc=educem,dc=cat" "(objectClass=posixAccount)" uid cn mail uidNumber

# Cercar un usuari concret per uid
ldapsearch -x -b "dc=educem,dc=cat" "(uid=pep)"

# Llistar tots els grups i els seus membres
ldapsearch -x -b "ou=grups,dc=educem,dc=cat" "(objectClass=posixGroup)" cn gidNumber memberUid

# Cercar usuaris membre d'un grup concret
ldapsearch -x -b "dc=educem,dc=cat" "(&(objectClass=posixGroup)(cn=admins))" memberUid
```

### Esquema Recomanat de UIDs/GIDs

```
Rang d'IDs reservats al sistema:
├── 0-99       → Usuaris i grups del sistema operatiu (root, daemon, etc.)
├── 100-999    → Usuaris de serveis del sistema (www-data, postfix, etc.)
└── 1000+      → Usuaris reals del sistema local

Rang d'IDs recomanat per LDAP:
├── 10000-19999 → Usuaris LDAP (evita conflictes amb users locals)
└── 20000-29999 → Grups LDAP
```

### Operacions en Bloc: Script d'Alta d'Usuaris

Per crear molts usuaris d'un cop, es pot generar el LDIF amb un script (Tema 7). En aquest exemple, tots els alumnes tenen com a grup principal el grup `alumnes` (gidNumber 20010), que s'ha de crear abans:

```bash
#!/bin/bash
# Exemple: crear 10 usuaris de pràctiques amb el grup principal alumnes (20010)

BASE_DN="dc=educem,dc=cat"
OU="ou=usuaris"
START_UID=10011
: > usuaris-alumnes.ldif                 # buida el fitxer si ja existia

for i in $(seq 1 10); do
    USERNAME="alumne$(printf '%02d' "$i")"
    UID_NUM=$((START_UID + i - 1))

    cat << EOF >> usuaris-alumnes.ldif
dn: uid=${USERNAME},${OU},${BASE_DN}
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Alumne $(printf '%02d' "$i")
sn: Alumne
uid: ${USERNAME}
uidNumber: ${UID_NUM}
gidNumber: 20010
homeDirectory: /home/${USERNAME}
loginShell: /bin/bash
userPassword: $(slappasswd -h {SSHA} -s "Canvia1234!")

EOF
done

# Importar tots els usuaris
ldapadd -x -D "cn=admin,${BASE_DN}" -W -f usuaris-alumnes.ldif
```
