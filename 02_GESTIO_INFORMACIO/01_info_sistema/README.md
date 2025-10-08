# 01. Informació del Sistema

## Introducció

La recollida d'informació del sistema és una de les tasques més fonamentals i crítiques en l'administració de sistemes. Abans de poder optimitzar, solucionar problemes o prendre decisions per actuar sobre un servidor, necessitem saber **que hi ha al servidor**, **com està configurat** i **quin és el seu estat actual**.

Aquest bloc es centra en dominar les comandes que ens permeten obtenir informació **estàtica** del sistema, és a dir, dades que no canvien al llarg del temps (valors constants).

## Per què és tan important?

Imagina aquestes situacions que pots trobar al teu primer dia de feina:

**Escenari 1: Herència del "de sistemes"**

> T'incorpores a una empresa i et diuen: "Aquest és el servidor de producció, necessitem que l'administris com bon administrador de sistemes que ets!". No hi ha documentació, tampoc saps què hi ha instal·lat i no hi ha informació sobre les seves especificacions de hardware.

**Escenari 2: Incident de rendiment**

> Estàs teletreballant i reps una alerta: "El servidor va lent". Abans de fer res, necessites saber:

- Quin és el consum actual de la CPU.
- Quantitat de memòria RAM disposa quina s'està utilitzant.
- Si hi ha espai suficient als discos.
- Quins processos s'estan executant.

**Escenari 3: Planificació de capacitat**

> El cap de sistemes et pregunta: "Podem afegir un nou servei o programari a aquest servidor o necessitem hardware nou?"

**Resposta correcta**: Primer analitzo els recursos actuals i l'ús que en fem.

**Escenari 4: Auditoria de seguretat**

> Us demanen un informe complet de tots els servidors: hardware, sistema operatiu, serveis, usuaris connectats, etc.

Sense saber com recollir informació, no podries fer cap d'aquestes tasques de manera encertada.

## Tipus d'informació que recollirem

### 1. Hardware i Recursos Físics

- Quina CPU té el servidor (model, nuclis, arquitectura)
- Quanta memòria RAM física està instal·lada
- Quins discos hi ha i quina capacitat tenen
- Quins dispositius estan connectats (targetes de xarxa, USB, PCI)

**Per a què serveix:**

- Planificar l'escalabilitat del sistema
- Saber si podem executar aplicacions amb requisits elevats
- Detectar hardware defectuós o obsolet
- Documentar l'inventari de servidors

**Exemple:**

```bash
# Descobrim que el servidor disposa de 4GB de RAM
# i volem instal·lar una base de dades que necessita 4GB mínim
# Decisió: Si compartirà recursos amb altres serveis cal ampliar la memòria abans d'instal·lar res
```

### 2. Sistema Operatiu i Kernel

- Quina distribució de Linux s'està executant
- Quina és la versió del kernel
- Quan es va instal·lar el sistema
- La seva arquitectura (32 o 64 bits)

**Per a què serveix:**

- Saber si el sistema està actualitzat
- Comprovar la compatibilitat amb aplicacions que volem instal·lar
- Identificar possibles vulnerabilitats de seguretat
- Planificar actualitzacions properes

**Exemple:**

```bash
# Descobrim que s'està executant una versió antiga del kernel o la distribució de Linux
# Decisió: Cal actualitzar per disposar de les últimes correccions de seguretat
```

### 3. Estat de la Memòria i Swap

- Quantitat de memòria RAM s'està utilitzant
- Quantitat de memòria lliure o a caché
- Si s'està emprant la swap (memòria virtual al disc)
- Com està distribuïda la memòria entre processos

**Per a què serveix:**

- Detectar problemes de rendiment per falta de memòria
- Decidir si cal afegir més RAM o balancejar la càrrega
- Identificar memory leaks (fuites de memòria)
- Optimitzar la configuració de swap

**Exemple:**

```bash
# Veiem que el servidor utilitza el 95% de la RAM i el 80% de la SWAP.
# Problema: El sistema necessita memòria i va lent perquè utilitza molta memòria swap.
# Solució immediata: Identificar processos que consumeixen una memòria excesiva.
# Solució a llarg termini: Ampliar la RAM o balancejar la càrrega entre servidors.
```

### 4. Discos i Sistemes de Fitxers

- Quins discos estan muntats
- Quantitat d'espai que queda lliure al disc
- Quins tipus de sistemes de fitxers s'utilitzen (ext4, xfs, etc.)
- Quines particions hi ha creades

**Per a què serveix:**

