## 07. Escriptori Remot: RDP i VNC

Els servidors s'administren per terminal, però de vegades cal veure un **escriptori gràfic** remot: donar suport a un usuari, administrar un equip Windows o fer servir una aplicació que només té interfície gràfica.

| Protocol | Port         | Xifrat                    | Ús habitual                                                   |
| -------- | ------------ | ------------------------- | ------------------------------------------------------------- |
| **RDP**  | 3389         | Sí (TLS)                  | Escriptori remot de Windows; també Linux (`xrdp`, GNOME)      |
| **VNC**  | 5900 + pantalla | No, en la majoria de servidors | Compartir la pantalla de qualsevol sistema operatiu      |

**Clients:** l'*Escriptori remot* de Windows (`mstsc`), Remmina (Linux, RDP i VNC) i MobaXterm (sessions RDP i VNC).

### RDP a Windows

Als equips Windows (edicions Pro, Enterprise i Server): *Configuració* → *Sistema* → *Escriptori remot* → activar-lo. Des d'un altre equip: `mstsc` → adreça de l'equip → usuari i contrasenya. Cal mantenir activada l'**autenticació a nivell de xarxa** (NLA): l'usuari s'autentica abans que s'obri la sessió gràfica.

### RDP a l'Escriptori d'Ubuntu (client de l'escenari)

Ubuntu Desktop 24.04 inclou un servidor RDP: *Configuració* → *Sistema* → *Escriptori remot*.

| Opció                        | Funció                                                                     |
| ---------------------------- | -------------------------------------------------------------------------- |
| *Compartició de l'escriptori* | Comparteix la sessió que l'usuari té oberta (com un suport remot)         |
| *Inici de sessió remot*       | Permet iniciar una sessió nova encara que ningú no hagi iniciat sessió a l'equip |

S'hi defineixen un usuari i una contrasenya específics per a l'RDP. Com que el client és a la xarxa interna, des de l'ordinador de l'aula s'hi arriba amb un túnel a través del servidor 0374 (fitxer 05):

```bash
ssh -N -L 13389:192.168.1.50:3389 srv0374
```

I a l'escriptori remot de Windows: `localhost:13389`.

**Alternativa per a altres escriptoris Linux: `xrdp`**

```bash
sudo apt install xrdp
sudo systemctl enable --now xrdp
sudo ss -tlnp | grep 3389
```

`xrdp` obre una sessió gràfica nova per a cada usuari que s'hi connecta. Amb GNOME pot donar problemes si l'usuari també té la sessió oberta a la pantalla; amb escriptoris lleugers (XFCE) funciona millor.

### VNC

VNC transmet la imatge de la pantalla i els moviments del ratolí. La majoria de servidors VNC **no xifren** el trànsit (ni la contrasenya de manera segura): s'han de fer servir sempre dins d'un túnel SSH.

```bash
# Al client Ubuntu: compartir la pantalla actual amb x11vnc, només a localhost
sudo apt install x11vnc
x11vnc -localhost -display :0 -usepw

# A l'ordinador: túnel fins al port 5900 del client, passant pel 0374
ssh -N -L 5900:localhost:5900 -J srv0374 alumne@192.168.1.50
```

I el client VNC es connecta a `localhost:5900`. Amb l'opció `-localhost`, el servidor VNC només accepta connexions del mateix equip: l'únic camí per arribar-hi és el túnel SSH.

`Nota:` `x11vnc` només funciona amb sessions gràfiques X11. Ubuntu 24.04 fa servir Wayland per defecte: cal escollir *Ubuntu on Xorg* a la pantalla d'inici de sessió, o bé fer servir l'RDP integrat, que sí que funciona amb Wayland.

### Seguretat

- **No exposeu mai els ports 3389 ni 5900 a Internet:** són dels més atacats. L'accés ha de passar per una VPN o un túnel SSH.
- Comptes amb contrasenyes robustes i, a Windows, l'NLA activada.
- Avisar l'usuari abans de connectar-se a la seva sessió (protecció de dades): les eines de suport mostren un avís o demanen permís.

### Resolució de Problemes

| Problema                                                        | Causa                                                      | Solució                                                     |
| --------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------- |
| `mstsc` no es connecta a `localhost:3389`                       | Windows no permet connectar-se al seu propi port 3389      | Túnel amb un altre port local (`13389`)                     |
| La sessió `xrdp` es tanca just després d'iniciar-la             | L'usuari ja té una sessió gràfica oberta a la pantalla     | Tancar la sessió local o fer servir un escriptori lleuger   |
| El client VNC diu `connection refused`                          | El servidor VNC només escolta a `localhost` (correcte) i no s'ha fet el túnel | Crear el túnel SSH abans de connectar-se       |
| La imatge va molt lenta                                         | Molta resolució o efectes gràfics                          | Reduir la resolució i la profunditat de color al client     |
