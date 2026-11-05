## 05. Salts entre Servidors i Túnels SSH

### Saltar a través d'un Servidor: `ProxyJump`

A l'escenari, el client Ubuntu (`192.168.1.50`) només és a la xarxa interna: des de l'ordinador de l'aula no s'hi pot arribar directament. El servidor 0374 sí que hi té accés, i es pot fer servir de **servidor de salt** (*jump host* o *bastion*):

```
Ordinador ──SSH──► srv0374 (10.0.1.37) ──SSH──► client (192.168.1.50)
```

```bash
ssh -J xavi@10.0.1.37 alumne@192.168.1.50
```

Amb `-J` (o `ProxyJump` al fitxer `~/.ssh/config`, fitxer 02), la connexió amb el destí es xifra de punta a punta: el servidor 0374 només reenvia les dades i les claus privades no surten de l'ordinador. En empreses és habitual que només un servidor de salt sigui accessible des de fora i que tots els altres només acceptin connexions SSH que hi passin.

### Túnels: Reenviament de Ports

Un **túnel SSH** transporta una altra connexió TCP dins de la connexió SSH, xifrada. Serveix per accedir a serveis que no són accessibles directament o que no xifren el trànsit.

| Tipus       | Opció                          | Què fa                                                                  |
| ----------- | ------------------------------ | ----------------------------------------------------------------------- |
| **Local**   | `-L port_local:destí:port`     | Un port de l'**ordinador** porta a un servei que veu el **servidor**    |
| **Remot**   | `-R port_remot:destí:port`     | Un port del **servidor** porta a un servei que veu l'**ordinador**      |
| **Dinàmic** | `-D port`                      | Un servidor intermediari SOCKS a l'ordinador: tot el que hi passi surt pel servidor |

Opcions útils: `-N` (no obre cap terminal, només el túnel) i `-f` (passa a segon pla).

### Túnel Local (`-L`)

**Exemple 1: un servei que només escolta a `localhost` del servidor.** La base de dades de GLPI del servidor 0375 (MariaDB, port 3306) no és accessible per la xarxa. Per administrar-la des de l'ordinador amb una eina gràfica:

```bash
ssh -N -L 3306:localhost:3306 srv0375
```

Mentre el túnel està obert, connectar-se a `localhost:3306` des de l'ordinador és connectar-se a la base de dades del servidor 0375. `localhost` s'interpreta **des del servidor**.

**Exemple 2: un equip de la xarxa interna.** Per obrir l'escriptori remot del client Ubuntu (port 3389, fitxer 07) des de l'ordinador de l'aula:

```bash
ssh -N -L 13389:192.168.1.50:3389 srv0374
```

Ara l'escriptori remot es connecta a `localhost:13389`. Es fa servir el port local 13389 i no el 3389 perquè Windows no permet connectar l'escriptori remot al seu propi port 3389.

### Túnel Remot (`-R`)

El túnel remot fa el camí invers: obre un port al **servidor** que porta a un servei de l'**ordinador**. Per exemple, per mostrar a un company connectat al servidor una web que s'està desenvolupant a l'ordinador:

```bash
ssh -N -R 8080:localhost:80 srv0374     # al servidor, localhost:8080 porta al port 80 de l'ordinador
```

És la tècnica que fan servir algunes eines de suport remot per "sortir" des de dins d'una xarxa protegida. Per això els túnels remots s'han de controlar: amb `GatewayPorts no` (valor per defecte), el port obert al servidor només és accessible des del mateix servidor.

### Túnel Dinàmic (`-D`)

```bash
ssh -N -D 1080 srv0374
```

Configurant el navegador perquè faci servir el servidor intermediari SOCKS `localhost:1080`, totes les pàgines es carreguen **a través del servidor 0374**: es poden obrir les interfícies web de la xarxa interna (per exemple, `http://192.168.1.2`) com si s'estigués a dins.

### Seguretat dels Túnels

Els túnels poden saltar-se les regles del tallafoc: un usuari amb accés SSH podria obrir camins cap a serveis que no haurien de ser accessibles. Al servidor es poden limitar:

| Directiva (`sshd_config`)   | Efecte                                                       |
| --------------------------- | ------------------------------------------------------------ |
| `AllowTcpForwarding no`     | Desactiva tots els túnels                                    |
| `AllowTcpForwarding local`  | Només túnels locals (`-L`)                                   |
| `PermitOpen 192.168.1.2:3306` | Només permet túnels cap a aquest destí                     |
| `GatewayPorts no`           | Els ports dels túnels remots no s'obren a la xarxa           |

Per als usuaris que no són administradors (per exemple, els que només fan servir SFTP), el més segur és desactivar-los amb un bloc `Match` (fitxer 02).

### Resolució de Problemes

| Problema                                                      | Causa                                                      | Solució                                                     |
| ------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------- |
| `bind [127.0.0.1]:3306: Address already in use`               | El port local ja el fa servir un altre programa            | Fer servir un altre port local: `-L 13306:localhost:3306`   |
| `channel 2: open failed: administratively prohibited`         | El servidor té els túnels desactivats (`AllowTcpForwarding`) | Revisar la configuració del servidor                      |
| `channel 2: open failed: connect failed: Connection refused`  | El servei de destí no escolta a l'adreça i el port indicats | Comprovar-ho des del servidor: `ss -tln`                    |
| El túnel funciona però es talla al cap d'una estona           | Connexió inactiva tallada per un tallafoc intermedi        | `ServerAliveInterval 60` al client                          |
| `ssh -J` demana la contrasenya del servidor de salt           | La clau no està autoritzada al servidor de salt            | Copiar la clau pública als dos servidors                    |
