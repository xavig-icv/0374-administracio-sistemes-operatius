## 08. Administrar Diversos Servidors Alhora

Quan un administrador té desenes de servidors, no pot entrar a cadascun per fer la mateixa tasca. Les dues opcions bàsiques són els **scripts** que es connecten per SSH i les eines d'**automatització de la configuració**, com Ansible.

### Scripts amb SSH

Amb claus SSH i el fitxer `~/.ssh/config` (fitxers 02 i 03), un bucle de Bash (Tema 7) pot executar una ordre a molts servidors:

```bash
#!/bin/bash
# estat-servidors.sh - Mostra la càrrega i el disc de tots els servidors
servidors=("srv0374" "srv0375")

for s in "${servidors[@]}"; do
    echo "=== $s ==="
    ssh -o ConnectTimeout=5 "$s" "uptime; df -h / | tail -n 1" || echo "No respon"
done
```

Per executar-ho en paral·lel, cada `ssh` es llança en segon pla i s'espera amb `wait` (Tema 7, fitxer 11).

### Ansible

**Ansible** aplica configuracions a molts equips des d'un equip central (el *node de control*). No cal instal·lar res als servidors administrats: es connecta per **SSH** i només necessita Python, que Ubuntu ja porta.

| Concepte       | Què és                                                                        |
| -------------- | ----------------------------------------------------------------------------- |
| Inventari      | Fitxer amb la llista d'equips i grups d'equips                                |
| Mòdul          | Acció que Ansible sap fer: instal·lar paquets (`apt`), copiar fitxers (`copy`), gestionar serveis (`service`), usuaris (`user`)... |
| Ordre ad hoc   | Una tasca puntual des de la línia d'ordres                                    |
| *Playbook*     | Fitxer YAML amb la llista de tasques que s'han d'aplicar                      |
| Idempotència   | Executar un *playbook* dues vegades deixa el sistema igual: només canvia el que no estigui com es demana |

**Instal·lació al node de control** (per exemple, el servidor 0374, que arriba a tots els equips de la xarxa interna):

```bash
sudo apt install ansible
```

**Inventari** (`inventari.ini`):

```ini
[servidors]
srv0375 ansible_host=192.168.1.2

[clients]
client1 ansible_host=192.168.1.50 ansible_user=alumne

[all:vars]
ansible_user=xavi
```

**Ordres ad hoc:**

```bash
ansible all -i inventari.ini -m ping                          # comprova la connexió amb tots els equips
ansible all -i inventari.ini -a "uptime"                      # executa una ordre
ansible all -i inventari.ini -m apt -a "name=htop state=present" --become -K   # instal·la un paquet amb sudo
```

`--become` executa la tasca amb `sudo` i `-K` demana la contrasenya de `sudo`.

**Un *playbook*** (`base.yml`): la configuració bàsica que han de tenir tots els equips.

```yaml
- name: Configuració bàsica dels equips del laboratori
  hosts: all
  become: true
  tasks:
    - name: Paquets d'administració instal·lats
      ansible.builtin.apt:
        name: [htop, curl, rsync]
        state: present
        update_cache: true

    - name: Usuari de suport creat
      ansible.builtin.user:
        name: suport
        groups: sudo
        append: true
        shell: /bin/bash

    - name: Missatge d'avís a l'inici de sessió
      ansible.builtin.copy:
        content: "Accés restringit al personal autoritzat d'Educem SL\n"
        dest: /etc/issue.net
```

```bash
ansible-playbook -i inventari.ini base.yml -K
```

```
PLAY RECAP *********************************************************************
client1   : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
srv0375   : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

`ok` compta les tasques correctes (inclosa la recollida inicial d'informació de l'equip) i `changed`, les que han hagut de modificar alguna cosa. Si s'executa una segona vegada, `changed=0`: els equips ja estan com indica el *playbook*.

### Resolució de Problemes

| Problema                                                    | Causa                                                    | Solució                                                     |
| ----------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| `UNREACHABLE! ... Permission denied (publickey)`            | La clau del node de control no està autoritzada a l'equip | Copiar la clau pública amb `ssh-copy-id` i provar `ssh` a mà |
| `Host key verification failed`                              | El node de control no s'hi ha connectat mai               | Connectar-s'hi una vegada amb `ssh` per acceptar la clau    |
| `Missing sudo password`                                     | La tasca necessita `sudo` i no s'ha indicat la contrasenya | Afegir `-K`                                                |
| Error de sintaxi al *playbook*                              | Sagnat incorrecte del YAML (com a netplan, Tema 2)       | `ansible-playbook --syntax-check base.yml`                  |
