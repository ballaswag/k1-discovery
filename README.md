# k1-discovery

This repo contains my learnings of the Creality K1 printer.

## Mainboard (x2000e)
### USB Boot/Ingenic Cloner to dump/flash internal storage
See guide here https://github.com/ballaswag/k1-discovery/blob/main/k1-ingenic-cloner-instruction.pdf

USB Boot via SPL/uboot
* x2000
  - USB ID: a108:eaef
  - Stage1: load 0xb2401000, exec 0xb2401800
  - Stage2: load 0x80100000, exec 0x80100000


### Serial Pins
You can get serial with USB to TTL serial adapt. Pins are pictured below. Baudrate is 115200.

![Serial pinout front](https://github.com/ballaswag/k1-discovery/blob/main/serial_pinout.jpeg)
![Serial pinout back tx,rx,gnd](https://github.com/ballaswag/k1-discovery/blob/main/serial_pinout_back.jpeg)


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