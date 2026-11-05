## 07. Eines Gràfiques: Cockpit

Les tasques es poden planificar i supervisar també des d'una interfície gràfica. En un servidor sense entorn d'escriptori, la solució habitual és una **consola d'administració web**: s'hi accedeix des del navegador d'un altre equip.

**Cockpit** és la consola web que inclouen Ubuntu, Debian, Fedora i Red Hat. Permet veure l'estat del sistema, consultar els registres, gestionar serveis i temporitzadors, usuaris, emmagatzematge i actualitzacions, i obrir una terminal, tot des del navegador.

### Instal·lació i Accés

```bash
sudo apt install cockpit
systemctl status cockpit.socket          # escolta al port 9090 i engega Cockpit quan cal
sudo ss -tlnp | grep 9090
```

Des del navegador del client de l'escenari: **`https://192.168.1.1:9090`**

- El certificat és autosignat: el navegador mostrarà un avís que cal acceptar.
- S'inicia sessió amb un usuari del sistema (el vostre usuari administrador). `root` no pot iniciar sessió a Cockpit per defecte.
- Per fer canvis cal activar l'**accés administratiu** (botó de la part superior), que fa servir els permisos de `sudo` de l'usuari.

Si el servidor té el tallafoc `ufw` activat, cal obrir el port: `sudo ufw allow 9090/tcp`.

### Seccions Principals

| Secció              | Què permet fer                                                               | Equivalent en ordres            |
| ------------------- | ---------------------------------------------------------------------------- | ------------------------------- |
| Visió general       | Estat del sistema i gràfics de CPU, memòria, disc i xarxa                    | `top`, `free`, `df` (Tema 2)    |
| Registres           | Consultar el diari de systemd, filtrar per prioritat i per servei            | `journalctl`                    |
| Serveis             | Engegar, aturar i habilitar serveis; veure'n l'estat i els registres         | `systemctl`                     |
| Serveis → Temporitzadors | Llistar els temporitzadors i **crear-ne de nous**                       | `systemctl list-timers`, fitxers `.timer` |
| Comptes             | Crear usuaris i gestionar-ne els grups                                       | `useradd`, `usermod`            |
| Actualitzacions     | Instal·lar les actualitzacions pendents                                      | `apt update && apt upgrade`     |
| Terminal            | Terminal al servidor des del navegador                                       | SSH                             |

### Crear un Temporitzador des de Cockpit

1. **Serveis** (*Services*) → pestanya **Temporitzadors** (*Timers*) → **Crea un temporitzador** (*Create timer*).
2. Indicar el nom (per exemple, `neteja-temporals`), la descripció i l'ordre que s'ha d'executar (ruta absoluta de l'script).
3. Escollir quan s'executa: després d'engegar el sistema o en un moment concret (cada hora, cada dia, cada setmana...).
4. Desar. Cockpit crea el servei i el temporitzador a `/etc/systemd/system/` i l'habilita.

Es pot comprovar des de la terminal que el resultat és el mateix que si s'hagués fet a mà (fitxer 04):

```bash
systemctl list-timers neteja-temporals.timer
systemctl cat neteja-temporals.timer
```

Els noms dels botons i de les seccions depenen de l'idioma del navegador; si Cockpit no té la traducció d'algun text, el mostra en anglès.

### Altres Eines

| Eina                 | Característiques                                                              |
| -------------------- | ----------------------------------------------------------------------------- |
| **Webmin**           | Consola web amb mòduls per a gairebé tots els serveis (inclou un gestor de tasques de `cron`). No és als repositoris d'Ubuntu: cal afegir el repositori del projecte |
| **Eines d'escriptori** | En equips amb entorn gràfic, els editors de tasques programades depenen de cada escriptori |
| **Programador de tasques de Windows** | L'equivalent a `cron` als sistemes Windows (`taskschd.msc`) |

Les eines gràfiques faciliten la feina i la supervisió, però per automatitzar molts servidors alhora i entendre què passa quan una tasca falla cal dominar les ordres.

### Resolució de Problemes

| Problema                                              | Causa                                                     | Solució                                                |
| ----------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------ |
| El navegador no arriba a `https://192.168.1.1:9090`   | `cockpit.socket` aturat o tallafoc                        | `sudo systemctl enable --now cockpit.socket`; obrir el port 9090 |
| Avís de connexió no segura                            | Certificat autosignat                                     | Acceptar l'excepció al laboratori                      |
| No es poden fer canvis (botons desactivats)           | L'accés administratiu no està activat                     | Activar-lo amb la contrasenya de l'usuari              |
| No es pot iniciar sessió amb `root`                   | Cockpit ho impedeix per defecte                           | Fer servir l'usuari administrador amb `sudo`           |
