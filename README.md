# Colorlight i5 Tips

This is a tiny collection of tips for [Colorlight i5 FPGA board](https://github.com/wuxx/Colorlight-FPGA-Projects).


## Quick start

See [Tom Verbeure's great article](https://tomverbeure.github.io/2021/01/22/The-Colorlight-i5-as-FPGA-development-board.html) first. It's a complete guide for the board and its environment ATM.

See also [get-start](https://github.com/wuxx/Colorlight-FPGA-Projects/blob/master/get-start.md) of the above original site. The board has the CMSIS-DAP link for debug and serial port. The site gives the "dapprog" which is a wrapper script of OpenOCD for convenience.

```
$ ./dapprog xxx.svf # Configure with xxx.svf
$ ./dapprog xxx.bit # Write configuration to the flash
```

though the 2nd one fails because the flash is protected in shipping. See SPIFLASH below. If you follows the Tom's article, this problem can be avoided with "ecpdap".

```
$ ecpdap flash unprotect
```

See 'Programming a Bitstream into SPI flash' section of his article for details.
For loading and flash writing,

```
$ ecpdap program xxx.bit       # Configure with xxx.svf
$ ecpdap flash erase           # Erase flash
$ ecpdap flash write xxx.bit   # Write configuration to the flash
```

## LiteX

Now [litex-board](https://github.com/litex-hub/litex-boards) has colorlight i5 board support. It uses [Adam Greig's ecpdap](https://github.com/adamgreig/ecpdap) as the bit stream loader written in Rust.

For LiteX, see [Quick start guide of LiteX](https://github.com/enjoy-digital/litex#quick-start-guide).

```
$ wget https://raw.githubusercontent.com/enjoy-digital/litex/master/litex_setup.py
$ chmod +x litex_setup.py
$ ./litex_setup.py init install --user
```

should prepare the LiteX tree.

After these steps,

```
$ cd litex-boards/litex_boards/targets
$ ./colorlight_i5.py --with-sdcard --integrated-rom-size 0xc000 --l2-size 2048
```

for example, will generate .v and .lpf files in litex-boards/litex_boards/targets/build/colorlight_i5/gateware directory.

```
$ cd build/colorlight_i5/gateware
$ /bin/sh ./build_colorlight_i5.sh
```

will build .svf and .bin stream files.

With --build option

```
$ ./colorlight_i5.py --with-sdcard --with-spi --integrated-rom-size 0xc000 --l2-size 2048 --build
```

build_colorlight_i5.sh runs automatically. If ecpdap is installed already, --load option programs FPGA with the stream built

```
$ ./colorlight_i5.py --with-sdcard --with-spi --integrated-rom-size 0xc000 --l2-size 2048 --load
```


## SPIFLASH

[The start guide](https://github.com/wuxx/Colorlight-FPGA-Projects/blob/master/get-start.md) says that the SPI-Flash on i5 modules GD25Q16, is locked. You can free the lock with ecpdap.

## SDCARD

I've tested DIGILENT MicroSD PMOD.

![screenshot of LiteX BIOS](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/uSD-pmod.jpg)

It seems that the pull-up is a bit weak to work with the higher clock. An added 1k pull-up register to MOSI/CMD line works for me.

![added 1kohm pull-up register](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/uSD-pmod-pullup.jpg)

## Ethernet

Colorlight i5 has two Braodcom's B50612D gigabit ethernet transceiver chips. I made a simple extender with pulse transformer and RJ45 connector to test ethernet. i5ether directory includes KiCad files for it.

![ethernet adaptor](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/i5ether-adaptor.jpg)

netboot works with it.

B50612D datasheet says that the pin 41 of the chip PHYA[0] determines the PHY Address of B50612D. It seems that the chip connected to ETH2_* connecter pins has PHYA[0] = 0 and the LiteX ether MAC works with this PHY. ~~ATM, the platform file defines this chip as the 2nd ether so as to match the connector pins.~~ The order of the two PHYs is swapped with the naming of the connectors on the board so to match with the configuration of their PHYA[0] pins. The default is "--eth-phy 0".

~~The build option "--eth-phy 1" will make ETH2_* pins the working ethernet port.~~

```
$ ./colorlight_i5.py --integrated-rom-size 0xc000 --l2-size 2048 --with-sdcard --with-ethernet
```

## HDMI

The build option "--with-video-framebuffer" (resp. "--with-video-terminal") makes the experimental support for video frame buffer (resp. video terminal) enable. The configuration of the HDMI output pins on the Muse-lab development board are assumed.

![video frame buffer](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/video-fb.png)

![video terminal](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/video-terminal.png)

--with-video-framebuffer and --with-video-terminal will consume some RAM blocks of ECP5. If these options are used with --with-ethernet, for example, the build may fail due to the shortage of BRAM(DP16KD). One could reduce the usage of BRAM with specifying the lower values of integrated-sram-size and l2-size. Here are some examples.

```
$ ./colorlight_i5.py --with-ethernet --with-video-framebuffer --integrated-rom-size 0xa000 --integrated-sram-size 0x1000 --l2-size 2048 --build --load

$ ./colorlight_i5.py --with-ethernet --with-video-terminal --integrated-rom-size 0xa000 --integrated-sram-size 0x1000 --l2-size 1024 --build --load
```

The video fame buffer has a data skew issue on colorlight i5 ATM. It can be avoided temporarily with the one liner below:

```
diff --git a/litex_boards/targets/colorlight_i5.py b/litex_boards/targets/colorlight_i5.py
index 58aacfc..f081f43 100755
--- a/litex_boards/targets/colorlight_i5.py
+++ b/litex_boards/targets/colorlight_i5.py
@@ -150,7 +150,7 @@ class BaseSoC(SoCCore):
                 size                    = kwargs.get("max_sdram_size", 0x40000000),
                 l2_cache_size           = kwargs.get("l2_size", 8192),
                 l2_cache_min_data_width = kwargs.get("min_l2_data_width", 128),
-                l2_cache_reverse        = True
+                l2_cache_reverse        = False if with_video_framebuffer else True
             )
 
         # Ethernet / Etherbone ---------------------------------------------------------------------
```

## Zephyr

Links:

[Minimal Zephyr OS on RISC-V and other tests](https://blog.pcbxprt.com/index.php/2020/07/26/minimal-zephyr-os-on-risc-v-and-other-tests/)

[his github repo](https://github.com/ghent360/riscvOnColorlight-5A-75B)

Install Zephyr and make a modification for colorlight as written in the blog.

Although it may not be necessary in your environment, on my Ubuntu 20.04 TLS system, I needed a complete python-3.7 environment for building zephyr itself and its examples. It can be done

```
$ conda create --name py37 python=3.7
$ conda activate py37
```

if you have installed Anaconda.

Zephyr's Litex-vexrisc support expects 8-bit CSRs as well as linux. OTOH LiteX produces a 32-bit CSR SoC by default, so that needs to be changed with --csr-data-width 8 option.

```
$ ./colorlight_i5.py --with-sdcard --with-ethernet --integrated-rom-size 0xc000 --l2-size 2048 --csr-data-width 8
```

Also 2 DT-related zephyr's files

```
dts/riscv/riscv32-litex-vexriscv.dtsi
boards/riscv/litex_vexriscv/litex_vexriscv.dts
```

need to be modified for the changes in the configuration of the SoC, frequencies and the address of the CSRs. colorlight_i5.py generates build/colorlight_i5/software/include/generated/csr.h which contains the CSR's address information when building SoC RTL.

The simple example like blinky works successfully, though it looks some litex drivers in zephyr are WIP and not full-featured.

#### Updates for Zephyr

I've pushed [zephyr-rtos branch with modified DTS and default configs for colorlight-i5](https://github.com/kazkojima/zephyr/tree/colorlight_i5).

You can try net samples with it.

```
$ cd zephyrproject/zephyr/samples/net/telnet
$ mkdir build
$ cd build
$ cmake -DBOARD=litex_vexriscv ..
$ make menuconfig # optional
$ make
```

makes zephyr/zephyr.bin binary which can be used as a litex boot.bin.

![screenshot of samples/net/telnet](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/telnet-app.png)

## linux-on-litex-vexriscv

In the following sections, linux-on-litex-vexriscv HEAD and litex-hub/linux litex-rebase branch HEAD are assumed.

Due to the 8MB memory limit, it's difficult to put the root filesystem on the RAM like other boards. Putting it on NFS or micro SD will be a good candidate. Linux can boot on these two settings, though it's very slow. The main reason would be that there is not enough physical memory to keep a sufficient working set.

Since we have only 8GB of memory, we need to change the memory map for the device table and emulator binaries. The patch in linux/ assumes rv32.dtb at 0x40780000 and opensbi.bin at 0x407c0000.
There is a boot.json file in linux/ that matches this change.

### sdcard root

To make the micro SD the root FS, I'm using the litex-rebase branch of [litex's linux tree](https://github.com/litex-hub/linux.git), [a tiny patch for mmc](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/linux-litex-vexriscv-rebase-mmc.patch) and [.config for mmc](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/defconfig-colorlight-i5-mmc) and [linux-on-litex-vexriscv](https://github.com/litex-hub/linux-on-litex-vexriscv).

linux-on-litex-vexriscv includes some kernel patches required for litex.

```
$ for f in YOUR_linux-on-litex-vexriscv_PATH/buildroot/patches/linux/000[124]*.patch; do (cat $f | patch -p1); done
```

in the linux source directory will add these patches, ATM. Without them, the resulted kernel size is over 20GB!!

Currently colorlight i5 sdcard support is experimental and isn't yet merged. [A tiny patch](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/linux-on-litex-vexriscv-mmc.patch) is needed for linux-on-litex-vexriscv.

```
$ cd YOUR_linux-on-litex-vexriscv_PATH
$ cat linux-on-litex-vexriscv-mmc.patch | patch -p1
$ ./make.py --board colorlight_i5 --build
```

![screenshot of linux boot](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/boot-on-sdcard.png)

### Ethernet/nfsroot

Almost similar to sdcard root, except no mmc patch for kernel isn't required and [DT for ethernet/nfsroot](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/rv32-eth.dts)  [.config for nfsroot](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/defconfig-colorlight-i5-eth) have worked. 

~~### SDRAM issue on linux-on-litex-vexriscv HEAD~~

This is solved with [commit dee4331](https://github.com/litex-hub/linux-on-litex-vexriscv/commit/dee4331dc39124bca37f8f765ac20184ad447ae4)!

### OpenSBI

A tiny patch [opensbi-config-fixmap.patch](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/opensbi-config-fixmap.patch) for OpenSBI is needed so as to match 8MB RAM.

```
$ clone https://github.com/litex-hub/opensbi --branch 0.8-linux-on-litex-vexriscv
$ cd opensbi
$ cat opensbi-config-fixmap.patch | patch -p1
$ make CROSS_COMPILE=riscv32-unknown-elf- PLATFORM=litex/vexriscv
$ cp build/platform/litex/vexriscv/firmware/fw_jump.bin opensbi.bin
```

will give opensbi.bin that works on the 8MB memory map.

## Litescope

TODO

## History

### Updates (Mar 26 2021)

* Add HDMI section.
* Remove the old spiflash stuff.

### Updates (Feb 16 2021)

* Fix ecpdap command to write flash.
* Add example command lines to build a zephyr sample.

### Updates (Jan 30 2021)

* Add an OpenSBI subsection.

### Updates (Jan 29 2021)

* Update for merging colorlight i5 board support.
* uSD-pmod pull-up register issue.

### Updates (Jan 27 2021)

* Adjust for linux-on-litex-vexriscv HEAD.
* Remove 'Reset switch' section.

### Updates (Jan 19 2021)

* Add about forked Zephyr OS with colorlight_i5 branch.

### Updates (Jan 15 2021)

* Fix initial builds.

### Updates (Jan 6 2021)

* Change the order of ethernet PHYs.
* Add a PHY configuration to BIOS.
* Add a linux patch and .dts so as to enable the interrupt on the ethernet. 

