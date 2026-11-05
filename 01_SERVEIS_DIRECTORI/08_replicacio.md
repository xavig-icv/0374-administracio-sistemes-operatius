## 08. Alta Disponibilitat: Replicar Servidor LDAP

La replicació del Servidor LDAP permet mantenir dos o més servidors sincronitzats amb el mateix directori. Si el servidor principal cau, els clients continuen autenticant-se contra el secundari sense interrupció del servei. És l'equivalent directe al DHCP Failover vist al mòdul 0375 de serveis de xarxa.

OpenLDAP utilitza el mecanisme **syncrepl** (Synchronization Replication). El servidor secundari es subscriu als canvis del primari i els replica automàticament.

**Avantatges:**

- Redundància (Alta disponibilitat): Si el servidor primari cau, el secundari continua donant servei d'autenticació.
- Distribució de càrrega de lectura: Els clients poden consultar qualsevol dels dos servidors.
- Manteniment sense interrupció: Es pot actualitzar el servidor primari sense tallar el servei.

**Modes de replicació:**

- **Master-Slave (Provider-Consumer)**: El primari accepta escriptures i el secundari replica en mode lectura. És el mode més senzill i habitual.
- **Multi-Master**: Ambdós servidors accepten escriptures i es sincronitzen mútuament. Més complex i pot generar conflictes.

En aquest apartat configurarem el mode **Provider-Consumer** (Master-Slave), el més recomanat per la majoria d'entorns.

### Preparació: Compte de Replicació

Al servidor primari cal crear un compte específic que el secundari utilitzarà per connectar-se i llegir els canvis:

```bash
nano compte-replicacio.ldif
```

```ldif
dn: cn=replicador,dc=educem,dc=cat
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: replicador
description: Compte de servei per a la replicació LDAP
userPassword: {SSHA}z6r+3GqL3gk0y+qz+2K1QmGZk6iXQnLr
```

A `userPassword` hi va el hash que retorna `slappasswd` (fitxer 05). La contrasenya en text pla es farà servir a la configuració del secundari (`credentials`).

```bash
ldapadd -x -D "cn=admin,dc=educem,dc=cat" -W -f compte-replicacio.ldif
```

**Donar permisos de lectura total al compte de replicació** (necessita també poder les contrasenyes per replicar-les):

```bash
nano acl-replicacio.ldif
```

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
# El replicador pot llegir-ho tot, incloses les contrasenyes
olcAccess: to attrs=userPassword
  by self write
  by dn="cn=admin,dc=educem,dc=cat" write
  by dn="cn=replicador,dc=educem,dc=cat" read
  by anonymous auth
  by * none
olcAccess: to attrs=shadowLastChange,shadowExpire
  by self write
  by dn="cn=admin,dc=educem,dc=cat" write
  by users read
  by * none
olcAccess: to *
  by dn="cn=admin,dc=educem,dc=cat" write
  by users read
  by * none
```

Són les ACL del fitxer 07 amb una línia més per al replicador. Com que `replace` substitueix totes les regles, cal escriure-les totes. Per a la resta d'atributs, el replicador ja té permís de lectura amb `by users read`.

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f acl-replicacio.ldif
```

### Configurar el Servidor Primari (Provider)

Cal carregar el mòdul `syncprov` i configurar la base de dades per emetre canvis:

```bash
nano syncprov-module.ldif
```

```ldif
# Carregar el mòdul syncprov
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: syncprov
```

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f syncprov-module.ldif
```

```bash
nano syncprov-config.ldif
```

```ldif
# Activar syncprov a la base de dades principal
dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
# Punt de control cada 100 operacions o cada 10 minuts
olcSpCheckpoint: 100 10
# Historial de les últimes 100 operacions
olcSpSessionLog: 100
```

En LDIF, els comentaris han d'anar en una línia a part: un comentari al final d'una línia es consideraria part del valor i l'ordre donaria error.

```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov-config.ldif
```

**Habilitar el Server ID** (necessari per identificar el servidor a la replicació):

```bash
nano serverid.ldif
```

```ldif
dn: cn=config
changetype: modify
add: olcServerID
olcServerID: 1 ldap://192.168.1.1
```

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f serverid.ldif
```

### Configurar el Servidor Secundari (Consumer)

Al servidor secundari, cal configurar `syncrepl` a la base de dades per subscriure's als canvis del primari:

```bash
nano syncrepl-consumer.ldif
```

Aquí posarem el servidor que fem servir pel mòdul 0375 amb IP: 192.168.1.2.

```ldif
# Configurar Server ID del secundari
dn: cn=config
changetype: modify
add: olcServerID
olcServerID: 2 ldap://192.168.1.2

# Configurar la replicació a la base de dades
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSyncRepl
olcSyncRepl: rid=001
  provider=ldap://192.168.1.1
  bindmethod=simple
  binddn="cn=replicador,dc=educem,dc=cat"
  credentials=contrasenya_del_replicador
  searchbase="dc=educem,dc=cat"
  attrs="*,+"
  schemachecking=on
  type=refreshAndPersist
  retry="60 10 300 +"
```

El fitxer conté dos canvis separats per una línia en blanc. A `credentials` hi va la contrasenya **en text pla** del compte `replicador` (no el hash): el secundari la fa servir per autenticar-se al primari. Queda desada a `cn=config`, que només pot llegir root.

**Paràmetres de `syncrepl` explicats:**

