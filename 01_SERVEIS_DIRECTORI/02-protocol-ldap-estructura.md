## 02. El Protocol LDAP - Estructura i Funcionament

LDAP (Lightweight Directory Access Protocol) és el protocol estàndard per accedir i gestionar la informació emmagatzemada als serveis de directori. Els serveis de directori són bases de dades especialitzades que guarden la informació de manera jeràrquica, com ara usuaris, grups, dispositius i altres recursos de xarxa.

LDAP disposa d'un conjunt d'operacions per consultar, afegir, modificar o eliminar entrades dins d'un directori. Pràcticament tots els serveis de directori moderns (OpenLDAP, Active Directory, etc.) utilitzen LDAP per disposar d'una autenticació centralitzada i gestionar usuaris en entorns corporatius.

![Funcionament de LDAP](./img/1-funcionament-ldap.png)

### L'Arbre de Directoris (DIT - Directory Information Tree)

El directori LDAP organitza la informació en una estructura jeràrquica d'arbre, anomenada **DIT (Directory Information Tree)**. Cada node de l'arbre és una **entrada (entry)**, i cada entrada conté un conjunt d'**atributs** amb els seus valors.

```
dc=empresa,dc=cat                  ← Arrel (Root / Base DN)
│
├── ou=usuaris                     ← Unitat organitzativa (OU)
│   ├── uid=pep                    ← Usuari
│   ├── uid=julia
│   └── uid=pere
│
├── ou=grups
│   ├── cn=admins
│   └── cn=desenvolupadors
│
└── ou=equips
    ├── cn=servidor-web
    └── cn=impressora-planta2
```

### El Servidor LDAP

**Directori:**

Emmagatzema les dades en una estructura jeràrquica d'entrades normalment associats a un domini (d'una organització). Cada entrada té un identificador únic anomenat DN (Distinguished Name).

Imagina un arbre on les branques representen a cada departament de l'empresa (RRHH, IT, Vendes…) i les fulles a cada empleat dins d'aquests departaments.

**DN (Distinguished Name):**

És el camí complet i únic que identifica una entrada dins del directori. És com la ruta absoluta d'un fitxer (des de l'arrel): /home/usuari/Escriptori/ldap.txt.

```
uid=pep,ou=usuaris,dc=empresa,dc=cat
  ↑           ↑           ↑
 RDN  +  Unitat org. + Domini base = DN (tot plegat és el DN)
```

**RDN (Relative Distinguished Name):**

