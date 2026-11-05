# 02 GESTIÓ DE LA INFORMACIÓ - Informació del Sistema, Processos, Serveis i Monitorització

> Escenari del laboratori (servidor `srv0374`, IPs, xarxa interna): [00_RECURSOS/01_escenari_laboratori.md](../00_RECURSOS/01_escenari_laboratori.md)

**RA2:** Administra processos del sistema descrivint-los i aplicant criteris de seguretat i eficiència.

## Índex de continguts

1. [Introducció a la Gestió de la Informació del Sistema](./01_intro_gestio_informacio.md)
2. [Filtrar i Processar la Informació](./02_filtrar_informacio.md)
3. [Informació del Sistema i del Maquinari](./03_sistema_hardware.md)
4. [Informació de la Xarxa](./04_info_xarxa.md)
5. [Usuaris i Sessions](./05_usuaris_sessions.md)
6. [Processos: Conceptes Bàsics](./06_processos_conceptes.md)
7. [Consultar els Processos](./07_veure_processos.md)
8. [Senyals, Prioritats i Control de Tasques](./08_senyals_prioritats.md)
9. [Serveis amb systemd](./09_serveis_systemd.md)
10. [Monitorització de Recursos](./10_monitoritzacio.md)
11. [Registres del Sistema i Diagnosi](./11_logs_diagnosi.md)

## Distribució Orientativa d'Hores (26 h)

| Fitxers    | Continguts                                                    | Hores |
| ---------- | ------------------------------------------------------------- | ----- |
| 01 i 02    | Introducció i filtratge de la informació                      | 3     |
| 03 a 05    | Informació del sistema, de la xarxa i dels usuaris            | 4     |
| 06 a 08    | Processos: conceptes, consulta, senyals i prioritats          | 5     |
| 09         | Serveis amb systemd                                           | 3     |
| 10 i 11    | Monitorització, registres i diagnosi                          | 3     |
| Pràctiques | Inventari, usuaris i seguretat, processos, serveis i monitorització | 8 |

## Pràctiques del Tema

| Pràctica                               | Fitxers del tema |
| -------------------------------------- | ---------------- |
| 1. Inventari del sistema               | 02, 03, 04       |
| 2. Usuaris, sessions i seguretat       | 02, 05           |
| 3. Processos                           | 06, 07, 08       |
| 4. Serveis i monitorització            | 09, 10, 11       |

## Relació amb Altres Temes

- **Tema 1 (Serveis de directori):** `getent passwd` també mostra els usuaris LDAP quan el servidor és client del directori.
- **Tema 3 (Automatització de tasques):** programar els scripts d'informe amb `cron` o amb temporitzadors de systemd, i rotar els registres amb `logrotate`.
- **Tema 4 (Administració remota):** obtenir la informació de servidors remots per SSH.
- **Tema 7 (Scripting):** convertir les ordres d'aquest tema en scripts complets.