| Paràmetre  | Valor               | Descripció                                                |
| ---------- | ------------------- | --------------------------------------------------------- |
| `rid`      | `001`               | ID de la relació de replicació (número únic)              |
| `provider` | `ldap://...`        | Adreça del servidor primari                               |
| `type`     | `refreshAndPersist` | Sincronitza i manté connexió oberta                       |
| `retry`    | `"60 10 300 +"`     | Si falla: reintenta cada 60s (10 cops), després cada 300s |
| `interval` | `00:00:05:00`       | Només amb `type=refreshOnly`: cada quant es sincronitza (amb `refreshAndPersist` no cal) |
| `attrs`    | `"*,+"`             | Replica tots els atributs (operacionals inclosos)         |

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f syncrepl-consumer.ldif
```

**Les ACL no es repliquen:** són a `cn=config`, i el que es replica són les dades de `dc=educem,dc=cat`. Cal aplicar al secundari les mateixes ACL que al primari (fitxer 07); si no, el secundari permetria llegir el directori de manera anònima. Per això les consultes de comprovació es fan amb un usuari autenticat (`-D ... -W`).

### Iniciar i Verificar la Replicació

```bash
# ALS DOS SERVIDORS: reiniciar slapd
sudo systemctl restart slapd

# Verificar l'estat als logs
sudo journalctl -u slapd -f

# Al secundari, hauries de veure:
# syncrepl_message_to_op: rid=001 LDAP_RES_SEARCH_ENTRY
# do_syncrep2: rid=001 CSN ...
```

**Verificar que el secundari té les dades:**

```bash
# Al servidor secundari, consultar el directori (amb l'administrador del secundari)
ldapsearch -x -H ldap://192.168.1.2 -D "cn=admin,dc=educem,dc=cat" -W -b "dc=educem,dc=cat" "(objectClass=posixAccount)" uid cn

# Ha de retornar els mateixos usuaris que el primari
```

**Provar la replicació en temps real:**

```bash
# TERMINAL 1 - Al secundari, monitorar els logs
sudo journalctl -u slapd -f

# TERMINAL 2 - Al primari, crear un usuari nou
ldapadd -x -D "cn=admin,dc=educem,dc=cat" -W << EOF
dn: uid=test-replica,ou=usuaris,dc=educem,dc=cat
objectClass: inetOrgPerson
objectClass: posixAccount
cn: Test Replica
sn: Replica
uid: test-replica
uidNumber: 99999
gidNumber: 99999
homeDirectory: /home/test-replica
loginShell: /bin/bash
EOF

# Al secundari, verificar que l'usuari ha aparegut
ldapsearch -x -H ldap://192.168.1.2 -D "cn=admin,dc=educem,dc=cat" -W -b "dc=educem,dc=cat" "(uid=test-replica)"
```

El secundari és **només de lectura**: si s'hi intenta afegir o modificar una entrada, respon amb una referència al primari (`referral`) o un error. Tots els canvis es fan al primari.

### Simular una Fallada del Primari

```bash
# Al servidor primari, aturar slapd
sudo systemctl stop slapd

# Des d'un client, verificar que l'autenticació segueix funcionant
# (el client ha d'estar configurat per consultar ambdós servidors)
ssh pep@IP_CLIENT   # Ha de seguir funcionant via secundari

# Verificar logs al secundari: missatges similars a aquests
sudo journalctl -u slapd -f
# slap_client_connect: URI=ldap://192.168.1.1 ... ldap_sasl_bind_s failed (-1)
# do_syncrepl: rid=001 rc -1 retrying
```

Quan el primari torna a estar en marxa, el secundari s'hi reconnecta sol (segons el paràmetre `retry`) i recupera els canvis pendents.

**Configurar el client per utilitzar ambdós servidors** (`/etc/nslcd.conf`):

```bash
# Especificar ambdós servidors LDAP (el client provarà el primer i si falla, el segon)
uri ldap://192.168.1.1 ldap://192.168.1.2
```

### Monitorar la Replicació

```bash
# Veure l'estat de la replicació al secundari
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" "(objectClass=olcSyncReplConfig)" olcSyncRepl

# Veure el CSN (Context Sequence Number) - indica fins on s'ha replicat
ldapsearch -x -H ldap://192.168.1.2 -D "cn=admin,dc=educem,dc=cat" -W -b "dc=educem,dc=cat" -s base contextCSN

# Comparar el CSN entre primari i secundari (han de coincidir)
ldapsearch -x -H ldap://192.168.1.1 -D "cn=admin,dc=educem,dc=cat" -W -b "dc=educem,dc=cat" -s base contextCSN
```

| Estat als logs      | Significat                                               |
| ------------------- | -------------------------------------------------------- |
| `refreshAndPersist` | Connexió activa i sincronitzada                          |
| `connection lost`   | El primari no és accessible                              |
| `retrying`          | Intentant reconnectar                                    |
| `CSN too old`       | El secundari s'ha quedat molt enrere; cal resincronitzar |

**Resincronització manual** (si el secundari s'ha quedat molt enrere):

```bash
# Al secundari: aturar slapd, esborrar la BD i reimportar del primari
sudo systemctl stop slapd

# Exportar la BD del primari
sudo slapcat -v -l /tmp/sync-primari.ldif  # Executar al PRIMARI

# Copiar el fitxer al secundari i importar
sudo rm -f /var/lib/ldap/*
sudo slapadd -v -l /tmp/sync-primari.ldif
sudo chown -R openldap:openldap /var/lib/ldap

sudo systemctl start slapd
# La replicació incremental continuarà automàticament des d'aquí
```
