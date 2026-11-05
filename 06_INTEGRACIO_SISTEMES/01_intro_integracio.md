## 01. Introducció a la Integració de Sistemes

### El Problema de la Interoperabilitat

En una empresa conviuen sistemes **lliures** (servidors Linux, escriptoris Ubuntu) i **propietaris** (equips Windows, macOS, impressores, aplicacions). Cada sistema té les seves pròpies maneres de compartir fitxers, de gestionar usuaris i permisos i d'autenticar-se. **Integrar** vol dir aconseguir que treballin junts: que un usuari pugui entrar a qualsevol equip amb el mateix compte, obrir els mateixos fitxers des de Windows i des de Linux, imprimir a les mateixes impressores i fer servir les aplicacions amb les mateixes credencials.

| Necessitat                                   | Problema sense integració                               | Solució                                 |
| -------------------------------------------- | ------------------------------------------------------- | --------------------------------------- |
| Compartir fitxers entre Windows i Linux      | Protocols diferents (SMB a Windows, NFS a Linux)        | Samba (SMB a Linux) i NFS               |
| Un sol usuari i contrasenya a tota l'empresa | Un compte diferent a cada equip i a cada aplicació      | Directori centralitzat (LDAP, Active Directory) |
| Permisos coherents                           | Els usuaris tenen identificadors diferents a cada equip | UID/GID del directori, ACL              |
| Aplicacions web amb els usuaris de l'empresa | Cada aplicació té la seva pròpia base d'usuaris         | Autenticació contra el directori (GLPI amb LDAP) |

### Protocols d'Integració

| Protocol            | Funció                                              | Ports             | Sistemes                     |
| ------------------- | --------------------------------------------------- | ----------------- | ---------------------------- |
| **SMB/CIFS**        | Compartir fitxers i impressores                     | 445 (139)         | Windows; Linux amb Samba; macOS |
| **NFS**             | Compartir fitxers                                   | 2049              | Linux i UNIX                 |
| **LDAP / LDAPS**    | Directori d'usuaris i grups                         | 389 / 636         | Tots (Tema 1)                |
| **Kerberos**        | Autenticació amb tiquets (base d'Active Directory)  | 88                | Windows (AD), Linux          |
| **IPP**             | Impressió per xarxa                                 | 631               | Tots (Tema 5)                |
| **SSH / SFTP**      | Administració i transferència de fitxers            | 22                | Tots (Tema 4)                |

### Integració a l'Escenari del Laboratori

```
                     ┌──────────────────────────────────────────────┐
  Ordinador Windows  │ srv0374 (10.0.1.37 i 192.168.1.1)            │
  (xarxa del centre) │  · OpenLDAP: usuaris i grups (Tema 1)        │
   \\srv0374\... ───►│  · Samba: carpetes compartides per a Windows │
                     │  · NFS: carpetes i directoris personals      │
                     └───────────────┬──────────────────────────────┘
                                     │ xarxa interna 192.168.1.0/24
              ┌──────────────────────┴─────────────────────┐
     Client Ubuntu (192.168.1.50)              Servidor 0375 (192.168.1.2)
      · usuaris LDAP (nslcd)                     · GLPI (web) amb usuaris LDAP
      · directoris personals per NFS             · correu, FTP (mòdul 0375)
```

### Continguts del Tema

| Fitxer                                   | Contingut                                                            |
| ---------------------------------------- | -------------------------------------------------------------------- |
| [02](./02_samba_installacio.md)          | Instal·lació i configuració bàsica de Samba                          |
| [03](./03_samba_recursos_permisos.md)    | Recursos compartits, usuaris de Samba i permisos                     |
| [04](./04_clients_smb.md)                | Accés als recursos des de Windows i des de Linux                     |
| [05](./05_nfs.md)                        | NFS i directoris personals compartits per als usuaris LDAP           |
| [06](./06_dominis_ad.md)                 | Dominis Windows: Active Directory, Samba AD i unir equips Linux      |
| [07](./07_aplicacions_ldap.md)           | Aplicacions integrades amb el directori: GLPI                        |
