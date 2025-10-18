# 01. Sistema i Hardware

Abans de gestionar un servidor, necessitem conèixer les seves característiques de hardware i la configuració del sistema operatiu. En aquest bloc treballarem les comandes essencials per obtenir informació sobre el processador, la memòria, els dispositius connectats i la configuració bàsica del sistema.

**Objectiu**: Aprendre a identificar les especificacions tècniques d'un servidor Linux i interpretar la informació del hardware.

## Comandes d'aquest tema

1. `uname` - Informació del kernel i sistema operatiu
2. `hostnamectl` - Informació completa del sistema
3. `lscpu` - Detalls de la CPU
4. `lspci` - Dispositius PCI
5. `lsusb` - Dispositius USB
6. `lsblk` - Dispositius de bloc (discos)
7. `dmidecode` - Informació DMI/SMBIOS del hardware
8. `hwinfo` - Resum complet del hardware

## 1. `uname` - Informació del kernel

### Descripció

La comanda `uname` (unix name) mostra informació bàsica sobre el sistema operatiu i el kernel de Linux. És una de les primeres comandes que executaràs per identificar ràpidament un sistema.

### Sintaxi

```bash
uname [OPCIONS]
```

### Opcions principals

| Opció | Descripció                                     |
| ----- | ---------------------------------------------- |
| `-a`  | Mostra tota la informació disponible           |
| `-s`  | Nom del kernel (per defecte)                   |
| `-n`  | Nom de xarxa del sistema (hostname)            |
| `-r`  | Versió del kernel (release)                    |
| `-v`  | Versió completa del kernel                     |
| `-m`  | Arquitectura del hardware (x86_64, i686, etc.) |
| `-p`  | Tipus de processador                           |
| `-o`  | Sistema operatiu                               |

### Exemples pràctics

**Exemple 1: Informació completa del sistema**

```bash
uname -a
```

**Sortida:**

```
Linux srv-prod-01 6.8.0-45-generic #45-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 18 12:03:06 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

**Exemple 2: Només la versió del kernel**

```bash
uname -r
```

**Sortida:**

```
6.8.0-45-generic
```

**Exemple 3: Arquitectura del sistema**

```bash
uname -m
```

**Sortida:**

```
x86_64
```

### Interpretació de la sortida

Quan executem `uname -a`, la sortida conté (en ordre):

1. **Nom del kernel**: `Linux`
2. **Hostname**: `srv-prod-01`
3. **Versió del kernel**: `6.8.0-45-generic`
4. **Info de compilació**: Data i hora de compilació del kernel
5. **Arquitectura**: `x86_64` (sistema de 64 bits)
6. **Sistema operatiu**: `GNU/Linux`

## 2. `hostnamectl` - Informació completa del sistema

### Descripció

`hostnamectl` és part de systemd i proporciona informació detallada sobre el sistema operatiu, hostname, arquitectura i virtualització. És més completa que `uname` per identificar la distribució.

### Sintaxi

```bash
hostnamectl [OPCIONS] [COMANDA]
```

### Opcions i comandes principals

| Comanda            | Descripció                            |
| ------------------ | ------------------------------------- |
| (sense opcions)    | Mostra tota la informació del sistema |
| `status`           | Igual que sense opcions               |
| `set-hostname NOM` | Canvia el hostname (requereix sudo)   |
| `--static`         | Mostra només el hostname estàtic      |
| `--transient`      | Mostra el hostname transitori         |

### Exemples pràctics

**Exemple 1: Informació completa del sistema**

```bash
hostnamectl
```

**Sortida:**

```
 Static hostname: srv-prod-01
       Icon name: computer-vm
         Chassis: vm
      Machine ID: 1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p
         Boot ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
  Virtualization: kvm
Operating System: Ubuntu 24.04 LTS
          Kernel: Linux 6.8.0-45-generic
    Architecture: x86-64
 Hardware Vendor: QEMU
  Hardware Model: Standard PC (Q35 + ICH9, 2009)
