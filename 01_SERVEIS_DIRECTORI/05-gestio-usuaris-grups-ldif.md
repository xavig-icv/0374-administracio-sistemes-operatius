## 05. Gestió d'Usuaris i Grups amb LDIF

La gestió d'usuaris i grups és l'acció més habitual que es realitza a un servidor LDAP. En aquest apartat veurem com crear, modificar i eliminar usuaris i grups, com gestionar contrasenyes i com realitzar operacions en bloc.

### Crear Usuaris

Cada usuari, al directori LDAP, ha de tenir com a mínim les classes d'objecte `inetOrgPerson` i `posixAccount` (per poder integrar-se amb els sistemes Linux).

**Fitxer LDIF per crear un usuari:**

**Abans generem una contrasenya per l'usuari**
**Generar un hash de contrasenya manualment:**

```bash
# Generar hash SSHA (recomanat)
slappasswd -h {SSHA} -s "contrasenya_provisional"
New password: contrasenya_provisional
Re-enter new password: contrasenya_provisional
#Retorna: {SSHA}z6r+3GqL3gk0y+qz+2K1QmGZk6iXQnLr

# Generar hash SHA-256 (més modern però servidors antics no ho soporten)
slappasswd -h {SHA256} -s "la_meva_contrasenya"
```

```bash
nano usuari-pep.ldif
```

```ldif
dn: uid=pep,ou=usuaris,dc=empresa,dc=cat
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
mail: pep@empresa.cat
userPassword: {SSHA}z6r+3GqL3gk0y+qz+2K1QmGZk6iXQnLr
```

```bash
# Afegir l'usuari al directori
sudo ldapadd -x -D "cn=admin,dc=empresa,dc=cat" -W -f usuari-pep.ldif
```

**Crear múltiples usuaris en un sol fitxer LDIF** (separats per un salt de línia i una línia en blanc):

```ldif
dn: uid=julia,ou=usuaris,dc=empresa,dc=cat
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
mail: julia@empresa.cat
userPassword: {SSHA}contrasenya_provisional

dn: uid=pere,ou=usuaris,dc=empresa,dc=cat
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
mail: pere@empresa.cat
userPassword: {SSHA}contrasenya_provisional
```

**Nota sobre uidNumber i gidNumber:**
Han de ser únics al directori i no coincidir amb UIDs de sistema (normalment < 1000). Una bona pràctica és iniciar-los a 1000 o 10000.

### Crear Grups

```bash
nano grups.ldif
```

Recomano crear els grups genèrics a partir d'un número que no entri en conflicte amb els usuaris. Per exemple 11001 en endavant.

```ldif
# Grup d'Administradors
dn: cn=admins,ou=grups,dc=empresa,dc=cat
objectClass: posixGroup
cn: admins
gidNumber: 11001
memberUid: pep
memberUid: julia

# Grup de Desenvolupadors
dn: cn=developers,ou=grups,dc=empresa,dc=cat
objectClass: posixGroup
cn: developers
gidNumber: 11002
memberUid: julia
memberUid: pere
```

```bash
sudo ldapadd -x -D "cn=admin,dc=empresa,dc=cat" -W -f grups.ldif
```

### Gestió de Contrasenyes

**Establir o canviar la contrasenya d'un usuari:**

```bash
# Canviar contrasenya (demanarà la contrasenya nova interactivament)
sudo ldappasswd -x -D "cn=admin,dc=empresa,dc=cat" -W -S "uid=pep,ou=usuaris,dc=empresa,dc=cat"

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
dn: uid=pep,ou=usuaris,dc=empresa,dc=cat
changetype: modify
replace: mail
mail: pguardiola@empresa.cat

# Afegir un número de telèfon
dn: uid=pep,ou=usuaris,dc=empresa,dc=cat
changetype: modify
add: telephoneNumber
telephoneNumber: +34 617 123 456

# Eliminar un atribut
dn: uid=pep,ou=usuaris,dc=empresa,dc=cat
changetype: modify
delete: telephoneNumber
```

```bash
ldapmodify -x -D "cn=admin,dc=empresa,dc=cat" -W -f modificar-usuari.ldif
```

**Afegir un membre a un grup:**

```ldif
dn: cn=admins,ou=grups,dc=empresa,dc=cat
changetype: modify
add: memberUid
memberUid: pere
```

### Eliminar Entrades

```bash
# Eliminar un usuari
ldapdelete -x -D "cn=admin,dc=empresa,dc=cat" -W "uid=pep,ou=usuaris,dc=empresa,dc=cat"

# Eliminar un grup
ldapdelete -x -D "cn=admin,dc=empresa,dc=cat" -W "cn=developers,ou=grups,dc=empresa,dc=cat"
```

### Cercar i Consultar el Directori

```bash
# Llistar tots els usuaris
ldapsearch -x -b "ou=usuaris,dc=empresa,dc=cat" "(objectClass=posixAccount)" uid cn mail uidNumber

# Cercar un usuari concret per uid
ldapsearch -x -b "dc=empresa,dc=cat" "(uid=pep)"

# Llistar tots els grups i els seus membres
ldapsearch -x -b "ou=grups,dc=empresa,dc=cat" "(objectClass=posixGroup)" cn gidNumber memberUid

# Cercar usuaris membre d'un grup concret
ldapsearch -x -b "dc=empresa,dc=cat" "(&(objectClass=posixGroup)(cn=admins))" memberUid
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

Per crear molts usuaris d'un cop, es pot generar el LDIF programàticament:

```bash
#!/bin/bash
# Exemple: crear 10 usuaris de pràctiques

BASE_DN="dc=empresa,dc=cat"
OU="ou=usuaris"
START_UID=10011

for i in $(seq 1 10); do
    USERNAME="alumne$(printf '%02d' $i)"
    UID_NUM=$((START_UID + i - 1))

    cat << EOF >> usuaris-alumnes.ldif
dn: uid=${USERNAME},${OU},${BASE_DN}
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Alumne $(printf '%02d' $i)
sn: Alumne
uid: ${USERNAME}
uidNumber: ${UID_NUM}
gidNumber: 10011
homeDirectory: /home/${USERNAME}
loginShell: /bin/bash
userPassword: {SSHA}$(slappasswd -h {SSHA} -s "Canvia1234!" | sed 's/{SSHA}//')

EOF
done

# Importar tots els usuaris
ldapadd -x -D "cn=admin,${BASE_DN}" -W -f usuaris-alumnes.ldif
```
