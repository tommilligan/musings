# Dell XPS 15 9500 - Wi-Fi under Ubuntu 20.04

This guide covers getting working wifi and bluetooth connectivity on the Dell XPS 15 9500, which is currently [unsupported in mainline Linux](https://forums.linuxmint.com/viewtopic.php?t=329585), let along Ubuntu 20.04.

To clarify some potentially confusing terminology further on:

- The brand of laptop is `Dell XPS 15`, the model is `9500`
- The network card is branded as `Killer Wi-fi 6 AX500-DBS (2x2)`
- The actual chip on the card we care about is a Qualcomm `QCA6390`
- The driver we are going to install is [`ath11k`](https://wireless.wiki.kernel.org/en/users/Drivers/ath11k)

## Check your hardware is visible

Your laptop is equipped with a Qualcomm QCA6390 chipset for wifi and bluetooth. It should be visible to the OS even if the drivers to make it work are not installed:

```log
$ sudo lspci -nn | grep -i qual
6c:00.0 Network controller [0280]: Qualcomm Device [17cb:1101] (rev 01)
```

The device ID `17cb:1101` confirms this is the `Killer Wi-fi 6 AX500-DBS (2x2)` which we need to install support for.

### If your hardware is not visible

If you do not see a Qualcomm device listed in the output, the OS cannot see your hardware. This indicates a hardware level fault.

You can verify this by entering the BIOS of your machine (press F2 to enter setup on boot). In my case, under hardware, the BIOS showed `Wi-Fi: {None}`.

Dell will ask you to do the following as a first diagnostic, which actually fixed the PCI connectivity for me:

1. Reboot your machine. Hit F12 on boot, to access the One Time Boot menu, then select `Diagnostics`
1. Run the simple diagnostic (~15 minutes). If prompted to run a full memory test, select No.
1. After the diagnostic passes, reboot your machine and see if your hardware has become visible.

If your hardware has not become visible, you should contact Dell support.

## Adding Wi-Fi support

To add wifi support, we need to compile and install a new linux kernel, with modular support for `ath11k` driver.

The below instructions are accurate as of 09/11/2020 and are based on [this mailing list post](http://lists.infradead.org/pipermail/ath11k/2020-November/000537.html).

### Install firmware for QCA6390

Checkout the latest version of `linux-firmware`. This guide used **commit `2d55a0a`**.

```bash
# Pull latest versions
git clone https://github.com/kvalo/ath11k-firmware
cd ath11k-firmware

# Copy firmware files to system
sudo mkdir -p /lib/firmware/ath11k/QCA6390/hw2.0/
sudo cp QCA6390/hw2.0/1.0.1/WLAN.HST.1.0.1-01740-QCAHSTSWPLZ_V2_TO_X86-1/*.bin /lib/firmware/ath11k/QCA6390/hw2.0/
sudo cp QCA6390/hw2.0/board-2.bin /lib/firmware/ath11k/QCA6390/hw2.0/
```

### Make custom linux kernel

Support for `ath11k` is available in the latest release candidates. However, it is not enabled by default.

To install support, we need to build a custom linux kernel with support enabled. Pull the source code for
the version of the kernel you want to build - this guide used `v5.10-rc2`, which is **commit `3cea11cd`**:

```bash
# Dependencies to build the linux kernel
sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev

# Checkout the source for the version we want
git clone -b v5.10-rc2 git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

#### Configure the kernel

Configuring the kernel build is a bit of a dark art. Generally, you copy the config of a kernel that already works, and tweak settings you want. You can download the [full config file I used here](./config-5.10.0-051000rc2-generic-ath11).

I copied the config from the vanilla `v5.10-rc2` kernel, and applied the following changes:

- enable `ath11k` support
- disable debug info (otherwise the kernel is massive)

```diff
$ diff -u /boot/config-5.10.0-051000rc2-generic config-5.10.0-051000rc2-generic-ath11

--- /boot/config-5.10.0-051000rc2-generic	2020-11-01 23:30:41.000000000 +0000
+++ config-5.10.0-051000rc2-generic-ath11	2020-11-06 22:28:00.927250636 +0000
@@ -3608,10 +3608,11 @@
 # CONFIG_WCN36XX_DEBUGFS is not set
 CONFIG_ATH11K=m
 # CONFIG_ATH11K_AHB is not set
-# CONFIG_ATH11K_PCI is not set
-# CONFIG_ATH11K_DEBUG is not set
-# CONFIG_ATH11K_DEBUGFS is not set
-# CONFIG_ATH11K_TRACING is not set
+CONFIG_ATH11K_PCI=m
+CONFIG_ATH11K_DEBUG=y
+CONFIG_ATH11K_DEBUGFS=y
+CONFIG_ATH11K_TRACING=y
+CONFIG_ATH11K_SPECTRAL=y
 CONFIG_WLAN_VENDOR_ATMEL=y
 CONFIG_ATMEL=m
 CONFIG_PCI_ATMEL=m
@@ -10596,12 +10597,12 @@
 #
 # Compile-time checks and compiler options
 #
-CONFIG_DEBUG_INFO=y
+# CONFIG_DEBUG_INFO is not set
 # CONFIG_DEBUG_INFO_REDUCED is not set
 # CONFIG_DEBUG_INFO_COMPRESSED is not set
 # CONFIG_DEBUG_INFO_SPLIT is not set
-CONFIG_DEBUG_INFO_DWARF4=y
-CONFIG_DEBUG_INFO_BTF=y
+# CONFIG_DEBUG_INFO_DWARF4 is not set
+# CONFIG_DEBUG_INFO_BTF is not set
 CONFIG_GDB_SCRIPTS=y
 # CONFIG_ENABLE_MUST_CHECK is not set
 CONFIG_FRAME_WARN=1024
```

If you would like to rebase these changes on a different config, install a pre-built [mainline kernel](https://kernel.ubuntu.com/~kernel-ppa/mainline/), after which the config will be available in the `/boot` directory. You can apply the above changes manually.

#### Build the kernel

Rename the config file you want to use as `.config`, which will then be picked up as part of the build:

```mv
~/Downloads/config-5.10.0-051000rc2-generic-ath11 linux/.config
```

Build the kernel with:

```bash
cd linux

# add -j 15 as you're probably on the XPS 15 and you can make it go a lot faster
make
```

This will take around an hour, so you may want to run it over lunch (not sure, didn't time it).

#### Install the kernel

First, install the new kernel modules:

```bash
sudo make modules_install
```

Now let's install the actual kernel that knows how to load these modules. If you're running Ubuntu/Debian, let's package up the kernel as '.deb' packages:

```bash
make -j 15 bindeb-pkg
```

And then install as usual packages:

```bash
cd ..
sudo dpkg -i linux-*5.10.0-rc2*.deb
```

#### Update GRUB config

In order to let us choose the kernel on next boot, we want to see the Advanced options while booting.
This is not strictly necessary, but may be necessary to switch back to your previous kernel if the new one doesn't work.

Edit your `/etc/default/grub` and ensure the config contains:

```cfg
# Instead of using the first Advanced menu entry by default, use the last selected one
# GRUB_DEFAULT=0
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true

# Show the menu for 30 seconds before booting, to allow Advanced options selection
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=30

# Don't hide the grub menu or show a nice UI
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_CMDLINE_LINUX=""
```

Then regenerate the boot config:

```bash
sudo update-grub
```

#### Reboot and enjoy!

**Please note:** Secure Boot was disabled in the BIOS when I did these steps. I believe it may be required, as we're loading unsigned kernel modules at this point. To disable Secure Boot, enter the BIOS using F2 and disable it from the 'Security' menu.

You can now shutdown your laptop. On rebooting, you should see the GRUB menu. Use the arrow keys to select **Advanced**, and then select the entry containing the kernel version (**5.10.0-rc2**).

On bootup, you should now be able to use wifi via the GNOME GUI.

## Get missing Bluetooth firmware

We seem to have lost our Bluetooth functionality now Wi-Fi is working.

### Find out what's wrong

Running `dmesg` showed that there was an issue loading Bluetooth functionality:

```log
$ sudo dmesg | grep -i blue

[   38.676906] Bluetooth: hci0: QCA controller version 0x02000200
[   38.676906] Bluetooth: hci0: QCA Downloading qca/htbtfw20.tlv
[   38.677109] bluetooth hci0: Direct firmware load for qca/htbtfw20.tlv failed with error -2
[   38.677111] Bluetooth: hci0: QCA Failed to request file: qca/htbtfw20.tlv (-2)
[   38.678679] Bluetooth: hci0: QCA Failed to download patch (-2)
[   39.042528] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
```

Searching for `htbtfw20.tlv` online turns up an entry in the `linux-firmware` repo, under the `qca` directory. Let's see what the current status of our local firmware is:

```log
$ ls /lib/firmware/qca

crbtfw21.tlv
crbtfw32.tlv
crnv21.bin
...
```

Well, that looks promising - there are other `.tlv` files in here, but no `htbtfw20.tlv`. As mentiond in the kernel release candidate testing notes, there are new hardware files required, and Ubuntu must be using a version of `linux-firmware` that doesn't have these files.

### Install the latest linux-firmware for qca

Let's pull the latest firmware files. This guide used **commit `dae4b4c`**.

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git
```

and see which files are missing from our system:

```diff
$ diff -u <(ls /usr/lib/firmware/qca) <(ls linux-firmware/qca)

--- /proc/self/fd/11	2020-11-09 09:41:47.602012054 +0000
+++ /proc/self/fd/15	2020-11-09 09:41:47.602012054 +0000
@@ -2,8 +2,13 @@
 crbtfw32.tlv
 crnv21.bin
 crnv32.bin
+crnv32u.bin
+htbtfw20.tlv
+htnv20.bin
+NOTICE.txt
 nvm_00130300.bin
 nvm_00130302.bin
+nvm_00230302.bin
 nvm_00440302.bin
 nvm_00440302_eu.bin
 nvm_00440302_i2s_eu.bin
@@ -14,6 +19,7 @@
 nvm_usb_00000302_eu.bin
 rampatch_00130300.bin
 rampatch_00130302.bin
+rampatch_00230302.bin
 rampatch_00440302.bin
 rampatch_usb_00000200.bin
 rampatch_usb_00000201.bin
```

Well, that looks sensible. Let's copy all the missing files over, minus `NOTICE.txt`:

```bash
sudo cp linux-firmware/qca/{crnv32u.bin,htbtfw20.tlv,htnv20.bin,nvm_00230302.bin,rampatch_00230302.bin} /lib/firmware/qca
```

### Restart to load new firmware

On restarting we see the issue is resolved, with us able to pair and connect with bluetooth devices via the GNOME GUI.
We can also confirm the issue no longer appears in `dmesg` logs:

```log
[   17.494711] Bluetooth: hci0: QCA controller version 0x02000200
[   17.494713] Bluetooth: hci0: QCA Downloading qca/htbtfw20.tlv
[   17.964791] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
```

## Fix broken NVIDIA modules

Nvidia kernel modules are currently broken after following the above steps. We can see this by running:

```bash
$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.
```

To fix this, we can patch the installed driver source files, then trigger DKMS to rebuild the kernel module.

### Patch your local driver source code

Locate your NVIDIA driver source code directory:

```bash
$ ls /usr/src | grep nvidia
nvidia-450.80.02
```

The major version (here, `450`) should match the installed apt package:

```bash
$ sudo apt list --installed | grep nvidia-dkms
nvidia-dkms-450/focal-updates,now 450.80.02-0ubuntu0.20.04.2 amd64 [installed,automatic]
```

If you have a different major version number, you will need to adjust the following commands accordingly. This guide uses `450`.

Download [this patch](./nvidia-fix-linux-5.10.patch) ([original gist](https://gist.github.com/joanbm/beaccedd729589df98332d70a1754e9a#file-nvidia-fix-linux-5-10-patch)), and copy it to the source `patches` directory. Then apply the patch:

```bash
cd /usr/src/nvidia-450.80.20
sudo cp ~/Downloads/nvidia-fix-linux-5.10.patch patches
sudo patch -s -p1 < patches/nvidia-fix-linux-5.10.patch
```

### Rebuild with DKMS

Now that the source is updated, we need to rebuild and reload the kernel module, which we can do with:

```bash
sudo dpkg-reconfigure nvidia-dkms-450
sudo modprobe nvidia
```

We should now have working NVIDIA kernel modules!

```bash
$ nvidia-smi
Wed Nov 25 09:51:23 2020
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.80.02    Driver Version: 450.80.02    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  GeForce GTX 165...  Off  | 00000000:01:00.0 Off |                  N/A |
| N/A   54C    P0     6W /  N/A |      0MiB /  3914MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```
