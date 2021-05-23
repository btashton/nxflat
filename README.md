# NXFLAT Toolchain

*NOTE:* This work is forked off from the NuttX buildroot repository at https://bitbucket.org/nuttx/buildroot to build as standalone applications.  This project is primarily the work of Gregory Nutt.

NXFLAT is a customized and simplified version of binary format implemented a few years ago called XFLAT With the NXFLAT binary format you will be able to do the following:

 * Place separately linked programs in a file system, and
 * Execute those programs by dynamically linking them to the base NuttX code.

This allows you to extend the NuttX base code after it has been written into FLASH. One motivation for implementing NXFLAT is support clean CGI under an HTTPD server.

This feature is especially attractive when combined with the NuttX ROMFS support: ROMFS allows you to execute programs in place (XIP) in flash without copying anything other than the .data section to RAM. In fact, the initial NXFLAT release only worked on ROMFS. Later extensions also support execution NXFLAT binaries from an SRAM copy as well.

This NuttX feature includes:

 * A dynamic loader that is built into the NuttX core (See GIT).
 * Minor changes to RTOS to support position independent code, and
 * A linker to bind ELF binaries to produce the NXFLAT binary format (See GIT).

## Background

NXFLAT is derived from XFLAT. XFLAT is a toolchain add that provides full shared library and XIP executable support for processors that have no Memory Management Unit (MMU). NXFLAT is greatly simplified for the deeply embedded environment targeted by NuttX:

 * NXFLAT does not support shared libraries, because
 * NXFLAT does not support exportation of symbol values from a module

Rather, the NXFLAT module only imports symbol values. In the NXFLAT model, the (PIC) NXFLAT module resides in a FLASH file system and when it is loaded at run time, it is dynamically linked only to the (non-PIC) base NuttX code: The base NuttX exports a symbol table; the NXFLAT module imports those symbol value to dynamically bind the module to the base code. 

## Programs

### mknxflat

Is used to build a thunk file. See below for usage.

```
    Usage: mknxflat [options] <bfd-filename>

    Where options are one or more of the following.  Note
    that a space is always required between the option and
    any following arguments.

      -d Use dynamic symbol table. [symtab]
      -f <cmd-filename>
          Take next commands from <cmd-filename> [cmd-line]
      -o <out-filename>
         Output to  [stdout]
      -v Verbose output [no output]
      -w Import weakly declared functions, i.e., weakly
         declared functions are expected to be provided at
         load-time [not imported]
```

### ldnxflat

ldnxflat is use to link your object files along with the thunk file generated by mknxflat to produce the NXFLAT binary module. See below for usage.

```
    Usage: ldnxflat [options] <bfd-filename>

    Where options are one or more of the following.  Note
    that a space is always required between the option and
    any following arguments.

      -d Use dynamic symbol table [Default: symtab]
      -e <entry-point>
         Entry point to module [Default: _start]
      -o <out-filename>
         Output to <out-filename> [Default: <bfd-filename>.nxf]
      -s <stack-size>
         Set stack size to <stack-size> [Default: 4096]
      -v Verbose output. If -v is applied twice, additional
         debug output is enabled [Default: no verbose output].
```

### readnxflat

readnxflat is used for dumping information from nxflat files.

```
   Usage: ./readnxflat [options] <flat-filename>

   Where options are one or more of the following:

     -h Dump the XFLAT file header     [not dumped]
     -r Dump relocation entries        [not dumped]
     -i Dump the imported symbol table [not dumped]
     -x Dump xFLT loader pathname      [not dumped]
     -c Disassemble the text section   [not dumped]
     -d Dump data section (hex)        [not dumped]
     -a Dump all of the above          [not dumped]
     -b Assume big-endian byteorder    [little endian]
     -v Output verbose debug info      [no output]
```

## Building

This project depends on libbfd which is bundled as part of GNU Binutils and the version used must include arm-elf target support.

These tools can be built for `arm` or `thumb2` by setting the ARCH variable, by default thumb2 will be used.

```
❯ ARCH=arm make
❯ ARCH=thumb2 make

```

### Ubuntu

Ubuntu includes a multiarch build of binutils that can be used with this.  You must install the `binutils-multiarch-dev` package.
Note the version as is must be included when compiling.

