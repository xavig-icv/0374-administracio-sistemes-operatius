## 03. Informació del Sistema i del Maquinari

Aquest fitxer respon les dues primeres preguntes de l'inventari d'un servidor: **quin sistema és** i **quins recursos té**. Els exemples són del servidor `srv0374`, una màquina virtual de VirtualBox amb Ubuntu Server 24.04. Les vostres sortides tindran valors diferents.

| Ordre                  | Informació                                         | `sudo` |
| ---------------------- | -------------------------------------------------- | ------ |
| `uname`                | Kernel i arquitectura                              | No     |
| `hostnamectl`          | Nom de l'equip, sistema, kernel, virtualització    | No     |
| `lscpu`                | Processador                                        | No     |
| `free`                 | Memòria RAM i swap                                 | No     |
| `lsblk`, `df`, `du`    | Discos, particions i espai ocupat                  | No     |
| `lspci`, `lsusb`       | Dispositius PCI i USB                              | No     |
| `dmidecode`            | Firmware, placa base, mòduls de RAM                | **Sí** |

### Sistema Operatiu i Kernel

**`uname`** mostra informació del kernel:

| Opció | Informació                                        |
| ----- | ------------------------------------------------- |
| `-n`  | Nom de l'equip (*hostname*)                       |
| `-r`  | Versió (*release*) del kernel                     |
| `-v`  | Compilació del kernel                             |
| `-m`  | Arquitectura de la màquina                        |
| `-a`  | Tota la informació                                |

```bash
uname -r
```

```
6.8.0-87-generic
```

`6.8.0` és la versió del kernel, `87` la revisió del paquet d'Ubuntu (cada actualització de seguretat la incrementa) i `generic` la variant del kernel.

**32 o 64 bits?** `uname -m` mostra `x86_64` en un sistema de 64 bits i `i686` en un de 32 bits. També es pot consultar amb `getconf LONG_BIT` (`64`).

**`hostnamectl`** reuneix en una sola ordre el nom de l'equip, el sistema operatiu, el kernel i el maquinari:

```
 Static hostname: srv0374
       Icon name: computer-vm
         Chassis: vm
      Machine ID: 4f1c9e2a7b3d4e5f8a9b0c1d2e3f4a5b
         Boot ID: 9b2e7d1c3a4f4b5c8d6e0f1a2b3c4d5e
  Virtualization: oracle
Operating System: Ubuntu 24.04.3 LTS
          Kernel: Linux 6.8.0-87-generic
    Architecture: x86-64
 Hardware Vendor: innotek GmbH
  Hardware Model: VirtualBox
Firmware Version: VirtualBox
```

| Camp                                | Significat                                                        |
| ----------------------------------- | ----------------------------------------------------------------- |
| `Static hostname`                   | Nom de l'equip (`/etc/hostname`)                                  |
| `Chassis`                           | Tipus d'equip: `vm` (màquina virtual), `desktop`, `laptop`, `server` |
| `Virtualization`                    | Hipervisor: `oracle` (VirtualBox), `kvm`, `vmware`, `microsoft` (Hyper-V). No apareix en una màquina física |
| `Operating System`                  | Distribució i versió                                              |
| `Hardware Vendor` / `Hardware Model`| Fabricant i model de l'equip. En una VM de VirtualBox: `innotek GmbH` i `VirtualBox` |

Per canviar el nom de l'equip (per exemple, per posar-hi `srv0374`):

```bash
sudo hostnamectl set-hostname srv0374
sudo nano /etc/hosts               # canviar també el nom a la línia 127.0.1.1
```

**Versió de la distribució:**

```bash
grep -E "^(PRETTY_NAME|VERSION_CODENAME)" /etc/os-release
lsb_release -a
```

```
PRETTY_NAME="Ubuntu 24.04.3 LTS"
VERSION_CODENAME=noble
```

`LTS` (*Long Term Support*) indica que la versió té 5 anys d'actualitzacions de seguretat. És la que s'utilitza als servidors.

**Màquina física o virtual?** A més de `hostnamectl`, `systemd-detect-virt` retorna només el nom de l'hipervisor (`oracle`) o `none` en una màquina física.

### Processador: `lscpu`

```bash
lscpu
```

```
Architecture:             x86_64
  CPU op-mode(s):         32-bit, 64-bit
  Address sizes:          39 bits physical, 48 bits virtual
  Byte Order:             Little Endian
CPU(s):                   2
  On-line CPU(s) list:    0,1
Vendor ID:                GenuineIntel
  Model name:             Intel(R) Core(TM) i5-10400 CPU @ 2.90GHz
    CPU family:           6
    Model:                165
    Thread(s) per core:   1
    Core(s) per socket:   2
    Socket(s):            1
    Flags:                fpu vme de pse tsc msr pae ... lm ... hypervisor ...
Virtualization features:
  Hypervisor vendor:      KVM
  Virtualization type:    full
Caches (sum of all):
  L1d:                    64 KiB (2 instances)
  L1i:                    64 KiB (2 instances)
  L2:                     512 KiB (2 instances)
  L3:                     12 MiB (1 instance)
```

