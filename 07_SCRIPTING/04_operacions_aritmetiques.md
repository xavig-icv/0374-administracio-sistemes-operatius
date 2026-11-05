## 04. Operacions Aritmètiques

Bash fa operacions amb **nombres enters**. Hi ha tres maneres d'escriure-les:

| Forma               | Ús                                                         | Exemple                |
| ------------------- | ---------------------------------------------------------- | ---------------------- |
| `$(( expressió ))`  | Calcula i retorna el resultat                              | `total=$((a + b))`     |
| `(( expressió ))`   | Calcula sense retornar res (increments, comparacions)      | `((comptador++))`      |
| `let`               | Forma antiga, equivalent a `(( ))`                         | `let total=a+b`        |

Dins de `$(( ))` i de `(( ))` les variables es poden escriure sense `$` i es poden posar espais.

| Operador | Significat       | Exemple (`a=17`, `b=5`) | Resultat |
| -------- | ---------------- | ----------------------- | -------- |
| `+`      | Suma             | `$((a + b))`            | 22       |
| `-`      | Resta            | `$((a - b))`            | 12       |
| `*`      | Multiplicació    | `$((a * b))`            | 85       |
| `/`      | Divisió entera   | `$((a / b))`            | 3        |
| `%`      | Residu (mòdul)   | `$((a % b))`            | 2        |
| `**`     | Potència         | `$((b ** 2))`           | 25       |

| Operador d'assignació                          | Equivalent              |
| ---------------------------------------------- | ----------------------- |
| `((x++))` / `((x--))`                          | `x=x+1` / `x=x-1`       |
| `((x += 5))`                                   | `x=x+5`                 |
| `((x -= 5))`, `((x *= 2))`, `((x /= 2))`, `((x %= 3))` | Resta, multiplica, divideix o calcula el residu i assigna el resultat |

```bash
#!/bin/bash
# aritmetica.sh
a=17
b=5
echo "Suma: $((a + b))"
echo "Divisió entera: $((a / b)) i residu: $((a % b))"

comptador=0
((comptador += 10))
((comptador++))
echo "Comptador: $comptador"
```

```
Suma: 22
Divisió entera: 3 i residu: 2
Comptador: 11
```

### Compte amb la Divisió Entera

Aquest càlcul del percentatge de memòria lliure sempre dona 0:

```bash
ram_total=8192
ram_lliure=2048
percentatge=$(( (ram_lliure / ram_total) * 100 ))    # 2048 / 8192 = 0 (enter) → 0 × 100 = 0
```

Cal **multiplicar primer i dividir després**:

```bash
percentatge=$(( ram_lliure * 100 / ram_total ))      # 204800 / 8192 = 25
```

### Decimals: `bc` i `awk`

Per fer operacions amb decimals cal una eina externa:

```bash
echo "scale=2; 2048 / 8192 * 100" | bc                # 25.00 (scale: nombre de decimals)
awk 'BEGIN {printf "%.2f\n", 2048 / 8192 * 100}'      # 25.00
```

**Exemple real:** percentatge del disc arrel ocupat, a partir de la sortida de `df` (Tema 2):

```bash
usat=$(df --output=used / | tail -n 1)
total=$(df --output=size / | tail -n 1)
echo "Disc arrel ocupat: $(( usat * 100 / total ))%"
```

### Comparar Números

Dins de `(( ))` es poden fer comparacions amb els símbols habituals (`<`, `>`, `<=`, `>=`, `==`, `!=`). Es fan servir a les condicions (fitxer 05):

```bash
if (( percentatge > 80 )); then
    echo "Disc gairebé ple"
fi
```

### Resolució de Problemes

| Problema                                                       | Causa                                               | Solució                                    |
| -------------------------------------------------------------- | --------------------------------------------------- | ------------------------------------------ |
| Un percentatge sempre dona 0                                   | Divisió entera abans de multiplicar                 | Multiplicar primer: `a * 100 / b`          |
| `syntax error: invalid arithmetic operator (error token is ".5")` | Decimals dins de `$(( ))`                        | Fer servir `bc` o `awk`                    |
| `resultat: command not found`                                  | Espais al voltant del `=`                           | `resultat=$((a + b))`                      |
| `value too great for base (error token is "08")`               | Bash llegeix en octal els números que comencen per 0 | `$((10#$numero))`                         |
| `bc: command not found`                                        | El paquet no està instal·lat                        | `sudo apt install bc`                      |