```

**Exemple 2: Només el hostname**

```bash
hostnamectl --static
```

**Sortida:**

```
srv-prod-01
```

**Exemple 3: Canviar el hostname (requereix sudo)**

```bash
sudo hostnamectl set-hostname nou-servidor
```

### Interpretació de la sortida

Informació clau que ens proporciona:

- **Static hostname**: Nom permanent del servidor
- **Chassis**: Tipus de sistema (vm = màquina virtual, server, laptop, etc.)
- **Virtualization**: Si és una VM, quin hipervisor (kvm, vmware, virtualbox)
- **Operating System**: Distribució i versió exacta
- **Kernel**: Versió del kernel
- **Architecture**: x86-64 (64 bits)
- **Hardware Vendor/Model**: Fabricant i model del hardware (o hipervisor)

## 3. `lscpu` - Detalls de la CPU

### Descripció

`lscpu` mostra informació detallada sobre l'arquitectura de la CPU: nombre de nuclis, threads, velocitat, caché, etc. És essencial per planificar la càrrega de treball.

### Sintaxi

```bash
lscpu [OPCIONS]
```

### Opcions principals

| Opció           | Descripció                          |
| --------------- | ----------------------------------- |
| (sense opcions) | Mostra tota la informació de la CPU |
| `-e`            | Format estès amb tots els detalls   |
| `-p`            | Format parseable (útil per scripts) |
| `--json`        | Sortida en format JSON              |

### Exemples pràctics

**Exemple 1: Informació completa de la CPU**

```bash
lscpu
```

**Sortida (simplificada):**

```
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         40 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  4
  On-line CPU(s) list:   0-3
Vendor ID:               GenuineIntel
  Model name:            Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz
    CPU family:          6
    Model:               79
    Thread(s) per core:  1
    Core(s) per socket:  4
    Socket(s):           1
    Stepping:            1
    BogoMIPS:            4788.92
    Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr
Virtualization features:
  Virtualization:        VT-x
Caches (sum of all):
  L1d:                   128 KiB (4 instances)
  L1i:                   128 KiB (4 instances)
  L2:                    1 MiB (4 instances)
  L3:                    35 MiB (1 instance)
```

**Exemple 2: Format JSON (útil per scripts o importar amb JavaScript)**

```bash
lscpu --json
```

**Exemple 3: Només nombre de CPUs**

```bash
lscpu | grep "^CPU(s):"
```

**Sortida:**

```
CPU(s):                  4
```

### Interpretació de la sortida

Camps més importants:

- **CPU(s)**: Total de CPUs lògiques disponibles (nuclis × threads)
- **Thread(s) per core**: Threads per nucli (2 si té Hyper-Threading)
- **Core(s) per socket**: Nuclis físics per processador
- **Socket(s)**: Nombre de processadors físics
- **Model name**: Model exacte del processador
- **CPU MHz**: Velocitat actual (pot variar amb CPU scaling)
- **BogoMIPS**: Mesura de velocitat del processador
- **Virtualization**: Si suporta virtualització (VT-x per Intel, AMD-V per AMD)
- **L1d/L1i/L2/L3**: Mides de les memòries caché

**Càlcul de CPUs:**

```
Total CPUs = Sockets × Cores per socket × Threads per core
Exemple: 1 × 4 × 1 = 4 CPUs
```

## 4. `lspci` - Dispositius PCI

### Descripció

`lspci` llista tots els dispositius connectats al bus PCI (Peripheral Component Interconnect): targetes de xarxa, gràfiques, controladors de disc, USB, etc.

### Sintaxi

```bash
lspci [OPCIONS]
```

### Opcions principals

| Opció           | Descripció                          |
| --------------- | ----------------------------------- |
| (sense opcions) | Llista bàsica de dispositius        |
| `-v`            | Informació detallada (verbose)      |
| `-vv`           | Molt detallat                       |
| `-k`            | Mostra els drivers del kernel en ús |
| `-nn`           | Mostra IDs de vendor i dispositiu   |
| `-t`            | Vista en arbre                      |

### Exemples pràctics

**Exemple 1: Llista bàsica de dispositius**

```bash
lspci
```

**Sortida (parcial):**

```
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma]
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:01.1 IDE interface: Intel Corporation 82371SB PIIX3 IDE [Natoma/Triton II]
00:01.3 Bridge: Intel Corporation 82371AB/EB/MB PIIX4 ACPI
00:02.0 VGA compatible controller: Red Hat, Inc. QXL paravirtual graphic card
00:03.0 Ethernet controller: Red Hat, Inc. Virtio network device
00:04.0 SCSI storage controller: Red Hat, Inc. Virtio block device
```

**Exemple 2: Informació detallada de les targetes de xarxa**

```bash
lspci -v | grep -A 10 "Ethernet"
```

**Exemple 3: Veure quin driver utilitza cada dispositiu**

```bash
lspci -k
```

**Sortida (parcial):**

```
00:03.0 Ethernet controller: Red Hat, Inc. Virtio network device
        Subsystem: Red Hat, Inc. Virtio network device
        Kernel driver in use: virtio-pci
        Kernel modules: virtio_pci
