## 05. Control d'Accés i Quotes d'Impressió

### Qui Pot Imprimir a Cada Impressora

Per a cada impressora es pot indicar quins usuaris o grups hi poden imprimir (o quins no):

```bash
sudo lpadmin -p PDF -u allow:@professors,anna     # només el grup professors i l'usuari anna
sudo lpadmin -p PDF -u deny:marc                  # tothom menys marc
sudo lpadmin -p PDF -u allow:all                  # tothom (valor per defecte)
lpstat -l -p PDF | grep -i -A2 allowed            # comprovar-ho
```

Per als clients de la xarxa, l'usuari que veu CUPS és el que s'ha autenticat davant del servidor; si no hi ha autenticació, els treballs remots arriben com a anònims. En una empresa, la impressió s'integra amb els usuaris del directori (Tema 1) o del domini (Tema 6).

### Des de quines Xarxes s'Accepten Connexions

El fitxer `/etc/cups/cupsd.conf` controla l'accés al servidor i a la interfície web amb blocs `<Location>`:

```
# Accés a les impressores: només des de les xarxes del laboratori
<Location />
  Order allow,deny
  Allow from 127.0.0.1
  Allow from 192.168.1.0/24
  Allow from 10.0.1.0/24
</Location>

# Administració: només des de la xarxa interna i amb un usuari del grup lpadmin
<Location /admin>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
  Allow from 127.0.0.1
  Allow from 192.168.1.0/24
</Location>
```

```bash
sudo cupsd -t                     # comprovar la sintaxi de cupsd.conf
sudo systemctl restart cups
```

`@SYSTEM` són els grups d'administració de CUPS (`lpadmin` a Ubuntu).

### Quotes

CUPS pot limitar quantes pàgines o quants kilobytes pot imprimir cada usuari en un període:

```bash
# Màxim 50 pàgines per usuari cada setmana (604800 segons) a la impressora PDF
sudo lpadmin -p PDF -o job-quota-period=604800 -o job-page-limit=50

# Màxim 10 MB per usuari i dia
sudo lpadmin -p PDF -o job-quota-period=86400 -o job-k-limit=10240
```

| Opció                 | Significat                                           |
| --------------------- | ---------------------------------------------------- |
| `job-quota-period`    | Període de la quota en segons                        |
| `job-page-limit`      | Pàgines màximes per usuari en el període             |
| `job-k-limit`         | Kilobytes màxims per usuari en el període            |

Quan un usuari supera la quota, els seus treballs nous es rebutgen fins que s'acaba el període. Les pàgines impreses per cada usuari queden registrades al fitxer `page_log` (fitxer 06), que serveix per fer estadístiques i repercutir els costos per departament.

### Resolució de Problemes

| Problema                                                      | Causa                                                      | Solució                                                     |
| ------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------- |
| `Forbidden` o `client-error-not-authorized` en imprimir       | L'usuari no és a la llista `allow` de la impressora        | `lpstat -l -p` per veure-la; corregir-la amb `lpadmin -u`   |
| Un client remot no pot imprimir després d'editar `cupsd.conf` | La seva xarxa no és als blocs `Allow from`                 | Afegir-la i reiniciar CUPS                                  |
| CUPS no arrenca després d'editar `cupsd.conf`                 | Error de sintaxi                                           | `sudo cupsd -t` indica la línia                             |
| Les quotes no s'apliquen als treballs remots                  | Els treballs arriben com a anònims                         | Fer que els clients s'autentiquin                           |
