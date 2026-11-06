## 06. Dominis Windows: Active Directory, Samba AD i Equips Linux

### Active Directory

A les empreses amb equips Windows, els usuaris i els equips es gestionen amb **Active Directory** (AD), el servei de directori de Microsoft. Un **domini** AD reuneix:

| Component                | Funció                                                                    |
| ------------------------ | ------------------------------------------------------------------------- |
| Controlador de domini (DC) | Servidor que té la base de dades del domini i autentica usuaris i equips |
| LDAP                     | Directori d'usuaris, grups, equips i unitats organitzatives (com al Tema 1) |
| Kerberos                 | Autenticació amb tiquets: l'usuari introdueix la contrasenya una vegada i accedeix a tots els serveis |
| DNS                      | Els equips troben els controladors del domini amb registres SRV           |
| Directives de grup (GPO) | Configuració aplicada automàticament als equips i usuaris del domini      |

La diferència amb el directori OpenLDAP del Tema 1 és que AD és un conjunt integrat: directori, autenticació Kerberos, DNS i directives, que els equips Windows saben fer servir directament.

### Samba com a Controlador de Domini

**Samba AD DC** permet crear un domini compatible amb Active Directory en un servidor Linux: els equips Windows s'hi poden unir i administrar-lo amb les eines de Microsoft (RSAT).

`Nota:` Un controlador de domini Samba AD inclou el seu propi LDAP (port 389), Kerberos (88) i DNS (53). **No pot conviure** amb OpenLDAP (Tema 1) ni amb el servidor DNS BIND (mòdul 0375) al mateix servidor. Al laboratori cal una **màquina virtual dedicada**, per exemple `192.168.1.4` (la `192.168.1.3` és la màquina de Jitsi del mòdul 0375; vegeu el pla d'adreces de l'[escenari](../00_RECURSOS/01_escenari_laboratori.md)).

Esquema dels passos (en una VM dedicada amb Ubuntu Server, nom `dc1` i IP fixa):

```bash
sudo apt install samba winbind krb5-user smbclient
sudo systemctl disable --now smbd nmbd winbind        # un DC fa servir el servei samba-ad-dc
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo samba-tool domain provision --use-rfc2307 --interactive
#   Realm: AD.EDUCEM.CAT    Domain: EDUCEM    Server Role: dc    DNS backend: SAMBA_INTERNAL
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
```

```bash
sudo samba-tool domain level show                     # comprovar el domini
sudo samba-tool user create anna                      # crear un usuari
kinit administrator@AD.EDUCEM.CAT && klist            # obtenir i veure un tiquet de Kerberos
```

El domini fa servir un subdomini (`ad.educem.cat`) per no barrejar-se amb el domini DNS de l'empresa. Els equips del domini han de fer servir el DC com a servidor DNS.

### Unir un Equip Windows al Domini

*Configuració* → *Sistema* → *Quant a* → *Domini o grup de treball* → *Canvia* → *Domini*: `AD.EDUCEM.CAT`, amb l'usuari `Administrator`. Prèviament, el servidor DNS de l'equip Windows ha de ser el controlador de domini.

### Unir un Equip Linux a un Domini: `realmd` i `sssd`

Un equip Ubuntu es pot unir a un domini AD (de Microsoft o de Samba) perquè els usuaris del domini hi puguin iniciar sessió:

```bash
sudo apt install realmd sssd sssd-tools adcli krb5-user samba-common-bin packagekit
realm discover ad.educem.cat                          # informació del domini
sudo realm join -U administrator ad.educem.cat        # unir l'equip
realm list                                            # comprovar-ho
id anna@ad.educem.cat                                 # usuari del domini
sudo pam-auth-update --enable mkhomedir               # crear el directori personal en el primer inici
```

`sssd` fa per als dominis AD el mateix paper que `nslcd` per a OpenLDAP (Tema 1, fitxer 06): resol els usuaris i grups i autentica els inicis de sessió.

### OpenLDAP o Active Directory?

| Aspecte                       | OpenLDAP (Tema 1)                         | Active Directory / Samba AD                 |
| ----------------------------- | ----------------------------------------- | ------------------------------------------- |
| Clients Windows               | Calen eines addicionals                   | Integració nativa (unir l'equip al domini)  |
| Clients Linux                 | `nslcd` o `sssd`                          | `sssd` + `realmd`                           |
| Aplicacions (GLPI, Moodle...) | Sí, per LDAP                              | Sí, per LDAP                                |
| Directives de grup (GPO)      | No                                        | Sí                                          |
| Complexitat                   | Menor; molt flexible                      | Més gran; molt estàndard a les empreses     |

### Resolució de Problemes

| Problema                                                  | Causa                                                       | Solució                                                     |
| --------------------------------------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------- |
| `samba-tool domain provision` falla per ports en ús       | Hi ha altres serveis al port 53, 88 o 389 (BIND, slapd)     | Fer-ho en una VM dedicada                                   |
| L'equip no troba el domini                                | El seu DNS no és el controlador de domini                   | Configurar el DC com a servidor DNS de l'equip              |
| `kinit: Clock skew too great`                             | Diferència d'hora de més de 5 minuts entre equips           | Sincronitzar l'hora (NTP) a tots els equips                 |
| `realm join` falla amb `Couldn't join realm`              | Falten paquets, nom de domini incorrecte o credencials      | `realm discover` primer i revisar `journalctl -u realmd`    |