```

### Interpretació de la sortida

Format de cada línia: `[bus]:[dispositiu].[funció] [tipus]: [fabricant] [model]`

- **00:03.0**: Adreça PCI (bus 00, dispositiu 03, funció 0)
- **Ethernet controller**: Tipus de dispositiu
- **Red Hat, Inc.**: Fabricant (vendor)
- **Virtio network device**: Model específic
- **Kernel driver in use**: Driver que el controla

Tipus de dispositius comuns:

- **Ethernet controller**: Targeta de xarxa
- **VGA compatible controller**: Targeta gràfica
- **SCSI storage controller**: Controlador de disc
- **USB controller**: Controlador USB
- **Audio device**: Targeta de so

## 5. `lsusb` - Dispositius USB

### Descripció

`lsusb` mostra tots els dispositius USB connectats al sistema: pendrives, teclats, ratolins, càmeres, etc.

### Sintaxi

```bash
lsusb [OPCIONS]
```

### Opcions principals

| Opció                 | Descripció                           |
| --------------------- | ------------------------------------ |
| (sense opcions)       | Llista bàsica de dispositius USB     |
| `-v`                  | Informació detallada                 |
| `-t`                  | Vista jeràrquica en arbre            |
| `-s [[bus]:][devnum]` | Mostra només un dispositiu específic |

### Exemples pràctics

**Exemple 1: Llista de dispositius USB**

```bash
lsusb
```

**Sortida:**

```
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 001 Device 002: ID 0627:0001 Adomax Technology Co., Ltd QEMU USB Tablet
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

**Exemple 2: Vista jeràrquica**

```bash
lsusb -t
```

**Sortida:**

```
/:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=uhci_hcd/2p, 12M
    |__ Port 1: Dev 2, If 0, Class=Human Interface Device, Driver=usbhid, 12M
/:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=ehci-pci/6p, 480M
```

**Exemple 3: Informació detallada d'un dispositiu**

```bash
lsusb -v -s 001:002
```

### Interpretació de la sortida

Format: `Bus [num] Device [num]: ID [vendor]:[product] [descripció]`

- **Bus 001**: Número del bus USB
- **Device 002**: Número del dispositiu al bus
- **ID 0627:0001**: ID del fabricant i producte
- **QEMU USB Tablet**: Descripció del dispositiu

En un servidor físic, és habitual veure:

- Teclats i ratolins USB
- Pendrives de backup
- Dispositius de xarxa USB
- Lectors de targetes

En VMs, sovint només veurem els USB virtuals.

## 6. `lsblk` - Dispositius de bloc (discos)

### Descripció

`lsblk` (list block devices) mostra informació sobre tots els dispositius d'emmagatzematge de bloc: discos durs, SSDs, particions, volums LVM, etc. És essencial per entendre l'estructura d'emmagatzematge del sistema.

