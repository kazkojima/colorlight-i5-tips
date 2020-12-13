# Colorlight i5 Tips

This is a tiny collection of tips for [Colorlight i5 FPGA board](https://github.com/wuxx/Colorlight-FPGA-Projects).

## Quick start

See [get-start](https://github.com/wuxx/Colorlight-FPGA-Projects/blob/master/get-start.md) of the above original site. The board has the CMSIS-DAP link for debug and serial port. The site gives the "dapprog" which is a wrapper script of OpenOCD for convenience.

```
$ ./dapprog xxx.svf # Configure with xxx.svf
$ ./dapprog xxx.bit # Write configuration to the flash
```

though the 2nd one fails because the flash is protected in shipping. See SPIFLASH below.

## LiteX

[A trial litex-board platform/target definitions for colorlight i5](https://github.com/kazkojima/litex-boards/tree/colorlight_i5)

[A trial litex adjustment for colorlight i5](https://github.com/kazkojima/litex/tree/colorlight_i5)

The latter includes some new flash commands. They help you to clear block protection of spiflash which is set in shipping.

![screenshot of LiteX BIOS](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/flash-cmd-added.jpg)

For LiteX, see [Quick start guide of LiteX](https://github.com/enjoy-digital/litex#quick-start-guide).

Colorlight i5 support can be added with replacing litex-board and litex with the above colorlight_i5 branches. After the general setup,

```
$ cd litex-boards/litex_boards/targets
$ ./colorlight_i5.py --with-sdcard --with-spi --integrated-rom-size 0xc000 --l2-size 2048
```

for example, will generate .v and .lpf files in litex-boards/litex_boards/targets/build/colorlight_i5/gateware directory.

```
$ cd build/colorlight_i5/gateware
$ /bin/sh ./build_colorlight_i5.sh
```

will build .svf and .bin stream files.

## SDRAM

[The original page](https://github.com/wuxx/Colorlight-FPGA-Projects#sdram-u18) says BA1 pin of SDRAM is connected to GND which means only 4MB enabled. It looks it isn't the case and is connected to C8 instead of GND. Now all 8MB is available.

## SPIFLASH

[The start guide](https://github.com/wuxx/Colorlight-FPGA-Projects/blob/master/get-start.md) says that the SPI-Flash on i5 modules GD25Q16, is locked. You can free the lock with the new flash command added to LiteX BIOS.

```
litex> flash_write_protect 0
```

should clear all block protection of GD25B16C flash chip. The modified .bit and .svf of LiteX are [here](https://github.com/kazkojima/colorlight-i5-tips/streams). If it works,

```
litex> flash_erase
```

erases GD25B16C and you can reconfigure the board with "dapprog xxx.bit".

```
litex> flash_write_protect 7
```

will lock all blocks again.

## SDCARD

I've tested DIGILENT MicroSD PMOD successfully.

![screenshot of LiteX BIOS](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/uSD-pmod.jpg)

## Ethernet

I can get nothing yet :-(

## Reset switch

I've defined reset button as K18 pin

```
    # Reset button
    ("cpu_reset_n", 0, Pins("K18"), IOStandard("LVCMOS33"), Misc("PULLMODE=UP")),
```

But this doesn't reset CPU. ???

Currently I need a change generated .v file by hand:

```
$ sed "s/assign main_cpu_reset = main_soccontroller_reset;/assign main_cpu_reset = main_soccontroller_reset | (~cpu_reset_n);/" colorlight_i5.v > /tmp/colorlight_i5.v
$ mv /tmp/colorlight_i5.v colorlight_i5.v
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
$ ./colorlight_i5.py --with-sdcard --with-spi --integrated-rom-size 0xc000 --l2-size 2048 --csr-data-width 8
```

Also 2 DT-related zephyr's files

```
dts/riscv/riscv32-litex-vexriscv.dtsi
boards/riscv/litex_vexriscv/litex_vexriscv.dts
```

need to be modified for the changes in the configuration of the SoC, frequencies and the address of the CSRs. colorlight_i5.py generates build/colorlight_i5/software/include/generated/csr.h which contains the CSR's address information when building SoC RTL.

The simple example like blinky works successfully, though it looks some litex drivers in zephyr are WIP and not full-featured.

## linux-on-litex-vexriscv

Due to the 8MB memory limit, it's difficult to put the root filesystem on the RAM like other boards. Putting it on NFS or micro SD will be a good candidate.

The latter, making the micro SD the root FS, is partially successful. Using the litex-vexriscv-rebase branch of [litex's linux tree](https://github.com/litex-hub/linux.git), [a tiny patch for mmc](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/linux-litex-vexriscv-rebase-mmc.patch) and [.config](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/defconfig-colorlight-i5), [linux-on-litex-vexriscv](https://github.com/litex-hub/linux-on-litex-vexriscv)
 with [colorlight i5 patch](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/linux-on-litex-vexriscv-colorlight-i5.patch) has booted, though it's very slooooow.

![screenshot of linux boot](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/boot-on-sdcard.png)

Too bad it doesn't work SDCard clock over 2MHz which causes "DMA timeout" errors ATM.
