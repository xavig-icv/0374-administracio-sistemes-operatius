## 01. Introducció a l'Administració Remota

### Per què Administrar a Distància?

Els servidors d'una empresa no tenen pantalla ni teclat: són en un centre de dades (CPD), en un armari de comunicacions o al núvol. L'administrador hi treballa **sempre a distància**, des del seu ordinador. Administrar remotament vol dir poder fer-hi qualsevol tasca (configurar serveis, consultar registres, instal·lar actualitzacions) com si s'hi estigués al davant, però **sense posar en risc la seguretat**: l'accés remot és també la porta d'entrada que busquen els atacants.

### Tipus d'Accés Remot

| Tipus                        | Protocol / eina                          | Port      | Ús habitual                                         |
| ---------------------------- | ---------------------------------------- | --------- | --------------------------------------------------- |
| Terminal                     | **SSH** (OpenSSH, MobaXterm, PuTTY)      | 22        | Administració de servidors Linux. El més habitual   |
| Escriptori gràfic            | **RDP** (Escriptori remot de Windows, xrdp) | 3389   | Equips Windows i escriptoris Linux                  |
| Escriptori gràfic            | **VNC** (TigerVNC, x11vnc)               | 5900+     | Compartir la pantalla de qualsevol sistema          |
| Consola web                  | Cockpit, Webmin                          | 9090, 10000 | Administració des del navegador                   |
| Fora de banda                | iLO, iDRAC, IPMI, consola de l'hipervisor | —        | Accedir a l'equip encara que el sistema no arrenqui |
| Automatització               | Ansible (per SSH)                        | 22        | Aplicar la mateixa configuració a molts servidors   |

L'accés **fora de banda** (*out-of-band*) és el que fa servir l'administrador quan el sistema operatiu no respon: una targeta de gestió del servidor físic o, en una màquina virtual, la consola de l'hipervisor (la finestra de VirtualBox).

### Criteris de Seguretat

| Criteri                     | Com s'aplica                                                                       |
| --------------------------- | ---------------------------------------------------------------------------------- |
| **Xifrat**                  | Tot el trànsit ha d'anar xifrat: SSH, RDP amb TLS. VNC i Telnet no xifren: cal un túnel |
| **Autenticació forta**      | Claus en lloc de contrasenyes, sense accés directe de `root`, segon factor si cal  |
| **Mínima exposició**        | Obrir només els ports necessaris i només per a les xarxes que els necessiten (tallafoc, VPN) |
| **Mínim privilegi**         | Cada administrador amb el seu usuari i `sudo`; comptes de servei limitats          |
| **Registre i auditoria**    | Saber qui ha entrat, des d'on i què ha fet (Tema 2, fitxers 05 i 11)               |

### L'Accés Remot a l'Escenari del Laboratori

```
 Ordinador de l'aula ── SSH ──► srv0374 (10.0.1.X, enp0s3) ── xarxa interna 192.168.1.0/24 ──┬─► servidor 0375 (192.168.1.2)
   (MobaXterm, ssh)                  gateway i servidor LDAP                                    └─► client Ubuntu (192.168.1.50)
```

- La connexió bàsica per SSH (claus i desactivació de les contrasenyes) es fa a l'inici del curs: [00_RECURSOS/02_connexio_ssh.md](../00_RECURSOS/02_connexio_ssh.md).
- El client de la xarxa interna no és accessible directament des de l'aula: s'hi arriba **saltant** a través del servidor 0374 (fitxer 05).

### Continguts del Tema

| Fitxer                                                   | Contingut                                                      |
| -------------------------------------------------------- | -------------------------------------------------------------- |
| [02](./02_configuracio_ssh.md)                           | Configuració avançada del servidor i del client SSH            |
| [03](./03_claus_agent.md)                                | Gestió de claus, agent SSH i claus dels servidors              |
| [04](./04_transferencia_fitxers.md)                      | Transferència de fitxers: `scp`, `sftp`, `rsync` i `sshfs`     |
| [05](./05_tunels_ssh.md)                                 | Salts entre servidors i túnels SSH                             |
| [06](./06_bastionat_ssh.md)                              | Bastionat de l'SSH: tallafoc, `fail2ban` i auditoria           |
| [07](./07_escriptori_remot.md)                           | Escriptori remot: RDP i VNC                                    |
| [08](./08_diversos_servidors.md)                         | Administrar diversos servidors alhora: scripts i Ansible       |