```

❯ apt install binutils-multiarch-dev       
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  binutils-multiarch-dev
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 1524 B of archives.
After this operation, 15.4 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu focal-updates/main amd64 binutils-multiarch-dev amd64 2.34-6ubuntu1.1 [1524 B]
Fetched 1524 B in 0s (3337 B/s)                  
debconf: delaying package configuration, since apt-utils is not installed
Selecting previously unselected package binutils-multiarch-dev.
(Reading database ... 11884 files and directories currently installed.)
Preparing to unpack .../binutils-multiarch-dev_2.34-6ubuntu1.1_amd64.deb ...
Unpacking binutils-multiarch-dev (2.34-6ubuntu1.1) ...
Setting up binutils-multiarch-dev (2.34-6ubuntu1.1) ...

❯ LIBBFD=lbfd-2.34-multiarch make
cc -c -Wall -I. -DPACKAGE=1 -DPACKAGE_VERSION=1 -o ldnxflat.o ldnxflat.c
cc  ldnxflat.o -o ldnxflat -lbfd-2.34-multiarch
cc -c -Wall -I. -DPACKAGE=1 -DPACKAGE_VERSION=1 -o mknxflat.o mknxflat.c
cc  mknxflat.o -o mknxflat -lbfd-2.34-multiarch
cc -c -Wall -I. -DPACKAGE=1 -DPACKAGE_VERSION=1 -o readnxflat.o readnxflat.c
make -C arch CC="cc"
make[1]: Entering directory '/src2/thumb2'
cc -c -Wall -o disthumb2.o disthumb2.c
ar rcs libarch.a disthumb2.o
make[1]: Leaving directory '/src2/thumb2'
cc  -L arch -o readnxflat readnxflat.o -lbfd-2.34-multiarch -larch
```

Cleaning:

```
❯ make clean
make -C arch clean
make[1]: Entering directory '/home/bashton/nuttx/nxflat/arm'
rm -f *.o *.a *~ .*.swp core
make[1]: Leaving directory '/home/bashton/nuttx/nxflat/arm'
```

### Non-Ubuntu

Many distributions do not provide a multiarch version of libbfd so it must be build from binutils source.  This will install only the BFD library.

```
❯ wget https://ftp.gnu.org/gnu/binutils/binutils-2.36.tar.xz
❯ tar -xz binutils-2.36.tar.xz && cd binutils-2.36
❯ ./configure --prefix=$(HOME)/.local --target=arm-elf  --enable-install-libbfd --disable-plugins --enable-shared=yes
❯ make all-bfd
❯ make install-bfd
....
----------------------------------------------------------------------
Libraries have been installed in:
   /home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/lib

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the `-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the `LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the `LD_RUN_PATH' environment variable
     during linking
   - use the `-Wl,-rpath -Wl,LIBDIR' linker flag
   - have your system administrator add LIBDIR to `/etc/ld.so.conf'

❯ ls i$HOME/.local/x86_64-pc-linux-gnu/arm-elf/lib
libbfd-2.36.so  libbfd.a  libbfd.la  libbfd.so

```

You can then choose to dynamically or statically link.  Shown here is dynamically linking, make sure you do one of the recommendations shown above to handle the library search path.

```
❯ LIBBFD_PREFIX=$HOME/.local/x86_64-pc-linux-gnu/arm-elf
❯ CFLAGS="-I$LIBBFD_PREFIX/include" LDFLAGS="-L$LIBBFD_PREFIX/lib" make
cc -c -I/home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/include -Wall -I. -DPACKAGE=1 -DPACKAGE_VERSION=1 -o ldnxflat.o ldnxflat.c
cc -L/home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/lib ldnxflat.o -o ldnxflat -lbfd
cc -c -I/home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/include -Wall -I. -DPACKAGE=1 -DPACKAGE_VERSION=1 -o mknxflat.o mknxflat.c
cc -L/home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/lib mknxflat.o -o mknxflat -lbfd
cc -c -I/home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/include -Wall -I. -DPACKAGE=1 -DPACKAGE_VERSION=1 -o readnxflat.o readnxflat.c
make -C arch CC="cc"
make[1]: Entering directory '/home/bashton/nuttx/nxflat/thumb2'
cc -c -I/home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/include -Wall -I. -DPACKAGE=1 -DPACKAGE_VERSION=1 -Wall -o disthumb2.o disthumb2.c
ar rcs libarch.a disthumb2.o
make[1]: Leaving directory '/home/bashton/nuttx/nxflat/thumb2'
cc -L/home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/lib -L arch -o readnxflat readnxflat.o -lbfd -larch


❯ LD_LIBRARY_PATH=$LIBBFD_PREFIX/lib ldd ldnxflat
	linux-vdso.so.1 (0x00007ffffd7fd000)
	libbfd-2.36.so => /home/bashton/.local/x86_64-pc-linux-gnu/arm-elf/lib/libbfd-2.36.so (0x00007f2cc478a000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f2cc4595000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f2cc4896000)
```

For statically linking you will need to make sure you have static versions of other libraries including glibc and libz.


