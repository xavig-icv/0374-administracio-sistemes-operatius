# Mòdul 0374 - Administració de Sistemes Operatius

Material de teoria del mòdul 0374 del cicle formatiu de grau superior d'Administració de Sistemes Informàtics en Xarxa (ASIX). Cada tema explica els conceptes i les ordres amb exemples interpretats. Les pràctiques es lliuren a part i l'alumnat les resol a partir d'aquesta teoria.

> **Escenari del laboratori:** tots els temes fan servir les mateixes màquines i adreces (servidor `srv0374`, xarxa interna `192.168.1.0/24`), compartides amb el mòdul 0375 - Serveis de Xarxa i Internet: [00_RECURSOS/01_escenari_laboratori.md](./00_RECURSOS/01_escenari_laboratori.md)

## Estructura del Repositori

| Carpeta                      | Continguts                                                                                                    | Hores | RA  | Estat         |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------- | ----- | --- | ------------- |
| `00_RECURSOS/`               | Escenari del laboratori i recursos comuns                                                                     | -     | -   | Disponible    |
| `01_SERVEIS_DIRECTORI/`      | Servei de directori amb OpenLDAP: instal·lació, usuaris i grups, autenticació de clients, ACL, replicació i TLS | 14    | RA1 | Disponible    |
| `02_GESTIO_INFORMACIO/`      | Informació del sistema, processos, serveis amb systemd, monitorització, registres i diagnosi                  | 26    | RA2 | Disponible    |
| `03_AUTOMATITZACIO_TASQUES/` | Planificació de tasques (`cron`, `at`, temporitzadors de systemd), rotació de registres i còpies automàtiques | 26    | RA3 | Disponible    |
| `04_ADMINISTRACIO_REMOTA/`   | SSH amb claus, transferència de fitxers, túnels, bastionat i escriptori remot                                 | 12    | RA4 | Disponible    |
| `05_SERVIDORS_IMPRESSIO/`    | Servidor d'impressió CUPS, impressores de xarxa i clients                                                     | 14    | RA5 | Disponible    |
| `06_INTEGRACIO_SISTEMES/`    | Integració amb sistemes Windows: Samba, recursos compartits, NFS, dominis i aplicacions amb LDAP              | 14    | RA6 | Disponible    |
| `07_SCRIPTING/`              | Scripts de Bash per administrar el sistema                                                                    | 26    | RA7 | Disponible    |

## Seqüència del Curs

Les carpetes estan numerades per resultat d'aprenentatge (RA), però el curs s'imparteix en aquest ordre:

| Bloc | Continguts                                                  | Carpetes                                          | RA       |
| ---- | ----------------------------------------------------------- | ------------------------------------------------- | -------- |
| 0    | Escenari del laboratori i connexió SSH inicial              | `00_RECURSOS/`                                    | —        |
| 1    | Gestió de la informació i dels processos del sistema        | `02_GESTIO_INFORMACIO/`                           | RA2      |
| 2    | Administració remota                                        | `04_ADMINISTRACIO_REMOTA/`                        | RA4      |
| 3    | Automatització de tasques i scripting (s'imparteixen junts) | `03_AUTOMATITZACIO_TASQUES/`, `07_SCRIPTING/`     | RA3, RA7 |
| 4    | Serveis de directori (LDAP)                                 | `01_SERVEIS_DIRECTORI/`                           | RA1      |
| 5    | Integració de sistemes                                      | `06_INTEGRACIO_SISTEMES/`                         | RA6      |
| 6    | Servidors d'impressió                                       | `05_SERVIDORS_IMPRESSIO/`                         | RA5      |

**Projectes transversals.** Es fan al final del bloc corresponent, després de les pràctiques més senzilles:

- **Monitorització i alertes** (RA3 + RA7, amb serveis del mòdul 0375): en acabar el bloc 3.
- **GLPI amb autenticació LDAP** (RA1 + RA6, amb els serveis web i FTP del mòdul 0375): en acabar el bloc 5.

## Com Està Organitzat Cada Tema

- El `README.md` de cada carpeta és l'índex del tema i indica el resultat d'aprenentatge (RA) que treballa.
- Els fitxers estan numerats en l'ordre recomanat de lectura (`01_...`, `02_...`).
- Cada fitxer explica un concepte o una eina amb taules d'opcions i exemples de sortida interpretats, i acaba amb una secció de **resolució de problemes**.
- Els exemples fan servir el servidor `srv0374` de l'escenari. Les sortides de la vostra màquina poden tenir valors diferents.

## Relació amb el Mòdul 0375

El servidor 0374 és alhora la porta d'enllaç del client de la xarxa interna i el servidor secundari dels serveis DNS i DHCP del mòdul 0375. Tots dos mòduls comparteixen el client de proves i el pla d'adreces. La pràctica transversal de GLPI integra el directori LDAP del Tema 1 amb els serveis web i FTP del mòdul 0375.

## Resultats d'Aprenentatge

`RA1` Administra el servei de directori interpretant-ne especificacions i integrant-lo en una xarxa.

`RA2` Administra processos del sistema descrivint-los i aplicant criteris de seguretat i eficiència.

`RA3` Gestiona l'automatització de tasques del sistema, aplicant-hi criteris d'eficiència i utilitzant comandes i eines gràfiques.

`RA4` Administra de forma remota el sistema operatiu en xarxa valorant-ne la seva importància i aplicant-hi criteris de seguretat.

`RA5` Administra servidors d'impressió descrivint-ne les funcions i integrant-los en una xarxa.

`RA6` Integra sistemes operatius lliures i propietaris, justificant i garantint-ne la interoperabilitat.

`RA7` Utilitza llenguatges de guions en sistemes operatius, descrivint-ne la seva aplicació i administrant serveis del sistema operatiu.
