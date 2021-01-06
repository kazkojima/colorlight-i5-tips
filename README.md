# Colorlight i5 Tips

This is a tiny collection of tips for [Colorlight i5 FPGA board](https://github.com/wuxx/Colorlight-FPGA-Projects).

### Updates (Jan 6 2021)

* Change the order of ethernet PHYs.
* Add PHY configuration to BIOS.
* Add a linux patch and .dts so as to enable the interrupt on the ethernet. 

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

```
$ wget https://raw.githubusercontent.com/enjoy-digital/litex/master/litex_setup.py
$ chmod +x litex_setup.py
$ ./litex_setup.py init install --user
```

should prepare the original LiteX tree.

Colorlight i5 support can be added with replacing litex-board and litex with the above colorlight_i5 branches:

```
$ mv litex-boards litex-boards-upstream
$ git clone https://github.com/kazkojima/litex-boards.git -b colorlight_i5
$ mv litex litex-upstream
$ git clone https://github.com/kazkojima/litex.git -b colorlight_i5
```

After these steps,

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

Colorlight i5 has two Braodcom's B50612D gigabit ethernet transceiver chips. I made a simple extender with pulse transformer and RJ45 connector to test ethernet. i5ether directory includes KiCad files for it.

![ethernet adaptor](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/i5ether-adaptor.jpg)

netboot works with it, though a few tips are needed.

B50612D datasheet says that the pin 41 of the chip PHYA[0] determines the PHY Address of B50612D. It seems that the chip connected to ETH2_* connecter pins has PHYA[0] = 0 and the LiteX ether MAC works with this PHY. ~~ATM, the platform file defines this chip as the 2nd ether so as to match the connector pins.~~ The order of the two PHYs is swapped with the naming of the connectors on the board so to match with the configuration of their PHYA[0] pins. The default is "--eth-phy 0".

~~The build option "--eth-phy 1" will make ETH2_* pins the working ethernet port.~~

```
$ ./colorlight_i5.py --integrated-rom-size 0xc000 --l2-size 2048 --with-sdcard --with-ethernet
```

One another tip to get Ethernet working on your ColorLite i5.

B50612D datasheet:
"The RGMII transmit timing can be adjusted, if needed, by software or hardware control. The TXD to GTXCLK delay time can be increased by approximately 1.9 ns by setting bit 9 of register 1Ch, shadow value 00011."

TXD to GTXCLK delay is enabled by default. It looks that the ethernet packets sent from colorlight can't be seen by the other hosts with it.  Disabling it with

```
litex> mdio_write 0 0x1c 0x8c00
```

![screenshot of netboot](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/netboot.png)

Now this is added to the board specific bios initialization. There is no need to do by hand. See the [litex-boards commit](https://github.com/kazkojima/litex-boards/commit/858b62292c69aa8452d939b01cabd68fab29449f) and the [litex commit](https://github.com/kazkojima/litex/commit/dd1009d3137074f0b2816b2644a6d930432210b2).

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

Since we have only 8GB of memory, we need to change the memory map for the device table and emulator binaries. The patch in linux/ assumes rv32.dtb at 0x40780000 and emulator.bin at 0x407c0000.
There is a boot.json file in linux/ that matches this change.

![screenshot of linux boot](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/boot-on-sdcard.png)

Too bad it doesn't work SDCard clock over 2MHz which causes "DMA timeout" errors ATM.

### Ethernet interrupt

Original litex ethernet driver uses polling instead of interrupt. The tiny ugly patch(https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/linux-litex-vexriscv-rebase-eth-irq.patch) enables the ethernet interrupt, though it is NOT the right way to do it. The corresponding DT source is [here](https://github.com/kazkojima/colorlight-i5-tips/blob/main/linux/rv32-eth.dts).

### SDRAM issue on linux-on-litex-vexriscv HEAD

The linux-on-litex-vexriscv HEAD has been changed to use the VexRiscV SMP core.
After this change, colorlight-i5 has the SRRAM issue that the entire 32-bit is changed when writing bytes to SDRAM. For example, if 0x01 is written to 0x40000003 with byte write, the 32-bit word at 0x40000000 becomes 0x01010101. Oops.

The colorlight-i5 is different from other boards in that the /we (DQM0-DQM3 pins of the EM638325 SDRAM) for each byte of the 32-bit wide SDRAM are all connected to GND, which may be a problem.
I thought that access to SDRAM occurs on each cache line, so byte access does not occur.

I'm using linux-on-litex-vexriscv commit 4a44b7244422c901e74a4729eaa770624ea5eea1 so as to avoid this issue.

