## 07. Aplicacions Integrades amb el Directori: GLPI

Moltes aplicacions de les empreses (gestió d'incidències, plataformes d'aprenentatge, núvol privat, correu) poden autenticar els usuaris contra el directori LDAP. Així, cada persona entra a totes les aplicacions amb el mateix usuari i contrasenya, i quan se la dona de baixa del directori perd l'accés a totes alhora.

### Com Funciona l'Autenticació d'una Aplicació contra LDAP

```
Usuari ──(usuari i contrasenya)──► Aplicació (GLPI, servidor 0375)
                                     │ 1. Es connecta a LDAP amb el compte de servei i cerca el DN de l'usuari
                                     │ 2. Intenta autenticar-se amb aquest DN i la contrasenya de l'usuari
                                     ▼
                              Directori LDAP (srv0374)
                                     │ 3. Si la contrasenya és correcta, l'aplicació deixa entrar l'usuari
```

L'aplicació **no llegeix mai les contrasenyes** del directori: només comprova si el directori accepta la combinació d'usuari i contrasenya (Tema 1, fitxer 07: `by anonymous auth`).

### Dades que Necessita l'Aplicació

| Paràmetre                     | Valor a l'escenari                                    |
| ----------------------------- | ----------------------------------------------------- |
| Servidor i port               | `192.168.1.1`, port 389 (o `ldaps://192.168.1.1`, port 636) |
| Base DN                       | `dc=educem,dc=cat` (o `ou=usuaris,dc=educem,dc=cat`)  |
| DN i contrasenya de connexió  | Un **compte de servei de només lectura**: `cn=glpi-incidencies,dc=educem,dc=cat` |
| Filtre d'usuaris              | `(objectClass=inetOrgPerson)`                         |
| Atribut d'inici de sessió     | `uid`                                                 |
| Atributs de dades             | `mail`, `sn`, `givenName`                             |

No connecteu mai una aplicació amb el compte d'administrador del directori: si l'aplicació es veu compromesa, l'atacant obtindria el control de tot el directori. El compte de servei i les seves ACL es creen al Tema 1, fitxer 07.

**Comprovar les dades abans de configurar l'aplicació** (des del servidor de l'aplicació):

```bash
ldapsearch -x -H ldap://192.168.1.1 -D "cn=glpi-incidencies,dc=educem,dc=cat" -W \
  -b "ou=usuaris,dc=educem,dc=cat" "(uid=usuari01)" uid mail sn givenName
```

Si aquesta cerca funciona, l'aplicació també podrà fer-la amb les mateixes dades.

### Connexió Xifrada (LDAPS)

Amb LDAP sense xifrar, les contrasenyes dels usuaris viatgen en clar entre l'aplicació i el directori (Tema 1, fitxer 09). Per fer servir LDAPS, el servidor de l'aplicació ha de confiar en la CA que va signar el certificat del directori:

```bash
# Al servidor de l'aplicació (0375)
sudo cp ca.crt /usr/local/share/ca-certificates/ca-ldap-educem.crt
sudo update-ca-certificates
ldapsearch -x -H ldaps://192.168.1.1 -D "cn=glpi-incidencies,dc=educem,dc=cat" -W -b "dc=educem,dc=cat" "(uid=usuari01)"
```

El fitxer copiat a `/usr/local/share/ca-certificates/` ha de tenir l'extensió `.crt`.

### La Pràctica Transversal de GLPI

La pràctica de GLPI integra els dos mòduls: GLPI s'instal·la al servidor 0375 (servidor web i base de dades, desplegament per SFTP, notificacions per correu) i autentica els usuaris contra el directori LDAP del srv0374. És la pràctica final del bloc d'integració.

### Resolució de Problemes

| Problema                                                    | Causa                                                    | Solució                                                     |
| ----------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| La prova de connexió de l'aplicació falla                   | DN o contrasenya del compte de servei incorrectes        | Provar la mateixa cerca amb `ldapsearch`                    |
| La connexió funciona però no troba cap usuari               | Base DN o filtre incorrectes, o les ACL no deixen llegir el compte de servei | Revisar el filtre i les ACL (Tema 1, fitxer 07) |
| Els usuaris es troben però no poden entrar                  | Atribut d'inici de sessió incorrecte (`cn` en lloc d'`uid`) | Revisar l'atribut configurat                            |
| Amb LDAPS: `Can't contact LDAP server`                      | El servidor de l'aplicació no confia en la CA o la IP no és al SAN | `update-ca-certificates` i revisar el certificat (Tema 1, fitxer 09) |