- Evitar que els discos s'omplin (i provoquin crashes)
- Planificar ampliacions d'emmagatzematge
- Identificar directoris que ocupen massa espai
- Gestionar backups de manera eficient

**Exemple:**

```bash
# La partició arrel (/) està ocupada al 98%
# Problema crític: En pocs dies el servidor pot deixar de funcionar
# Acció: Identificar què ocupa espai (logs? backups al mateix server? fitxers temporals?)
```

### 5. Gestió de la Xarxa

- Quines interfícies de xarxa hi ha al servidor
- Quines IPs estan assignades i quines són les seves MAC
- Quines rutes de xarxa estan configurades
- Quines connexions estan actives

**Per a què serveix:**

- Diagnosticar problemes de connectivitat
- Verificar la configuració de xarxa
- Detectar connexions sospitoses
- Configurar noves interfícies o rutes

**Exemple:**

```bash
# El nou servidor no pot accedir a Internet
# Diagnòsi: Comprovem les interfícies i rutes
# Descobriment: Disposem de gateway però la ruta per sortir a Internet no està configurada
```

### 6. Usuaris i Sessions

- Qui està connectat al servidor
- Des d'on s'ha connectat (IP origen)
- Quan es va connectar
- Quines accions està realitzant

**Per a què serveix:**

- Auditoria de seguretat
- Detectar accessos no autoritzats
- Gestionar sessions d'usuaris
- Troubleshooting de problemes causats per usuaris

**Exemple:**

```bash
# Veiem un usuari connectat des d'una IP d'un país extranger
# Possible problema de seguretat o fa ús d'una VPN
# Acció: Investigar si és legítim o un intent d'intrusió
```

### 7. Kernel i Paràmetres del Sistema

- Missatges del kernel sobre hardware i drivers
- Paràmetres de configuració interna del sistema
- Errors de maquinari o software

**Per a què serveix:**

- Diagnosticar problemes de hardware
- Optimitzar el rendiment del sistema
- Solucionar errors de drivers
- Configurar paràmetres avançats

**Exemple:**

```bash
# dmesg mostra errors de lectura del disc
# Problema: El disc dur pot estar fallant
# Acció urgent: Fer backup i planificar una substitució del disc
```

## Flux de treball: Recollida d'Informació Professional

Quan et trobes amb un servidor nou o desconegut, segueix aquest ordre:

```
1. Identificació bàsica
   └─> uname -a, hostnamectl
       "Què és aquest sistema?"

2. Recursos de hardware
   └─> lscpu, free -h, lsblk, df -h
       "Quins recursos té disponibles?"

3. Estat actual dels recursos
   └─> uptime, free -h, df -h
       "Com està de carregat?"

4. Configuració de xarxa
   └─> ip a, ip route, ss -tuln
       "Com està connectat?"

5. Què s'està executant
   └─> ps aux, systemctl list-units
       "Quins serveis té actius?"

6. Qui està connectat
   └─> who, w, last
       "Hi ha algú més treballant aquí?"

7. Problemes recents
   └─> dmesg | tail, journalctl -xe
       "Hi ha hagut errors?"
```

---

## Cas pràctic: Primer dia de feina

**Situació:**
Has entrat a treballar com a administrador de sistemes júnior. El teu cap et diu:

> "Necessito un informe complet del servidor `srv-prod-01`. Documenta-ho tot: hardware, sistema operatiu, recursos, xarxa, serveis actius. Ho vull en un document ben estructurat per demà."

**Tasca:**
Utilitzar les comandes d'aquest bloc per recollir tota la informació necessària i generar un informe professional.

**Resultat esperat:**
Un document que contingui:

- Especificacions tècniques completes
- Estat actual dels recursos
- Configuració de xarxa
- Serveis en execució
- Recomanacions d'optimització o millora

Aquest és el tipus de tasca real que realitzaràs com a administrador de sistemes.

## Objectius d'aprenentatge d'aquest tema

Al finalitzar aquest tema, seràs capaç de:

1. Identificar les característiques de hardware d'un servidor Linux
2. Extreure informació completa sobre CPU, memòria, discos i xarxa
3. Consultar l'estat del sistema operatiu i el kernel
4. Llistar processos actius i la seva jeràrquia
5. Identificar usuaris connectats i el seu històric
6. Navegar i cercar informació en l'estructura de directoris
7. Analitzar missatges del kernel per detectar problemes
8. Generar informes professionals de l'estat del sistema
9. Combinar comandes per filtrar i processar informació
10. Documentar l'estat complet d'un servidor de producció
