## 01. Introducció als Serveis de Directori

### Infraestrcutura de les organitzacions

Per implementar un servidor empresarial és important conèixer els conceptes clau o les diferents peces de programari que el componen i com es relacionen entre si per oferir correctament els diferents recursos i serveis als usuaris de l'organització.

Els Sistemes Operatius en Xarxa (instal·lats als servidors) permeten una administració centralitzada de la infraestructura, d'aquesta manera, els administradors poden definir polítiques d'ús per a usuaris i grups, desplegar aplicacions de forma massiva a diversos equips, gestionar actualitzacions de programari, compartir recursos (carpetes i documents) i monitoritzar l'estat de la xarxa, tant dels servidors i com dels diferents dispositius connectats (ordinadors, smartphones, impresores, switchos, NAS, punts d'accés Wi-Fi, etc.).

Amb les tecnologies de virtualització actuals és possible executar diversos servidors virtuals dins d'un únic servidor físic. En aquest context pren especial importància el concepte de redundància, que consisteix a duplicar informació o serveis per garantir la continuïtat del sistema. Al Mòdul 0375 (Serveis de Xarxa), veurem com instal·lar, configurar i assegurar la disponibilitat de serveis essencials de com l'autenticació, la resolució de noms (DNS) o l'assignació automàtica d'adreces de xarxa (DHCP), etc. Que poden executar-se simultàniament en diversos servidors per assegurar que continuïn funcionant fins i tot en el cas que un dels servidors deixi de funcionar.

### Serveis de Directori

Un servei de directori és un sistema centralitzat que emmagatzema, organitza i proporciona accés a informació sobre els recursos d'una xarxa: usuaris, grups, equips, impressores, serveis, etc.

Es pot comparar amb un "inventari" o una "base de dades" de la infraestructura d'una organització, que també permet gestionar l'autenticació i l'autorització de manera centralitzada.

Sense un servei de directori, cada servidor o aplicació hauria de gestionar els seus propis usuaris de forma independent, fet que genera duplicitats, inconsistències i una administració difícil d'escalar.

### Per a què serveix un Servei de Directori?

**Gestió centralitzada d'identitats:**

- Definir usuaris i grups en un únic lloc.
- Controlar l'accés a recursos de la xarxa (servidors, carpetes, fitxers, impressores, etc.).
- Establir polítiques de contrasenyes, permisos i accés.

**Proporciona Autenticació i Autorització:**

- **Autenticació**: Verificar que un usuari és qui diu fent ús d'un usuari i contrasenya
- **Autorització**: Determinar a quins recursos té accés l'usuari un cop autenticat.

**Escalabilitat:**

- Gestionar milers d'usuaris i equips des d'una consola o tauler de control centralitzat.
- Afegir, modificar o eliminar usuaris a tots els sistemes simultàniament (mitjançant scripts).

### Exemple d'ús reals

**Empresa amb molts treballadors i serveis:**

```
Nou empleat → l'administrador crea l'usuari al servei de directori

L'empleat, amb el mateix usuari i contrasenya, pot accedir automàticament a:
  - L'Ordinador de l'empresa (login a Linux/Windows)
  - Correu electrònic corporatiu
  - Servidor de fitxers i carpetes compartides
  - Aplicacions web internes (sistema d'incidències, Moodle intern, etc.)
  - VPN
```

### Principals Implementacions de Serveis de Directori

| Servei                   | Fabricant   | Sistema        | Protocol              |
| ------------------------ | ----------- | -------------- | --------------------- |
| **OpenLDAP**             | Open Source | Linux          | LDAP                  |
| **Active Directory**     | Microsoft   | Windows Server | LDAP + Kerberos       |
| **FreeIPA**              | Red Hat     | Linux          | LDAP + Kerberos + DNS |
| **389 Directory Server** | Red Hat     | Linux          | LDAP                  |

En aquest mòdul treballarem amb **OpenLDAP**, la implementació de referència en entorns Linux, i que és la base de moltes solucions empresarials. Active Directory de Microsoft també utilitza LDAP com a protocol de base, fet que facilita la integració entre sistemes i que treballarem al (RA6 - Integració de Sistemes Operatius).