La part més a l'esquerra del DN, que identifica l'entrada dins del seu contenidor pare (un DN pot identificar a usuaris, grups, equips, OU's, etc.).

```
uid=pep           → RDN d'un usuari
cn=admins         → RDN d'un grup
ou=usuaris        → RDN d'una unitat organitzativa
```

**Atributs comuns:**

| Atribut        | Significat          | Exemple                |
| -------------- | ------------------- | ---------------------- |
| `dc`           | Domain Component    | `dc=empresa`           |
| `ou`           | Organizational Unit | `ou=usuaris`           |
| `cn`           | Common Name         | `cn=Pep Guardiola`     |
| `uid`          | User ID             | `uid=pguardiola`       |
| `sn`           | Surname             | `sn=Guardiola`         |
| `mail`         | Adreça de correu    | `mail=pep@empresa.cat` |
| `userPassword` | Contrasenya (hash)  | `{SSHA}abc123...`      |
| `uidNumber`    | UID Unix            | `uidNumber=1001`       |
| `gidNumber`    | GID Unix            | `gidNumber=1001`       |

**Esquemes**

Els esquemes defineixen l'estructura de les dades dins d'un directori LDAP. Especifiquen els tipus d'objectes que poden existir i quins atributs poden o han de tenir. D'aquesta manera asseguren que la informació emmagatzemada segueixi una estructura coherent i consistent.

Imagina un formulari d'inscripció on es defineixen les regles que cada usuari ha de tenir: un nom, una adreça de correu electrònic, un número de telèfon, etc.

**ObjectClass:**

Cada entrada del directori ha de pertànyer a una o més classes d'objecte (objectClass). Aquestes classes defineixen quins atributs són obligatoris i quins són opcionals per a aquell tipus d'objecte.

Per exemple, una classe d'objecte que representi un usuari pot requerir atributs com el nom o el correu electrònic, mentre que altres atributs poden ser opcionals..

```
Classes d'objecte principals:
  inetOrgPerson    → Usuaris (cn, mail, uid, sn, etc.)
  posixAccount     → Integració Unix (uidNumber, gidNumber, homeDirectory)
  posixGroup       → Grups Unix (gidNumber, memberUid)
  organizationalUnit → Unitats organitzatives (ou)
```

### Els Clients LDAP

**Autenticació:**

Els clients poden demostrar la seva identitat davant del servidor LDAP per accedir a la informació del directori. Això es fa mitjançant les credencials proporcionades per l'administrador (usuari i contrasenya).

**Consultes:**

Els clients poden realitzar consultes per obtenir informació específica del directori. Les consultes es fan utilitzant filtres LDAP. Si ets un administrador de sistemes pots fer una consulta LDAP per obtenir una llista de tots els usuaris que pertanyen al departament de Vendes o de IT per exemple.

- **Informació d'usuaris:**

  Els clients fan consultes per obtenir informació sobre un usuari: com el seu correu electrònic, número de telèfon o departament.

- **Gestió de recursos:**

  Es poden fer consultes per trobar dispositius o recursos de xarxa, com impressores o servidors.

**Modificacions:**

Els clients amb els permisos adequats poden afegir, modificar o eliminar entrades del directori. Pensa en els permisos de Lectura, Escriptura i Execució sobre fitxers i carpetes a Linux. Quan un nou empleat s'incorpora a l'empresa, es pot afegir una nova entrada al directori LDAP amb tota la seva informació. Si un empleat deixa l'empresa, es pot eliminar la seva entrada

- **Actualització de dades:**

  Permet actualitzar la informació dels usuaris, com canviar el seu número de telèfon o adreça de correu electrònic.

- **Gestió d'usuaris:**

  Es poden afegir nous usuaris al directori o eliminar aquells que ja no necessiten accés.

### Operacions LDAP Principals

| Operació   | Descripció                       |
| ---------- | -------------------------------- |
| **bind**   | Autenticació al servidor (login) |
| **search** | Buscar entrades al directori     |
| **add**    | Afegir una nova entrada          |
| **modify** | Modificar atributs d'una entrada |
| **delete** | Eliminar una entrada             |
| **modrdn** | Reanomenar o moure una entrada   |
| **unbind** | Tancar la sessió                 |

### Filtres de Cerca LDAP

Els filtres permeten fer cerques d'entrades amb uns criteris o filtres específics:

```bash
# Buscar per uid
(uid=pep)

# Buscar per objectClass
(objectClass=posixAccount)

# Buscar usuaris amb un correu específic
(mail=*@empresa.cat)

# Buscar membres d'un grup (AND lògic)
(&(objectClass=posixAccount)(gidNumber=1001))

# Buscar per nom o cognom (OR lògic)
(|(cn=Pep*)(sn=Guardiola))

# Buscar usuaris sense correu configurat (NOT)
(!(mail=*))
```

### Ports i Seguretat

| Port               | Protocol | Descripció                    |
| ------------------ | -------- | ----------------------------- |
| **389**            | TCP      | LDAP estàndard (sense xifrar) |
| **636**            | TCP      | LDAPS (LDAP sobre TLS/SSL)    |
| **389 + STARTTLS** | TCP      | LDAP amb negociació TLS       |

### Format LDIF (LDAP Data Interchange Format)

LDIF és el format estàndard per representar i intercanviar dades LDAP. S'utilitza per importar, exportar i modificar entrades del directori.

Si es vol crear una o diverses entrades, normalment es crea un fitxer amb extensió LDIF i s'hi posa tota la informació necessària amb la sintaxi i regles corresponents. Per exemple: usuaris.ldif

```ldif
# Exemple d'una entrada d'usuari en format LDIF
dn: uid=pep,ou=usuaris,dc=empresa,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Pep Guardiola
sn: Guardiola
uid: pep
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/pep
loginShell: /bin/bash
mail: pep@empresa.cat
userPassword: {SSHA}contrasenya_en_hash
```

Les regles bàsiques del format LDIF:

- Cada atribut va en una línia separada amb el format `atribut: valor`
- Les entrades es separen amb una línia en blanc
- Els comentaris comencen amb `#`
- El `dn:` sempre ha de ser la primera línia de cada entrada
