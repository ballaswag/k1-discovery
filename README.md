# k1-discovery

This repo contains my learnings of the Creality K1 printer.

## Mainboard (x2000e)
### USB Boot/Ingenic Cloner to dump/flash internal storage
See guide here https://github.com/ballaswag/k1-discovery/blob/main/k1-ingenic-cloner-instruction.pdf

K1 default partition offsets. Use these offsets and length limits to read/write to specific partitions in the cloner too.

Please don't use blindly, check your K1 to see your partition layout.
```
GPT Table:                                                                                                                                                                                                                                                            
-------------
uboot(gpt/uboot):    Offset 0x000000000, Length 0x0000100000
ota:                 Offset 0x000100000, Length 0x0000100000
sn_mac:              Offset 0x000200000, Length 0x0000100000
rtos:                Offset 0x000300000, Length 0x0000400000
rtos2:               Offset 0x000700000, Length 0x0000400000
kernel:              Offset 0x000b00000, Length 0x0000800000
kernel2:             Offset 0x001300000, Length 0x0000800000
rootfs:              Offset 0x001b00000, Length 0x0012c00000
rootfs2:             Offset 0x014700000, Length 0x0012c00000
rootfs_data:         Offset 0x027300000, Length 0x0006400000
userdata:            Offset 0x02d700000, Length 0x01a4a00000

Total disk size:0x00000001d2104200, sectors:0x0000000000e90821
```

USB Boot via SPL/uboot
* x2000
  - USB ID: a108:eaef
  - Stage1: load 0xb2401000, exec 0xb2401800
  - Stage2: load 0x80100000, exec 0x80100000

See https://github.com/ballaswag/ingenic-usbboot/tree/main for booting into uboot with USB Mode.

## Buildroot/Toolchain (Distribution)
The board is an Ingenic board and they distribute their variant of Buildroot with prebuilt toolchains. These can be found in ftp.ingenic.com.cn.

ftp://ftp.ingenic.com.cn/DevSupport/X2000/X2000/01_SW/07_kernel4.4.94_x2000-sdk_v8.0-20220125
ftp://ftp.ingenic.com.cn/DevSupport/X2000/X2000/01_SW/01_Buildroot_dl

The selected MIPS functions for the built binaries are as follow:

```
./mips-linux-gnu-readelf -h ld-2.29.so 
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 03 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       3
  Type:                              DYN (Shared object file)
  Machine:                           MIPS R3000
  Version:                           0x1
  Entry point address:               0xde0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          176112 (bytes into file)
  Flags:                             0x70001407, noreorder, pic, cpic, nan2008, o32, mips32r2
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         8
  Size of section headers:           40 (bytes)
  Number of section headers:         26
  Section header string table index: 23

```

Note the Flags field and see GCC doc for detail of each of these flags (https://gcc.gnu.org/onlinedocs/gcc/MIPS-Options.html).

For self compiled dynamically linked executable to work with existing libs in /lib /usr/lib in the K1. Buildroot needs to use the same version of glibc as the ones used on the desire K1 firmware. For 1.2.9.22, the glbic version is 2.26. Newer firmware verion uses different version of glibc.


### Using the toolchain
Will only cover Buildroot and uboot

Extract archive `07_kernel4.4.94_x2000-sdk_v8.0-20220125` 
Extract archive `01_Buildroot_dl -> dl`, move `d1` directory into `buildroot`

```
source build/envsetup.sh
lunch
```

select a platform

Buildroot
```
make -j$(nproc) buildroot
```

To customize packages in buildroot

```
make buildroot-menuconfig
```

Uboot
Still doing some work here. The x2000e on the K1 runs LLDDR2, you'll need to open up the platform you configured via `lunch` and comment out other non-LPDDR2 CONFIG, e.g. in `u-boot/include/configs/halley5.h` look for `CONFIG_DDR_TYPE_LPDDR2`, make sure that's defined, and comment out `#define CONFIG_DDR_TYPE_LPDDR3` and other `CONFIG_DDR_TYPE_*`
```
make -j$(nproc) uboot
```

Outputs (executable, libs, uboot, etc) are in `out/product/<platform>/`.

All the goodies (docs) are here but in Chinese (use google translate if needed):
https://gitee.com/ingenic-dev/ingenic-linux-docs/tree/ingenic-master/zh-cn/X2000/X2000-halley5


### Serial Pins
You can get serial with USB to TTL serial adapter. Pins are pictured below. Baudrate is 115200.

![Serial pinout front](https://github.com/ballaswag/k1-discovery/blob/main/serial_pinout.jpeg)
![Serial pinout back tx,rx,gnd](https://github.com/ballaswag/k1-discovery/blob/main/serial_pinout_back.jpeg)


## Misc Guesswork
The K1 screen is driven by `display-server` (this is likely built using lvgl - https://lvgl.io/). `display-server` drives the display via /dev/fb0. If `Monitor` and `display-server` is killed, you can write random pixels to the screen:
```
cat /dev/urandom > /dev/fb0
```

You can also display any jpeg to the screen using `cmd_jpeg_display`, e.g.
```
cmd_jpeg_display /etc/logo/creality_landscape_rot0_480x800.jpg
```


### KlipperScreen
Building all the dependencies for KlipperScreen, not worth it :), e.g. xserver, gtk3, gobject-introspection, (a opengl lib), mesa, etc. via Buildroot is certainly doable but you will spend a lot time chasing down a working version of various libraries, e.g. librsvg is required by KlipperScreen. Newer librsvg uses rustc which doesn't support MIPS nan2008 (this matters if you want to use shared libs in /lib /usr/lib). The workaround is to pin an earlier version of librsvg in Buildroot that hasn't migrated to rustc. Once that's done you'll still need to workout all the implicit assumptions KlipperScreen makes after the OS, e.g. it expects a debian distrubtion. Might be more worth awhile to adapt KlipperScreen to lvgl or some graphic libraries intended for embedded systems.


## prtouch/prtouch_v2