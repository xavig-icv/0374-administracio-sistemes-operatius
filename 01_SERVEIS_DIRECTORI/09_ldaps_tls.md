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
sudo openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -subj "/C=ES/ST=Catalunya/L=Barcelona/O=Empresa SL/CN=CA-LDAP"

# 3. Generar la clau privada del servidor LDAP
sudo openssl genrsa -out ldap-server.key 2048

# 4. Generar la sol·licitud de certificat (CSR)
sudo openssl req -new -key ldap-server.key -out ldap-server.csr -subj "/C=ES/ST=Catalunya/L=Barcelona/O=Empresa SL/CN=192.168.1.1"

# IMPORTANT: El CN ha de ser la IP o el FQDN del servidor LDAP
# que utilitzaran els clients per connectar-s'hi.
# En un entorn real seria: CN=ldap.empresa.cat

# 5. Signar el certificat del servidor amb la nostra CA
sudo openssl x509 -req -days 3650 -in ldap-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ldap-server.crt

# 6. Verificar el certificat generat
openssl verify -CAfile ca.crt ldap-server.crt
# Ha de retornar: ldap-server.crt: OK
```

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
# Provar connexió LDAPS (amb verificació de certificat desactivada per a proves)
ldapsearch -x -H ldaps://192.168.1.1 -b "dc=empresa,dc=cat" -o tls_reqcert=never "(objectClass=posixAccount)" uid cn

# Provar amb verificació completa del certificat (mode producció)
ldapsearch -x -H ldaps://192.168.1.1 -b "dc=empresa,dc=cat" "(objectClass=posixAccount)" uid cn
```

**Captura comparativa** per veure la diferència entre LDAP i LDAPS:

```bash
# Terminal 1: Capturar trànsit
sudo tcpdump -i enp0s8 port 389 or port 636 -A

# Terminal 2: Connexió sense xifrar (port 389) → veurem dades en text pla
ldapsearch -x -H ldap://192.168.1.1 -D "uid=pep,ou=usuaris,dc=empresa,dc=cat" -w "la_contrasenya" -b "dc=empresa,dc=cat" "(uid=pep)"

# Terminal 2: Connexió xifrada (port 636) → veurem dades xifrades
ldapsearch -x -H ldaps://192.168.1.1 -D "uid=pep,ou=usuaris,dc=empresa,dc=cat" -w "la_contrasenya" -b "dc=empresa,dc=cat" "(uid=pep)" -o tls_reqcert=never
```

### Configurar els Clients per Utilitzar LDAPS

**Al servidor secundari (replicació sobre TLS):**

Copiar el certificat de la CA al secundari i actualitzar la configuració de `syncrepl`:

```bash
# Copiar el certificat de la CA del primari al secundari
scp /etc/ldap/certs/ca.crt usuari@192.168.1.2:/tmp/
# Al secundari:
sudo mv /tmp/ca.crt /etc/ldap/certs/ca.crt
sudo chown openldap:openldap /etc/ldap/certs/ca.crt
```

```ldif
# Actualitzar syncrepl per utilitzar LDAPS
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSyncRepl
olcSyncRepl: rid=001
  provider=ldaps://192.168.1.1
  bindmethod=simple
  binddn="cn=replicador,dc=empresa,dc=cat"
  credentials={SSHA}contrasenya_replicacio_segura
  searchbase="dc=empresa,dc=cat"
  attrs="*,+"
  schemachecking=on
  type=refreshAndPersist
  retry="60 10 300 +"
  tls_cacert=/etc/ldap/certs/ca.crt
```

**Als clients Linux (nslcd):**

```bash
sudo nano /etc/nslcd.conf
```

```bash
# Canviar uri per utilitzar LDAPS
uri ldaps://192.168.1.1 ldaps://192.168.1.2

# Especificar el certificat de la CA per verificar el servidor
tls_cacertfile /etc/ldap/certs/ca.crt

# En producció: always. En proves amb cert autosignat: never
tls_reqcert never
```

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
BASE    dc=empresa,dc=cat
TLS_CACERT /etc/ldap/certs/ca.crt
TLS_REQCERT allow    # 'allow' per cert autosignat, 'hard' en producció
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
