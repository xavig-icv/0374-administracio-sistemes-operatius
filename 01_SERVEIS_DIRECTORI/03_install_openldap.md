## 03. Instal·lació d'OpenLDAP

A continuació es mostra com instal·lar i preparar un servidor OpenLDAP a la màquina d'Ubuntu Server 0374.

### Servidor OpenLDAP (Debian/Ubuntu)

OpenLDAP és l'eina per desplegar els serveis de directori en sistemes Linux. El paquet principal és `slapd` (Stand-alone LDAP Daemon), el dimoni del servidor, i `ldap-utils` que inclou les eines de línia d'ordres per interactuar amb el servidor (fer consultes, afegir entrades, fer modificacions, etc.).

```bash
# Actualitzar repositoris
sudo apt update

# Instal·lar servidor OpenLDAP i eines client
sudo apt install slapd ldap-utils

# Durant la instal·lació demanarà la contrasenya de l'administrador LDAP
# Introduïu una contrasenya fàcil de recordar (la podeu canviar més endavant)
```

**Reconfigurar slapd (si teniu algun error amb la configuració inicial):**

```bash
sudo dpkg-reconfigure slapd
```

Durant la reconfiguració es demanen els paràmetres principals:

```
Omit OpenLDAP server configuration?   → No
DNS domain name: educem.cat          ← Defineix el Base DN: dc=educem,dc=cat
Organization name: Educem SL
Administrator password: ****
Confirm password: ****
Database backend: MDB                 ← Motor de base de dades (recomanat)
Remove database when slapd is purged? → No!
Move old database? → Yes
```

### Estructura de Fitxers

```
/etc/ldap/
├── ldap.conf              # Configuració del client LDAP (ldapsearch, etc.)
├── slapd.d/               # Configuració del servidor en format LDIF (OLC)
│   └── cn=config/         # Directori de configuració dinàmica
│       ├── cn=schema/     # Esquemes carregats
│       └── olcDatabase=*/ # Configuració de la base de dades

/var/lib/ldap/             # Base de dades del directori (dades reals)
├── data.mdb               # Fitxer de base de dades MDB
└── lock.mdb

/var/log/
└── syslog                 # Logs del servei slapd
```

### Iniciar i Habilitar el Servei

```bash
# Iniciar o reiniciar el servei
sudo systemctl restart slapd

# Verificar l'estat
sudo systemctl status slapd

# Habilitar l'encesa automàtica durant l'arrencada
sudo systemctl enable slapd

# Verificar que escolta al port 389 (quan sigui segur al 636)
sudo ss -plutn | grep slapd
```

### Verificar la Instal·lació

```bash
# Fer una consulta bàsica al servidor local (sense autenticació)
ldapsearch -x -b "" -s base namingContexts

# Ha de retornar el Base DN del directori:
# namingContexts: dc=educem,dc=cat
```

```bash
# Consultar tot el contingut del directori (autenticat com a admin)
ldapsearch -x -D "cn=admin,dc=educem,dc=cat" -W -b "dc=educem,dc=cat"

# -x     → Autenticació simple (sense SASL)
# -D     → DN de l'administrador
# -W     → Demanar contrasenya interactivament
# -b     → Base DN de la cerca
```

### Configurar el Client LDAP Local

El fitxer `/etc/ldap/ldap.conf` defineix els valors per defecte per a les eines client LDAP (`ldapsearch`, `ldapadd`, etc.):

```bash
sudo nano /etc/ldap/ldap.conf
```

**Contingut:**

```bash
# /etc/ldap/ldap.conf
# Configuració del client LDAP

# Servidor LDAP per defecte (la IP del 0374)
URI     ldap://192.168.1.1

# Base DN per defecte (evita haver d'especificar -b cada vegada)
BASE    dc=educem,dc=cat

# La configuració de TLS (TLS_CACERT i TLS_REQCERT) s'afegeix al fitxer 09
```

### Eines de Línia d'Ordres (ldap-utils)

| Ordre        | Funció                                  |
| ------------ | --------------------------------------- |
| `ldapsearch` | Buscar entrades al directori            |
| `ldapadd`    | Afegir entrades des d'un fitxer LDIF    |
| `ldapmodify` | Modificar entrades existents            |
| `ldapdelete` | Eliminar entrades                       |
| `ldappasswd` | Canviar contrasenyes                    |
| `slapcat`    | Exportar tota la base de dades (backup) |
| `slapadd`    | Importar una base de dades completa     |

```bash
# Exemple: buscar un usuari concret
ldapsearch -x -D "cn=admin,dc=educem,dc=cat" -W -b "ou=usuaris,dc=educem,dc=cat" "(uid=joan)"

# Exemple: exportar (backup) de tot el directori
sudo slapcat -v -l backup-ldap.ldif
```
