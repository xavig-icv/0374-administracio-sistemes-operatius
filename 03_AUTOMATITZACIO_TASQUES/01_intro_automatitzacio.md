## 01. Introducció a l'Automatització de Tasques

### Per què Automatitzar?

Un administrador repeteix moltes tasques: còpies de seguretat cada nit, informes de l'estat dels servidors, neteja de fitxers temporals, rotació de registres, comprovació de serveis. Fer-les a mà té tres problemes: costa temps, depèn que algú se'n recordi i és fàcil equivocar-se.

Automatitzar una tasca vol dir **escriure-la una vegada** (normalment en un script, Tema 7) i **programar-la** perquè el sistema l'executi sol en el moment adequat.

| Tasca                                      | Quan s'executa                     | Eina habitual                  |
| ------------------------------------------ | ---------------------------------- | ------------------------------ |
| Còpia de seguretat dels fitxers            | Cada nit a les 2:00                | `cron` o temporitzador de systemd |
| Informe de l'estat del servidor            | Cada dia a les 7:00                | `cron` o temporitzador         |
| Comprovar que els serveis responen         | Cada 5 minuts                      | `cron` o temporitzador         |
| Reiniciar un servei fora d'hores           | Una sola vegada, aquesta nit       | `at` o `systemd-run`           |
| Tasques diàries en un portàtil que s'apaga | Quan s'encengui, si no s'han fet   | `anacron` o `Persistent=true`  |
| Evitar que els registres omplin el disc    | Cada dia o cada setmana            | `logrotate`                    |

### Eines de Planificació

| Eina                         | Tipus de tasca                                                     | Fitxer del tema |
| ---------------------------- | ------------------------------------------------------------------ | --------------- |
| `at` / `batch`               | Una sola execució en un moment concret                             | [02](./02_at_batch.md) |
| `cron`                       | Tasques periòdiques (cada minut, hora, dia, setmana...)            | [03](./03_cron_anacron.md) |
| `anacron`                    | Tasques diàries, setmanals o mensuals en equips que no estan sempre engegats | [03](./03_cron_anacron.md) |
| Temporitzadors de systemd    | Alternativa moderna a `cron`, integrada amb els serveis i el registre | [04](./04_systemd_timers.md) |
| `logrotate`                  | Rotació i compressió dels fitxers de registre                      | [05](./05_logrotate.md) |
| Cockpit                      | Administració i planificació des d'una interfície web              | [07](./07_eines_grafiques.md) |

### Criteris d'Eficiència

Una tasca automatitzada s'executa sense ningú al davant. Cal pensar-la bé:

- **Escollir el moment:** les tasques pesades (còpies, compressions, indexacions) s'han de fer en hores de poca activitat. Si molts servidors fan la mateixa tasca, convé repartir-les en el temps perquè no coincideixin.
- **Limitar els recursos:** executar-les amb poca prioritat (`nice -n 19`, Tema 2) perquè no alenteixin els serveis.
- **Evitar solapaments:** si una execució encara no ha acabat quan comença la següent, poden interferir. `flock` ho impedeix (fitxer 03).
- **Registrar el resultat:** cada execució ha de deixar constància de si ha funcionat (`logger`, fitxers de registre) i avisar quan falla.
- **Seguretat:** executar cada tasca amb l'usuari que tingui els permisos mínims necessaris, amb rutes absolutes i amb scripts que només pugui modificar l'administrador.

### Execució Desatesa: Diferències amb la Terminal

Quan `cron` o systemd executen un script, no hi ha cap terminal ni cap usuari connectat. Això canvia algunes coses respecte de quan l'executem a mà:

| Aspecte                  | A la terminal                          | Execució programada                                   |
| ------------------------ | -------------------------------------- | ----------------------------------------------------- |
| Variable `PATH`          | Completa                               | Reduïda (`/usr/bin:/bin` a `cron`): cal escriure rutes absolutes |
| Directori de treball     | El directori actual                    | El directori personal de l'usuari o `/`               |
| Sortida per pantalla     | Es veu                                 | Es perd si no es redirigeix o no es registra          |
| Preguntes (`read`)       | L'usuari respon                        | No hi ha ningú per respondre: l'script s'encalla o falla |
| `sudo` amb contrasenya   | Es pot escriure                        | No funciona: la tasca s'ha de programar amb l'usuari adequat |

Per això els scripts que s'han de programar segueixen les bones pràctiques del Tema 7 (fitxer 11): rutes absolutes, cap pregunta a l'usuari, missatges al registre i codi de sortida correcte.

### Relació amb Altres Temes

- **Tema 2:** scripts d'informe, `logger`, serveis de systemd i registres.
- **Tema 7:** els scripts que aquí es programen.
- **Mòdul 0375:** les còpies es poden enviar a l'altre servidor de l'escenari (`192.168.1.2`) per SSH.
