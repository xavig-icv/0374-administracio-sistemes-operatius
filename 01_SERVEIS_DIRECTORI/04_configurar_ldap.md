## 04. Configuració Bàsica del Servidor LDAP

Un cop instal·lat OpenLDAP, el servidor ja disposa d'una estructura mínima (el node arrel i l'entrada de l'administrador). Ara hem de crear l'estructura del directori i carregar els esquemes necessaris.

### Estructura Inicial del Directori

Després d'instal·lar `slapd`, el directori conté únicament el node arrel. Cal crear les Unitats Organitzatives (OU) per organitzar els usuaris i grups.

```
dc=empresa,dc=cat              ← Creat automàticament per slapd
├── cn=admin                   ← Creat automàticament (administrador)
├── ou=usuaris                 ← Cal crear-lo
└── ou=grups                   ← Cal crear-lo
```

### Carregar Esquemes Addicionals

Els esquemes (schemas) defineixen les classes d'objecte i atributs disponibles. Per poder crear usuaris amb atributs Unix (uidNumber, gidNumber, homeDirectory, loginShell), es a dir, integrar el Servidor LDAP i l'Ubuntu Server cal carregar l'esquema `nis`:

`ldapadd` l'utilitzem per afegir noves entrades al servei de directoris.

```bash
# Carregar esquema NIS (necessari per posixAccount i posixGroup)
# Això permetrà al LDAP gestionar l'autenticació del usuaris del servidor Ubuntu.
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/nis.ldif

# Carregar esquema cosine (per atributs addicionals)
# Permet carregar informació com: mail, telephoneNumber, ou, title, etc.
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/cosine.ldif

# Carregar esquema inetorgperson (per inetOrgPerson)
# Amplia els atributs per tenir les màximes dades dels usuaris humans
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/inetorgperson.ldif

# Verificar esquemes carregats
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=schema,cn=config" dn
```

`-Y EXTERNAL -H ldapi:///` utilitza el socket Unix local per autenticar-se com a root del sistema, sense necessitat d'introduir contrasenya LDAP. Només funcionarà en comandes executades des del mateix servidor.

Els canvis a la configuració (`cn=config`), com carregar esquemes, només es poden fer d'aquesta manera. L'administrador del directori (`cn=admin,dc=empresa,dc=cat`) gestiona les dades de `dc=empresa,dc=cat` amb autenticació simple (`-x -D "cn=admin,dc=empresa,dc=cat" -W`), però no pot modificar la configuració del servidor.

`Nota:` A Ubuntu, els esquemes `cosine`, `nis` i `inetorgperson` ja es carreguen durant la instal·lació de `slapd`. Si els torneu a afegir, `ldapadd` mostra l'error `Duplicate attributeType`. Comproveu primer quins hi ha amb l'ordre de verificació anterior.

### Crear l'Estructura Base (DIT)

Crear un fitxer LDIF amb l'estructura inicial del directori:

```bash
nano estructura-base.ldif
```

**Contingut:**

```ldif
# Unitat Organitzativa: Usuaris
dn: ou=usuaris,dc=empresa,dc=cat
objectClass: organizationalUnit
ou: usuaris
description: Usuaris del sistema

# Unitat Organitzativa: Grups
dn: ou=grups,dc=empresa,dc=cat
objectClass: organizationalUnit
ou: grups
description: Grups del sistema
```

**Importar l'estructura al directori:**

```bash
sudo ldapadd -x -D "cn=admin,dc=empresa,dc=cat" -W -f estructura-base.ldif

# Verificar que s'ha creat correctament
sudo ldapsearch -x -b "dc=empresa,dc=cat" -s one dn
```

### Configurar Índexs per Millor Rendiment

Els índexs permeten accelerar les cerques dins del directori, de manera similar als índexs utilitzats en les bases de dades. En lloc de recórrer totes les entrades, el servidor pot utilitzar aquests índexs per localitzar la informació de manera més eficient.

A OpenLDAP, els índexs es configuren dins del sistema de configuració dinàmica OLC (Online Configuration):

```bash
nano indexs.ldif
```

**Contingut:**

Permet crear índexs per quan es facin cerques per nom (cn), per cognom (sn), per usuari (uid), per correu (mail), etc.

```ldif
# Afegir índexs a la base de dades
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcDbIndex
olcDbIndex: uid eq,pres,sub
olcDbIndex: cn eq,pres,sub
olcDbIndex: sn eq,pres,sub
olcDbIndex: mail eq,pres,sub
olcDbIndex: memberUid eq,pres
olcDbIndex: gidNumber eq,pres
olcDbIndex: uidNumber eq,pres
olcDbIndex: objectClass eq
```

```bash
# Aplicar els índexs (requereix permís root via socket)
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f indexs.ldif
```

### Verificar i Monitorar el Servei

```bash
# Verificar la sintaxi de la configuració
sudo slaptest -u

# Veure els logs del servei en temps real
sudo journalctl -u slapd -f

# Veure les connexions actives
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=monitor" \
  "(objectClass=monitorConnection)" dn 2>/dev/null

# Consultar estadístiques del servidor
sudo ldapsearch -Y EXTERNAL -H ldapi:/// \
  -b "cn=Statistics,cn=Monitor" 2>/dev/null
```

### Backup i Restauració

```bash
# BACKUP: Exportar tot el directori a LDIF
sudo slapcat -v -l /backup/ldap-backup-$(date +%Y%m%d).ldif

# RESTAURACIÓ: (el servei ha d'estar aturat)
sudo systemctl stop slapd
sudo rm -f /var/lib/ldap/*
sudo slapadd -v -l /backup/ldap-backup-20260315.ldif
sudo chown -R openldap:openldap /var/lib/ldap
sudo systemctl start slapd
```

### Resum de l'Estructura Final

```
dc=empresa,dc=cat
│
├── cn=admin                      ← Administrador LDAP
│
├── ou=usuaris                    ← Contenidor d'usuaris
│   ├── uid=joan
│   ├── uid=maria
│   └── uid=pere
│
└── ou=grups                      ← Contenidor de grups
    ├── cn=admins
    └── cn=desenvolupadors
```
