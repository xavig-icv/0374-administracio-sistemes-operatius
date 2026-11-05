## 09. Seguretat: LDAPS i Xifrat TLS

Fins ara totes les comunicacions entre clients i el servidor LDAP viatgen en text pla pel port 389. Això significa que qualsevol dispositiu al mateix segment de xarxa pot capturar les credencials dels usuaris durant el procés d'autenticació.

**Demostració del problema** (fes una captura amb tcpdump):

```bash
# Al servidor, capturar trànsit LDAP mentre un client s'autentica
sudo tcpdump -i enp0s8 port 389 -A

# A la captura es pot veure en text pla:
# uid=pep
# userPassword: la_contrasenya_en_clar
```

LDAPS resol aquest problema xifrant tota la comunicació amb TLS, de la mateixa manera que HTTPS xifra el trànsit web. Treballa al port **636** i és imprescindible per servidors en producció.

**STARTTLS vs LDAPS:**
Hi ha dues maneres de xifrar les comunicacions de LDAP:

- **LDAPS** (port 636): Connexió xifrada des del primer byte. És el mètode més senzill i recomanat.
- **STARTTLS** (port 389): Comença sense xifrar i negocia el xifrat durant la connexió. Més complex però necessari quan el client no suporta LDAPS.

En aquest apartat configurem **LDAPS**, que és el que requeriran les integracions posteriors com GLPI i clients Windows (RA6).

### Generar els Certificats

Per a un entorn de laboratori, generarem un certificat autosignat. En producció s'hauria d'utilitzar un certificat d'una CA vàlida (Let's Encrypt, etc.).

En el nostre cas, els clients Ubuntu hauran d'instal·lar el certificat que generem com a Autoritat Certificadora (CA).

```bash
# Crear el directori per als certificats
sudo mkdir -p /etc/ldap/certs
cd /etc/ldap/certs

# 1. Generar la clau privada de la CA (Autoritat Certificadora pròpia)
sudo openssl genrsa -out ca.key 4096

# 2. Generar el certificat de la CA (vàlid 10 anys)
sudo openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -subj "/C=ES/ST=Catalunya/L=Barcelona/O=Educem SL/CN=CA-LDAP"

# 3. Generar la clau privada del servidor LDAP
sudo openssl genrsa -out ldap-server.key 2048

# 4. Generar la sol·licitud de certificat (CSR)
sudo openssl req -new -key ldap-server.key -out ldap-server.csr -subj "/C=ES/ST=Catalunya/L=Barcelona/O=Educem SL/CN=ldap.educem.cat"

# 5. Signar el certificat amb la nostra CA, afegint les adreces dels dos servidors (SAN)
echo "subjectAltName=DNS:ldap.educem.cat,IP:192.168.1.1,IP:192.168.1.2" | sudo tee san.ext
sudo openssl x509 -req -days 3650 -in ldap-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ldap-server.crt -extfile san.ext

# 6. Verificar el certificat generat
openssl verify -CAfile ca.crt ldap-server.crt
# Ha de retornar: ldap-server.crt: OK
openssl x509 -in ldap-server.crt -noout -ext subjectAltName
# Ha de mostrar: DNS:ldap.educem.cat, IP Address:192.168.1.1, IP Address:192.168.1.2
```

**Per què cal el SAN?** En connectar-se, el client comprova que l'adreça que ha fet servir (`192.168.1.1` o el nom `ldap.educem.cat`) apareix a l'extensió *subjectAltName* (SAN) del certificat. Els clients actuals no accepten una IP escrita només al CN. Com que aquest certificat inclou les IP dels dos servidors, el mateix certificat serveix per al primari i per al secundari.

