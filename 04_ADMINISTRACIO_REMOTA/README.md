# 04 ADMINISTRACIÓ REMOTA - Administració Remota i Segura de Sistemes

> Escenari del laboratori: [00_RECURSOS/01_escenari_laboratori.md](../00_RECURSOS/01_escenari_laboratori.md)

**RA4:** Administra de forma remota el sistema operatiu en xarxa valorant-ne la seva importància i aplicant-hi criteris de seguretat.

La connexió bàsica per SSH (MobaXterm, claus i desactivació de l'accés amb contrasenya) es fa a l'inici del curs: [00_RECURSOS/02_connexio_ssh.md](../00_RECURSOS/02_connexio_ssh.md). Aquest tema en parteix i amplia l'administració remota.

`Nota:` Tema en preparació. Aquest és l'índex previst.

## Continguts Previstos

1. Introducció a l'administració remota: SSH, escriptori remot i eines web
2. Configuració avançada del servidor OpenSSH: port, `AllowUsers`, límits d'intents i activació per socket a Ubuntu 24.04
3. Gestió de claus: `ssh-agent`, diverses claus i revocació
4. Configuració del client (`~/.ssh/config`) i salts entre servidors (`ProxyJump`)
5. Còpia de fitxers: `scp`, `sftp` i `rsync`
6. Túnels SSH: reenviament de ports local i remot
7. Bastionat de l'SSH: tallafoc i `fail2ban`
8. Escriptori remot: VNC i RDP (`xrdp`) amb clients Windows i Linux
9. Administració web amb Cockpit
10. Administrar diversos servidors alhora (introducció a Ansible)

## Relació amb Altres Temes

- **Tema 2 (Gestió de la informació):** consultar la informació i els registres de servidors remots.
- **Tema 3 (Automatització de tasques):** claus SSH per a les còpies de seguretat desateses.
- **Mòdul 0375 (Transferència de fitxers):** SFTP, gàbies `chroot` i claus per a desplegaments.
