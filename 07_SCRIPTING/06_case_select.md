## 06. Selecció Múltiple i Menús: `case` i `select`

### `case`

Quan una variable pot tenir molts valors diferents, `case` és més clar que una cadena d'`elif`:

```bash
case "$variable" in
    valor1)
        ordres
        ;;
    valor2|valor3)          # diversos valors separats per |
        ordres
        ;;
    patró*)                 # patrons: *, ?, [abc]
        ordres
        ;;
    *)                      # qualsevol altre valor (opció per defecte)
        ordres
        ;;
esac
```

Cada bloc acaba amb `;;`. Només s'executa el **primer** bloc que coincideix.

```bash
#!/bin/bash
# servei.sh - Ús: ./servei.sh {start|stop|status} nom_servei
accio=$1
servei=$2

case "$accio" in
    start|engegar)
        sudo systemctl start "$servei"
        ;;
    stop|aturar)
        sudo systemctl stop "$servei"
        ;;
    status|estat)
        systemctl status "$servei" --no-pager
        ;;
    *)
        echo "Ús: $0 {start|stop|status} nom_servei" >&2
        exit 1
        ;;
esac
```

**Patrons per acceptar diverses respostes:**

```bash
read -rp "Vols continuar? (s/n) " resposta
case "$resposta" in
    [sS]|[sS][iI]) echo "Continuem" ;;
    [nN]|[nN][oO]) echo "Cancel·lat"; exit 0 ;;
    *)             echo "Resposta no vàlida" ;;
esac
```

### `select`: Menús Automàtics

`select` mostra una llista numerada, llegeix el número que escriu l'usuari i desa l'element triat a la variable. Es repeteix fins que s'executa `break`:

```bash
#!/bin/bash
# menu.sh - Menú d'informació del servidor
PS3="Tria una opció: "                  # missatge que mostra select
opcions=("Sistema" "Memòria" "Disc" "Usuaris connectats" "Sortir")

select opcio in "${opcions[@]}"; do
    case "$opcio" in
        "Sistema")            hostnamectl ;;
        "Memòria")            free -h ;;
        "Disc")               df -h / ;;
        "Usuaris connectats") who ;;
        "Sortir")             break ;;
        *)                    echo "Opció no vàlida: $REPLY" ;;
    esac
done
```

```
1) Sistema
2) Memòria
3) Disc
4) Usuaris connectats
5) Sortir
Tria una opció: 3
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  5.1G  5.8G  47% /
Tria una opció: 5
```

`$REPLY` conté el que ha escrit l'usuari i `$opcio`, l'element triat (queda buit si el número no és vàlid). Després de cada opció, `select` només torna a mostrar el missatge; la llista sencera torna a aparèixer si es prem Enter sense escriure res.

### Menú Principal amb `while` i `case`

Per controlar millor el format (colors, capçalera, netejar la pantalla), el menú es pot construir a mà amb un bucle (fitxer 07):

```bash
#!/bin/bash
# menu2.sh
while true; do
    clear
    echo "===== MENÚ srv0374 ====="
    echo "1) Informació del sistema"
    echo "2) Espai de disc"
    echo "3) Sortir"
    read -rp "Opció: " opcio
    case "$opcio" in
        1) hostnamectl ;;
        2) df -h ;;
        3) echo "Adéu!"; break ;;
        *) echo "Opció no vàlida" ;;
    esac
    read -rp "Prem Enter per continuar..."
done
```

### Resolució de Problemes

| Problema                                       | Causa                                                    | Solució                                                     |
| ---------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| `syntax error near unexpected token ')'`       | Falta `;;` al bloc anterior o `in` després de la variable | Revisar l'estructura del `case`                            |
| Sempre s'executa l'opció `*)`                  | El valor no coincideix exactament (majúscules, espais)   | Mostrar la variable entre claudàtors: `echo "[$var]"`       |
| `select` no torna a mostrar la llista          | Comportament normal de `select`                          | Prémer Enter sense escriure res                             |
| El menú no acaba mai                           | Falta `break` a l'opció de sortir                        | Afegir `break`                                              |
