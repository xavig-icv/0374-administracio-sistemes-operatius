# 03 AUTOMATITZACIÓ DE TASQUES - Planificació i Execució Automàtica de Tasques

> Escenari del laboratori: [00_RECURSOS/01_escenari_laboratori.md](../00_RECURSOS/01_escenari_laboratori.md)

**RA3:** Gestiona l'automatització de tasques del sistema, aplicant-hi criteris d'eficiència i utilitzant comandes i eines gràfiques.

`Nota:` Tema en preparació. Aquest és l'índex previst.

## Continguts Previstos

1. Introducció a l'automatització: què convé automatitzar i com comprovar-ne el resultat
2. Tasques puntuals amb `at` i `batch`
3. Tasques periòdiques amb `cron`: `crontab` d'usuari, `/etc/crontab` i `/etc/cron.d/`
4. Els directoris `cron.daily`, `cron.weekly`... i `anacron` per als equips que no estan sempre engegats
5. Temporitzadors de systemd (`.timer`): l'alternativa moderna a `cron`
6. Rotació de registres amb `logrotate`
7. Còpies de seguretat automàtiques amb `tar` i `rsync`
8. Programar els scripts d'informe del Tema 2 (`informe-srv0374.sh`, `estat-srv0374.sh`) i registrar-ne el resultat amb `logger`
9. Eines gràfiques de planificació (Cockpit, Webmin)
10. Bones pràctiques i resolució de problemes: entorn i `PATH` de `cron`, permisos, registres

## Relació amb Altres Temes

- **Tema 2 (Gestió de la informació):** scripts d'informe, serveis de systemd, `logger` i registres.
- **Tema 7 (Scripting):** scripts més complets per automatitzar tasques d'administració.