### Sintaxi

```bash
lsblk [OPCIONS] [DISPOSITIU...]
```

### Opcions principals

| Opció           | Descripció                               |
| --------------- | ---------------------------------------- |
| (sense opcions) | Vista en arbre de tots els dispositius   |
| `-a`            | Inclou dispositius buits                 |
| `-f`            | Mostra informació del sistema de fitxers |
| `-o COLUMNES`   | Especifica quines columnes mostrar       |
| `-p`            | Mostra el path complet dels dispositius  |

### Exemples pràctics

**Exemple 1: Vista bàsica dels dispositius**

```bash
lsblk
```

**Sortida:**

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   50G  0 disk
├─sda1   8:1    0    1M  0 part
├─sda2   8:2    0    2G  0 part /boot
└─sda3   8:3    0   48G  0 part /
sdb      8:16   0  100G  0 disk
└─sdb1   8:17   0  100G  0 part /data
sr0     11:0    1 1024M  0 rom
```

**Exemple 2: Amb informació del sistema de fitxers**

```bash
lsblk -f
```

**Sortida:**

```
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1
├─sda2 ext4   1.0         a1b2c3d4-e5f6-7890-abcd-ef1234567890    1.5G    25% /boot
└─sda3 ext4   1.0         f1e2d3c4-b5a6-9876-fedc-ba9876543210   38.2G    20% /
sdb
└─sdb1 ext4   1.0   DATA  1a2b3c4d-5e6f-7g8h-9i0j-k1l2m3n4o5p6   85.3G    10% /data
```

**Exemple 3: Columnes personalitzades**

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
```

### Interpretació de la sortida

Columnes principals:

- **NAME**: Nom del dispositiu (sda = primer disc SATA/SCSI, sdb = segon, etc.)
- **MAJ:MIN**: Major i minor number (identificadors del kernel)
- **RM**: Removable (1 si és extraïble, 0 si no)
- **SIZE**: Mida del dispositiu o partició
- **RO**: Read-Only (1 si és només lectura)
- **TYPE**: Tipus (disk, part=partició, rom, lvm, etc.)
- **MOUNTPOINTS**: On està muntat el sistema de fitxers
- **FSTYPE**: Tipus de sistema de fitxers (ext4, xfs, swap, etc.)
- **FSAVAIL**: Espai disponible
- **FSUSE%**: Percentatge d'ús

Estructura jeràrquica:

```
sda (disc sencer)
├─ sda1 (partició 1)
├─ sda2 (partició 2)
└─ sda3 (partició 3)
```

## 7. `dmidecode` - Informació DMI/SMBIOS

### Descripció

`dmidecode` llegeix la taula DMI (Desktop Management Interface) o SMBIOS del sistema, que conté informació detallada del hardware proporcionada pel BIOS/UEFI. Proporciona dades tècniques molt específiques.

**Important:** Requereix permisos de root (`sudo`).

### Sintaxi

```bash
sudo dmidecode [OPCIONS]
```

### Opcions principals

| Opció           | Descripció                      |
| --------------- | ------------------------------- |
| (sense opcions) | Mostra tota la informació DMI   |
| `-t TYPE`       | Mostra només un tipus específic |
| `-s KEYWORD`    | Mostra només un camp específic  |
| `-q`            | Mode silenciós (menys verbose)  |

### Tipus comuns (-t)

| Tipus | Descripció              |
| ----- | ----------------------- |
| `0`   | BIOS                    |
| `1`   | Sistema                 |
| `2`   | Placa base (baseboard)  |
| `3`   | Xassís                  |
| `4`   | Processador             |
| `16`  | Array de memòria física |
| `17`  | Dispositiu de memòria   |

### Exemples pràctics

**Exemple 1: Informació del processador**

```bash
sudo dmidecode -t processor
```

**Sortida (simplificada):**

