# 06 INTEGRACIÓ DE SISTEMES - Interoperabilitat entre Sistemes Lliures i Propietaris

> Escenari del laboratori: [00_RECURSOS/01_escenari_laboratori.md](../00_RECURSOS/01_escenari_laboratori.md)

**RA6:** Integra sistemes operatius lliures i propietaris, justificant i garantint-ne la interoperabilitat.

`Nota:` Tema en preparació. Aquest és l'índex previst.

## Continguts Previstos

1. Introducció a la interoperabilitat: protocols SMB/CIFS i NFS, dominis i directoris
2. Samba: instal·lació i configuració bàsica (`smb.conf`)
3. Recursos compartits, usuaris de Samba i permisos
4. Accés als recursos des de Windows i des de Linux (`smbclient`, `mount -t cifs`)
5. Usuaris centralitzats: Samba com a controlador de domini Active Directory i integració amb el directori LDAP (Tema 1)
6. NFS: recursos compartits entre sistemes Linux
7. Unir un equip Linux a un domini (`realmd`, `sssd`)
8. Pràctica transversal amb el mòdul 0375: GLPI amb autenticació LDAP, servidor web i FTP

## Relació amb Altres Temes

- **Tema 1 (Serveis de directori):** usuaris i grups centralitzats a LDAP.
- **Tema 5 (Servidors d'impressió):** impressores compartides amb clients Windows.
- **Mòdul 0375:** serveis web i FTP de la pràctica transversal.
