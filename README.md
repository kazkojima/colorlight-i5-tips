# Colorlight i5 Tips

This is a tiny collection of tips for [Colorlight i5 FPGA board](https://github.com/wuxx/Colorlight-FPGA-Projects).

## LiteX

[A trial litex-board platform/target definetions for colorlight i5](https://github.com/kazkojima/litex-boards/tree/colorlight_i5)

[A trial litex adjustment for colorlight i5](https://github.com/kazkojima/litex/tree/colorlight_i5)

The latter includes some new flash commands. They help you to clear block protection of spiflash which is set at shipment.

![screenshot of LiteX BIOS](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/flash-cmd-added.jpg)

For LiteX, see [Quick start guide of LiteX](https://github.com/enjoy-digital/litex#quick-start-guide).

Colorlight support can be added with replacing litex-board and litex with the above colorlight_i5 branches. After the general setup,

```
$ cd litex-boards/litex_boards/targets
$ ./colorlight_i5.py --with-sdcard --with-spi --integrated-rom-size 0xc000 --l2-size 2048
```

will generate .v file in litex-boards/litex_boards/targets/build/colorlight_i5/gateware directory.

## SDRAM

[The original page](https://github.com/wuxx/Colorlight-FPGA-Projects#sdram-u18) says BA1 pin of SDRAM is connected to GND which means only 4MB enabled. It looks it isn't a case and is connected to C8 instead of GND. Now all 8MB is available.

## SPIFLASH

[The start guide](https://github.com/wuxx/Colorlight-FPGA-Projects/blob/master/get-start.md) says that the SPI-Flash on i5 modules GD25Q16, is locked. You can free the lock with the new flash command added to LiteX BIOS.

```
litex> flash_write_protect 0
```

should clear all block protestion of GD25B16C flash chip. The modified .bit and .svf of LiteX are [here](https://github.com/kazkojima/colorlight-i5-tips/streams). Hope it hepls.

## SDCARD

I've tested DIGILENT MicroSD PMOD successfully.

![screenshot of LiteX BIOS](https://github.com/kazkojima/colorlight-i5-tips/blob/main/images/uSD-pmod.jpg)

## Ethernet

I can get nothing yet :-(

## Reset switch

...

## Zephyr

...

## linux-on-litex-vexriscv

...
