# 04 ADMINISTRACIÓ REMOTA - Administració Remota i Segura de Sistemes

> Escenari del laboratori: [00_RECURSOS/01_escenari_laboratori.md](../00_RECURSOS/01_escenari_laboratori.md)

**RA4:** Administra de forma remota el sistema operatiu en xarxa valorant-ne la seva importància i aplicant-hi criteris de seguretat.

La connexió bàsica per SSH (MobaXterm, claus i desactivació de l'accés amb contrasenya) es fa a l'inici del curs: [00_RECURSOS/02_connexio_ssh.md](../00_RECURSOS/02_connexio_ssh.md). Aquest tema en parteix i amplia l'administració remota.

## Índex de continguts

1. [Introducció a l'Administració Remota](./01_intro_administracio_remota.md)
2. [Configuració Avançada del Servidor i del Client SSH](./02_configuracio_ssh.md)
3. [Gestió de Claus i Agent SSH](./03_claus_agent.md)
4. [Transferència de Fitxers per SSH](./04_transferencia_fitxers.md)
5. [Salts entre Servidors i Túnels SSH](./05_tunels_ssh.md)
6. [Bastionat de l'SSH](./06_bastionat_ssh.md)
7. [Escriptori Remot: RDP i VNC](./07_escriptori_remot.md)
8. [Administrar Diversos Servidors Alhora](./08_diversos_servidors.md)

L'administració web amb Cockpit es veu al [Tema 3, fitxer 07](../03_AUTOMATITZACIO_TASQUES/07_eines_grafiques.md).

## Relació amb Altres Temes

- **Tema 2 (Gestió de la informació):** consultar la informació i els registres de servidors remots; detectar intents d'accés.
- **Tema 3 (Automatització de tasques):** claus SSH per a les còpies de seguretat desateses.
- **Tema 7 (Scripting):** scripts que administren diversos servidors per SSH.
- **Mòdul 0375 (Transferència de fitxers):** SFTP, gàbies `chroot` i claus per a desplegaments.