| Camp                  | Significat                                                                |
| --------------------- | ------------------------------------------------------------------------- |
| `CPU op-mode(s)`      | Modes que **suporta** el processador (32 i 64 bits). El sistema instal·lat es mira amb `uname -m` |
| `CPU(s)`              | CPU lògiques = `Socket(s)` × `Core(s) per socket` × `Thread(s) per core`   |
| `Thread(s) per core`  | `2` indica *Hyper-Threading* (Intel) o SMT (AMD): cada nucli executa dos fils |
| `Model name`          | Model i freqüència nominal del processador                                |
| `Caches`              | Memòria cau del processador (L1, L2, L3)                                  |
| `Hypervisor vendor`   | Només apareix en una màquina virtual                                      |
| `Virtualization`      | `VT-x` (Intel) o `AMD-V`: el processador pot executar màquines virtuals   |
| `Flags`               | Funcions del processador (`lm` = 64 bits, `vmx`/`svm` = virtualització)   |

**Particularitats d'una màquina virtual:**

- `CPU(s)` mostra les CPU **assignades a la VM** (2), no les del processador físic. En canvi, `Model name` és el del processador real de l'ordinador amfitrió.
- VirtualBox ofereix al sistema convidat una interfície de paravirtualització de tipus KVM. Per això `lscpu` mostra `Hypervisor vendor: KVM` encara que `hostnamectl` digui `oracle`.
- La línia `Virtualization: VT-x` normalment no apareix: VirtualBox no passa aquesta funció a la VM si no s'activa la virtualització imbricada (*nested*).

```bash
nproc                                          # nombre de CPU lògiques disponibles
lscpu | grep -E "^CPU\(s\)|Model name|Thread|Core|Socket"
grep -m1 "cpu MHz" /proc/cpuinfo               # freqüència actual
grep -owE "vmx|svm" /proc/cpuinfo | sort -u    # buit si no hi ha virtualització per maquinari
```

### Memòria: `free`

```bash
free -h
```

```
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       412Mi       2.9Gi       1.1Mi       720Mi       3.4Gi
Swap:          3.8Gi          0B       3.8Gi
```

Per a l'inventari interessen els totals: **3,8 GiB de RAM** i **3,8 GiB de swap**. La columna important per saber si en queda és `available`, no `free`. La interpretació completa de la memòria es veu a [10. Monitorització de Recursos](./10_monitoritzacio.md).

`Nota:` `free -h` mostra els valors en GiB (base 1024). Una VM amb 4096 MB assignats mostra `3.8Gi` perquè el kernel en reserva una part.

### Discos i Particions: `lsblk`, `df` i `du`

**`lsblk`** mostra els dispositius de bloc (discos, particions, volums) en forma d'arbre:

```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 11.5G  0 lvm  /
sr0                        11:0    1 1024M  0 rom
```

| Columna        | Significat                                                                  |
| -------------- | --------------------------------------------------------------------------- |
| `NAME`         | `sda` = primer disc, `sda1` = primera partició, `sr0` = unitat òptica        |
| `RM`           | `1` si és extraïble (USB, CD)                                               |
| `SIZE`         | Mida                                                                         |
| `TYPE`         | `disk`, `part` (partició), `lvm` (volum lògic), `rom`, `loop`               |
| `MOUNTPOINTS`  | On està muntat. Si és buit, no s'utilitza directament                       |

En aquest exemple el disc de 25 GB té la partició `sda3` de 23 GB, però el volum lògic arrel (`/`) només en fa servir 11,5 GB. L'instal·lador d'Ubuntu Server sovint deixa espai lliure al grup de volums per ampliar-lo més endavant (vegeu la resolució de problemes).

| Opció                         | Informació                                                      |
| ----------------------------- | --------------------------------------------------------------- |
| `lsblk -f`                    | Sistema de fitxers (`ext4`, `vfat`, `swap`, `LVM2_member`), UUID i espai lliure |
| `lsblk -d -o NAME,SIZE,MODEL,ROTA` | Només discos, amb el model i el tipus (`ROTA` 1 = disc mecànic, 0 = SSD) |

**`df`** mostra l'espai ocupat de cada sistema de fitxers muntat:

```bash
df -hT -x tmpfs                    # -T tipus, -x tmpfs amaga els sistemes de fitxers en memòria
```

```
Filesystem                        Type  Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv ext4   12G  5.1G  5.8G  47% /
/dev/sda2                         ext4  2.0G  101M  1.7G   6% /boot
```

**`du`** mostra quant ocupa un directori:

```bash
sudo du -sh /var/log                               # total d'un directori
sudo du -h --max-depth=1 /var | sort -rh | head    # subdirectoris de /var, de més gran a més petit
```

### Dispositius: `lspci` i `lsusb`

**`lspci`** llista els dispositius connectats al bus PCI (controladors de disc, targetes de xarxa i gràfiques):

```
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma] (rev 02)
00:01.1 IDE interface: Intel Corporation 82371AB/EB/MB PIIX4 IDE (rev 01)
00:02.0 VGA compatible controller: VMware SVGA II Adapter
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
00:04.0 System peripheral: InnoTek Systemberatung GmbH VirtualBox Guest Service
00:08.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
00:0d.0 SATA controller: Intel Corporation 82801HM/HEM (ICH8M/ICH8M-E) SATA Controller [AHCI mode] (rev 02)
```

El nom de les interfícies de xarxa surt de la seva posició al bus PCI: `enp0s3` = **en** (Ethernet) + **p0** (bus 0) + **s3** (ranura 3), és a dir, el dispositiu `00:03.0`. La segona targeta (`00:08.0`) és `enp0s8`.

| Opció             | Informació                                                       |
| ----------------- | ---------------------------------------------------------------- |
| `lspci -k`        | Controlador (*driver*) del kernel que fa servir cada dispositiu  |
| `lspci -nn`       | Identificadors numèrics de fabricant i dispositiu (`[8086:100e]`) |
| `lspci -v`        | Detall de cada dispositiu                                        |

```bash
lspci -k -s 00:03.0
```

```
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
	Subsystem: Intel Corporation PRO/1000 MT Desktop Adapter
	Kernel driver in use: e1000
	Kernel modules: e1000
```

**`lsusb`** llista els dispositius USB (`lsusb -t` els mostra en arbre):

```
Bus 001 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 001 Device 002: ID 80ee:0021 VirtualBox USB Tablet
```

### Firmware i Placa Base: `dmidecode`

`dmidecode` llegeix les taules DMI/SMBIOS del firmware: fabricant, model, número de sèrie, BIOS i mòduls de memòria. Necessita `sudo`.

| Opció                  | Informació                                         |
| ---------------------- | -------------------------------------------------- |
| `-t bios`              | Fabricant, versió i data del firmware              |
| `-t system`            | Fabricant, model, número de sèrie i UUID de l'equip |
| `-t baseboard`         | Placa base                                         |
| `-t processor`         | Processador                                        |
| `-t memory`            | Mòduls de RAM: mida, tipus, velocitat, ranura      |
| `-s paraula_clau`      | Un sol valor (`dmidecode -s` sense paraula mostra la llista) |

```bash
sudo dmidecode -s system-manufacturer      # innotek GmbH
sudo dmidecode -s system-product-name      # VirtualBox
sudo dmidecode -s bios-version             # VirtualBox
sudo dmidecode -t memory | grep -E "Size|Type:|Speed"
```

En un equip físic, la informació de memòria mostra cada mòdul:

```
	Size: 8 GB
	Type: DDR4
	Speed: 3200 MT/s
	Size: No Module Installed
```

`No Module Installed` indica una ranura buida: es pot ampliar la memòria sense retirar mòduls.

`Nota:` En una màquina virtual el firmware és simulat per l'hipervisor. El fabricant, el model i la versió de la BIOS són els de VirtualBox, i la informació de la memòria pot ser incompleta o no aparèixer. Compareu la sortida de la vostra VM amb la d'un equip físic de l'aula.

**Resum de tot el maquinari:** `sudo lshw -short` (o `hwinfo --short`, paquet `hwinfo`) mostra tots els components en una sola llista.

### Resolució de Problemes

| Problema                                                        | Causa                                                    | Solució                                                     |
| --------------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------- |
| `dmidecode` mostra `Permission denied`                          | Necessita accedir al firmware                            | `sudo dmidecode`                                            |
| `lspci: command not found`                                      | Paquet no instal·lat                                     | `sudo apt install pciutils`                                 |
| `df` mostra `/` gairebé ple, però el disc és més gran           | El volum lògic arrel no fa servir tot el disc            | `sudo vgs` (columna `VFree`) i `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv` |
| `lscpu` no mostra `Virtualization: VT-x`                        | L'hipervisor no ofereix la virtualització a la VM        | És normal en una VM. A l'amfitrió sí que hi ha de ser       |
| Després de `hostnamectl set-hostname`, `sudo` avisa `unable to resolve host` | `/etc/hosts` encara té el nom antic          | Canviar el nom a la línia `127.0.1.1` de `/etc/hosts`       |
| El *prompt* encara mostra el nom antic                          | El shell llegeix el nom en iniciar la sessió            | Tancar la sessió i tornar-la a obrir                        |