**Assignar els permisos correctes** (slapd s'executa com a usuari `openldap`):

```bash
sudo chown -R openldap:openldap /etc/ldap/certs/
sudo chmod 640 /etc/ldap/certs/ldap-server.key
sudo chmod 644 /etc/ldap/certs/ldap-server.crt
sudo chmod 644 /etc/ldap/certs/ca.crt
```

### Configurar TLS a OpenLDAP

```bash
nano tls-config.ldif
```

```ldif
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/certs/ca.crt
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/certs/ldap-server.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/certs/ldap-server.key
-
add: olcTLSVerifyClient
olcTLSVerifyClient: never
```

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f tls-config.ldif
```

Si els atributs TLS ja existien d'una configuració anterior, canviar `add:` per `replace:` al fitxer LDIF.

### Habilitar el Port 636 (LDAPS)

S'ha d'indicar al servei de LDAP que escolti també al port 636:

```bash
sudo nano /etc/default/slapd
```

**Modificar la línia `SLAPD_SERVICES`:**

```bash
# Afegir ldaps:/// a la llista de serveis
SLAPD_SERVICES="ldap:/// ldaps:/// ldapi:///"

#  ldap:///   → Port 389 (mantenint compatibilitat)
#  ldaps:///  → Port 636 (xifrat)
#  ldapi:///  → Socket Unix local (administració)
```

```bash
sudo systemctl restart slapd

# Verificar que escolta als dos ports
sudo ss -plutn | grep slapd
# Ha de mostrar els ports 389 i 636
```

### Verificar LDAPS

```bash
# Provar la connexió LDAPS verificant el certificat amb la nostra CA
LDAPTLS_CACERT=/etc/ldap/certs/ca.crt ldapsearch -x -H ldaps://192.168.1.1 -D "cn=admin,dc=educem,dc=cat" -W -b "dc=educem,dc=cat" "(objectClass=posixAccount)" uid cn
```

La variable `LDAPTLS_CACERT` indica a `ldapsearch` quina CA ha de fer servir per verificar el certificat. L'opció `-o tls_reqcert=never` desactiva la verificació: només s'ha de fer servir per diagnosticar un problema, mai com a configuració, perquè llavors qualsevol servidor es podria fer passar pel nostre.

**Captura comparativa** per veure la diferència entre LDAP i LDAPS:

```bash
# Terminal 1: Capturar trànsit
sudo tcpdump -i enp0s8 port 389 or port 636 -A

# Terminal 2: Connexió sense xifrar (port 389) → veurem dades en text pla
ldapsearch -x -H ldap://192.168.1.1 -D "uid=pep,ou=usuaris,dc=educem,dc=cat" -w "la_contrasenya" -b "dc=educem,dc=cat" "(uid=pep)"

# Terminal 2: Connexió xifrada (port 636) → veurem dades xifrades
LDAPTLS_CACERT=/etc/ldap/certs/ca.crt ldapsearch -x -H ldaps://192.168.1.1 -D "uid=pep,ou=usuaris,dc=educem,dc=cat" -w "la_contrasenya" -b "dc=educem,dc=cat" "(uid=pep)"
```

Escriure la contrasenya amb `-w` només té sentit en aquesta demostració: queda a l'historial del terminal. Normalment es fa servir `-W`, que la demana.

### Configurar els Clients per Utilitzar LDAPS

**Al servidor secundari (replicació sobre TLS):**

El secundari necessita el certificat de la CA per verificar el primari. A més, com que els clients també s'hi connectaran per LDAPS, necessita el certificat i la clau del servidor (el certificat ja inclou la IP `192.168.1.2`):

```bash
# Al primari: còpia temporal de la clau que pugui llegir el vostre usuari, i enviament
sudo install -m 600 -o "$USER" /etc/ldap/certs/ldap-server.key /tmp/
scp /etc/ldap/certs/ca.crt /etc/ldap/certs/ldap-server.crt /tmp/ldap-server.key usuari@192.168.1.2:/tmp/
rm /tmp/ldap-server.key
# Al secundari:
sudo mkdir -p /etc/ldap/certs
sudo mv /tmp/{ca.crt,ldap-server.crt,ldap-server.key} /etc/ldap/certs/
sudo chown -R openldap:openldap /etc/ldap/certs
sudo chmod 640 /etc/ldap/certs/ldap-server.key
```

Al secundari també cal aplicar el mateix `tls-config.ldif` i afegir `ldaps:///` a `SLAPD_SERVICES` (apartats anteriors).

```ldif
# Actualitzar syncrepl per utilitzar LDAPS
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSyncRepl
olcSyncRepl: rid=001
  provider=ldaps://192.168.1.1
  bindmethod=simple
  binddn="cn=replicador,dc=educem,dc=cat"
  credentials=contrasenya_del_replicador
  searchbase="dc=educem,dc=cat"
  attrs="*,+"
  schemachecking=on
  type=refreshAndPersist
  retry="60 10 300 +"
  tls_cacert=/etc/ldap/certs/ca.crt
```

Com al fitxer 08, `credentials` és la contrasenya en text pla del compte `replicador`.

**Als clients Linux (nslcd):**

```bash
sudo nano /etc/nslcd.conf
```

```bash
# Canviar uri per utilitzar LDAPS
uri ldaps://192.168.1.1 ldaps://192.168.1.2

# Especificar el certificat de la CA per verificar el servidor
tls_cacertfile /etc/ldap/certs/ca.crt

# Exigir un certificat vàlid, signat per la nostra CA i amb la IP al SAN
tls_reqcert demand
```

Amb la CA instal·lada al client i el SAN correcte, la verificació funciona també amb certificats propis: no cal desactivar-la (`tls_reqcert never`).

```bash
# Distribuir el certificat de la CA als clients
scp /etc/ldap/certs/ca.crt usuari@IP_CLIENT:/tmp/
# Al client:
sudo mkdir -p /etc/ldap/certs
sudo mv /tmp/ca.crt /etc/ldap/certs/ca.crt

sudo systemctl restart nslcd
```

**Al fitxer `/etc/ldap/ldap.conf`** (eines client com `ldapsearch`):

```bash
URI     ldaps://192.168.1.1
BASE    dc=educem,dc=cat
TLS_CACERT /etc/ldap/certs/ca.crt
# Exigir un certificat vàlid (els comentaris han d'anar en una línia a part)
TLS_REQCERT demand
```

### Desactivar LDAP No Xifrat (Producció)

Un cop tots els clients utilitzen LDAPS, es pot desactivar el port 389 per forçar el xifrat:

```bash
sudo nano /etc/default/slapd

# Eliminar ldap:/// per desactivar el port 389
SLAPD_SERVICES="ldaps:/// ldapi:///"

sudo systemctl restart slapd
```

**Atenció:** Abans de desactivar el port 389, verificar que **tots** els clients (Linux, Windows, aplicacions com GLPI) estan configurats per utilitzar LDAPS. En cas contrari, perdran l'autenticació.

### Preparació per a Integracions Posteriors (RA6)

El certificat de la CA generat en aquest apartat serà necessari per a les integracions amb:

- **GLPI**: Cal importar el `ca.crt` a la configuració d'autenticació LDAP de GLPI perquè accepti el certificat del servidor.
- **Clients Windows**: Cal distribuir el `ca.crt` com a certificat de confiança als clients Windows (via GPO o manualment) perquè acceptin la connexió LDAPS. Windows, per defecte, rebutja certificats autosignats que no estan al seu magatzem de confiança.
- **Samba / Active Directory**: Si s'integra OpenLDAP amb Samba per autenticar usuaris Windows al domini, LDAPS és obligatori per al funcionament correcte.

```
Resum de ports i quan utilitzar-los:
┌────────┬──────────┬───────────────────────────────────────┐
│ Port   │ Protocol │ Quan utilitzar-lo                     │
├────────┼──────────┼───────────────────────────────────────┤
│ 389    │ LDAP     │ Mai en producció. Només laboratori.   │
│ 389+   │ STARTTLS │ Si el client no suporta LDAPS (636)   │
│ 636    │ LDAPS    │ Sempre en producció. GLPI, Windows... │
└────────┴──────────┴───────────────────────────────────────┘
```
