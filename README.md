
# tty0tty - linux null modem emulator v1.2 


## Quick start — 100 virtual ports + friendly names

This checkout is [ecat-community/tty0tty](https://github.com/ecat-community/tty0tty) plus a small customization that:

- defaults the driver to **100 virtual serial ports** (50 null-modem pairs), and
- exposes **library-friendly aliases** `/dev/ttyUSB100`…`/dev/ttyUSB199` — many serial libraries (e.g. pyserial's `comports()`) only auto-list `ttyUSB*` / `ttyACM*` / `ttyS*`, not the bare `tnt*` names.

### One-command install

```bash
sudo ./install_tty0tty.sh
```

`install_tty0tty.sh` is path-relative (it resolves its own location via `BASH_SOURCE`), so it runs correctly from any directory and survives the folder being moved. It performs: build → install `.ko` + base udev rule → `depmod` → write `pairs=50` modprobe option → enable boot-time autoload → generate alias rules → reload udev and swap the module → verify.

### Resulting devices

Original kernel names (50 null-modem pairs, consecutive indices are cross-connected):

```
/dev/tnt0  ─┐  pair 0    (TX↔RX, RTS↔CTS, DTR↔DSR/CD)
/dev/tnt1  ─┘
/dev/tnt2  ─┐  pair 1
/dev/tnt3  ─┘
…                          (50 pairs total)
/dev/tnt98 ─┐  pair 49
/dev/tnt99 ─┘
```

Each `/dev/tntN` additionally gets an alias (both names point to the same device):

```
/dev/ttyUSB100 -> /dev/tnt0      /dev/ttyUSB101 -> /dev/tnt1      (pair 0)
/dev/ttyUSB102 -> /dev/tnt2      /dev/ttyUSB103 -> /dev/tnt3      (pair 1)
…
/dev/ttyUSB198 -> /dev/tnt98     /dev/ttyUSB199 -> /dev/tnt99     (pair 49)
```

Aliases start at **100** to stay clear of real USB-serial adapters already on this machine (`ttyUSB0`, `ttyUSB10`…`ttyUSB17`). Use whichever name your tool recognizes.

### Configuring the count / aliases (no source edit)

The port count comes from the module's built-in `pairs` parameter (`devices = 2 × pairs`, clamped to 1…128). Override per-run via env vars:

```bash
sudo -E PAIRS=16 ./install_tty0tty.sh                            # 32 ports
sudo -E ALIAS_PREFIX=ttyACM ALIAS_OFFSET=0 ./install_tty0tty.sh  # /dev/ttyACM0..
```

Or change the live count without rebuilding:

```bash
sudo modprobe -r tty0tty && sudo modprobe tty0tty pairs=16
```

### Reboot behaviour

| Written to disk by the script | Survives reboot? |
|---|---|
| `/lib/modules/<kernel>/extra/tty0tty.ko` | ✅ same kernel only |
| `/etc/udev/rules.d/50-tty0tty.rules` (permissions) | ✅ |
| `/etc/udev/rules.d/51-tty0tty-aliases.rules` (symlinks) | ✅ |
| `/etc/modprobe.d/tty0tty.conf` (`options tty0tty pairs=50`) | ✅ |
| `/etc/modules-load.d/tty0tty.conf` (boot-time autoload) | ✅ |

A plain reboot restores all 100 ports **and** aliases automatically — no need to re-run the script. **Exception:** a kernel update changes `uname -r`, so the installed `.ko` (under the old `/lib/modules/<old>/`) is no longer found → re-run `install_tty0tty.sh`. Fully hands-off rebuilds across kernel updates would require DKMS (see *Debian package* below).

### What changed vs. the upstream fork

Only one source file differs from ecat-community/tty0tty — the udev rule `module/50-tty0tty.rules`:

```diff
-KERNEL=="tnt[0-9]", GROUP="dialout"     # matched only tnt0..tnt9
+KERNEL=="tnt[0-9]*", GROUP="dialout"    # matches tnt0..tnt99 (required for >10 ports)
```

The 100-port count, the `ttyUSB*` aliases and the reboot persistence are all delivered by `install_tty0tty.sh` and the on-disk config files it writes — the driver source itself is stock.

---

The tty0tty directory tree is divided in:

  **module** - linux kernel module null-modem  
  **pts** - null-modem using ptys (without handshake lines)


## Null modem pts (unix98): 

  When run connect two pseudo-ttys and show the connection names:
  
  (/dev/pts/1) <=> (/dev/pts/2)  

  the connection is:
  
  TX -> RX  
  RX <- TX  



## Module:

 The module is tested in kernel 3.10.2 (debian) 

  When loaded, create 8 ttys interconnected:
  
  /dev/tnt0  <=>  /dev/tnt1  
  /dev/tnt2  <=>  /dev/tnt3  
  /dev/tnt4  <=>  /dev/tnt5  
  /dev/tnt6  <=>  /dev/tnt7  

  the connection is:
  
  TX   ->  RX  
  RX   <-  TX  
  RTS  ->  CTS  
  CTS  <-  RTS  
  DSR  <-  DTR  
  CD   <-  DTR  
  DTR  ->  DSR  
  DTR  ->  CD  
  

## Requirements:

  For building the module kernel-headers or kernel source are necessary.

## Installation:

Download the tty0tty package from one of these sources:
Clone the repo https://github.com/freemed/tty0tty

```
git clone https://github.com/freemed/tty0tty
```

Build the kernel module from provided source

```
cd tty0tty-1.2/module
make
```

Install the new kernel module into the kernel modules directory

```
sudo make modules_install
```

NOTE: if module signing is enabled, in order for depmop to complete, you may
need to create file certs/x509.genkey in the kernel modules include directory
and generate file signing_key.pem using openssl, guides are available online.

Appropriate permissions are provided thanks to a udev rule located under:

```
/etc/udev/rules.d/50-tty0tty.rules
```

NOTE: you need to add yourself to the dialout group (and do a full relog), with:

```
sudo usermod -a -G dialout ${USER}
```

Load the module

```
sudo udevadm control --reload-rules
sudo depmod
sudo modprobe tty0tty
```

You should see new serial ports in ```/dev/``` (```ls /dev/tnt*```)
You can now access the serial ports as /dev/tnt0 (1,2,3,4 etc) Note that the consecutive ports are interconnected. For example, /dev/tnt0 and /dev/tnt1 are connected as if using a direct cable.

Persisting across boot:

edit the file /etc/modules (Debian) or /etc/modules.conf

```
nano /etc/modules
```
and add the following line:

```
tty0tty
```

Note that this method will not make the module persist over kernel updates so if you ever update your kernel, make sure you build tty0tty again repeat the process.

## Debian package

In order to build the dkms Debian package

```
sudo apt-get update && sudo apt-get install -y dh-make dkms build-essential
debuild -uc -us
```

## Yocto package

In order to integrate tty0tty into Yocto, please copy contents of 'yocto' folder add to your local.conf

```
IMAGE_INSTALL_append = " \
 tty0tty-module \
"
```