```
Processor Information
        Socket Designation: CPU 0
        Type: Central Processor
        Family: Xeon
        Manufacturer: Intel
        ID: F2 06 04 00 FF FB EB BF
        Version: Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz
        Voltage: 1.6 V
        Max Speed: 4000 MHz
        Current Speed: 2400 MHz
        Core Count: 4
        Core Enabled: 4
        Thread Count: 4
```

**Exemple 2: Informació de la memòria RAM**

```bash
sudo dmidecode -t memory
```

**Sortida (simplificada):**

```
Memory Device
        Size: 4096 MB
        Type: DDR4
        Speed: 2400 MT/s
        Manufacturer: Samsung
        Serial Number: 12345678
        Part Number: M378A5143DB0-CPB
```

**Exemple 3: Obtenir només el número de sèrie del sistema**

```bash
sudo dmidecode -s system-serial-number
```

### Interpretació de la sortida

La informació DMI és molt tècnica i inclou:

**Per al processador:**

- Model exacte i velocitats
- Nombre de nuclis físics i threads
- Voltatge i temperatura màxima

**Per a la memòria:**

- Tipus de RAM (DDR3, DDR4, DDR5)
- Velocitat (MHz o MT/s)
- Fabricant i número de sèrie
- Configuració dels slots

**Per al sistema:**

- Fabricant (Dell, HP, Lenovo, etc.)
- Model exacte
- Números de sèrie
- UUID del sistema

**Ús pràctic:** Aquesta informació és essencial per:

- Verificar garanties (números de sèrie)
- Planificar ampliacions (tipus de RAM compatible)
- Inventariar hardware de l'empresa

## 8. `hwinfo` - Resum complet del hardware

### Descripció

`hwinfo` és una eina molt completa que detecta i mostra informació sobre tot el hardware del sistema. És com combinar totes les comandes anteriors en una sola.

**Nota:** Pot requerir instal·lació: `sudo apt install hwinfo`

### Sintaxi

```bash
hwinfo [OPCIONS]
```

### Opcions principals

| Opció       | Descripció                    |
| ----------- | ----------------------------- |
| `--short`   | Resum curt de tot el hardware |
| `--cpu`     | Només informació de CPU       |
| `--memory`  | Només memòria                 |
| `--disk`    | Només discos                  |
| `--network` | Només targetes de xarxa       |
| `--usb`     | Només dispositius USB         |
| `--bios`    | Informació del BIOS           |

### Exemples pràctics

**Exemple 1: Resum de tot el hardware**

```bash
hwinfo --short
```

**Sortida (parcial):**

```
cpu:
                       Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz, 2400 MHz
                       Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz, 2400 MHz
keyboard:
  /dev/input/event0    AT Translated Set 2 keyboard
mouse:
  /dev/input/mice      QEMU USB Tablet
disk:
  /dev/sda             QEMU HARDDISK
  /dev/sdb             QEMU HARDDISK
network:
  enp0s3               Ethernet network interface
  lo                   Loopback network interface
```

**Exemple 2: Informació detallada de la CPU**

```bash
hwinfo --cpu
```

**Exemple 3: Informació de les targetes de xarxa**

```bash
hwinfo --network
```

**Sortida (simplificada):**

```
21: eth0  Interface
  [Created at net.126]
  SysFS ID: /class/net/eth0
  Hardware Class: network interface
  Model: "Ethernet network interface"
  Device File: eth0
  HW Address: 52:54:00:12:34:56
  Link detected: yes
```

### Interpretació de la sortida

`hwinfo` proporciona informació molt estructurada:

- **SysFS ID**: Localització al sistema de fitxers virtual
- **Hardware Class**: Tipus de dispositiu
- **Model**: Nom del dispositiu
- **Driver**: Driver del kernel en ús
- **Link detected**: Si el dispositiu està connectat/actiu

**Avantatge de hwinfo:**

- Informació completa en un sol lloc
- Detecció automàtica de tot el hardware
- Útil per generar informes complets d'inventari
