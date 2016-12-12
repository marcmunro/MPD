# Cross-Compiling MPD for a Raspberry Pi

## Introduction

This documents how I built MPD for a Pi by first building a
cross-compiler toolchain.  It was not a trivial exercise and while I
hope that these notes help, there is no gurantee that this will work for
you.  I synthesised this approach from many sources most of which are
not attributed.

I was building using this machine:

`Linux myhost 3.16.0-4-amd64 #1 SMP Debian 3.16.36-1+deb8u2 (2016-10-19) x86_64 GNU/Linux`

The standard debian approach to cross-compilers did not work for me.  I
do not remember the exact reason but I believe it was a library
compatibility issue.

After a good deal of experimentation I chose to
use [crosstool-NG](http://crosstool-ng.org/).  The tools are very
sophisticated and the community very helpful.

## Process Overview

[This]( https://blog.kitware.com/cross-compiling-for-raspberry-pi/)
gives a far better overview of the process than I can.

# Building the Toolchain

1 Figure out which GCC, library and linux versions you want to build
  for.  You can get the clues you need from your target machine.

  - Use `file` to get information about an executable:

    ```
    [root@runeaudio ~]# file /usr/bin/mpd
    /usr/bin/mpd.orig: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.27, BuildID[sha1]=0486ac2da1f6d4586606205319c78e4babc49f6b, stripped
    [root@runeaudio ~]#
    ```

    This tells us the architecture and the version of linux for which
    the executable was built.

  - execute the libc shared library

    ```
    [root@runeaudio ~]# /usr/lib/libc-2.18.so
    GNU C Library (GNU libc) stable release version 2.18, by Roland McGrath et al.
    Copyright (C) 2013 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.
    There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
    PARTICULAR PURPOSE.
    Compiled by GNU CC version 4.8.2 20131219 (prerelease).
    Compiled on a Linux 3.12.4 system on 2014-01-19.
    Available extensions:
    	crypt add-on version 2.1 by Michael Glad and others
    	GNU Libidn by Simon Josefsson
    	Native POSIX Threads Library by Ulrich Drepper et al
    	BIND-8.2.3-T5B
    libc ABIs: UNIQUE
    For bug reporting instructions, please see:
    <https://github.com/archlinuxarm/PKGBUILDs/issues>.
    [root@runeaudio ~]# 

    ```
    
    This tells us the library and gcc versions.  Note that for the
    runeaudio Pi distribution we need to select hardware floating
    point.  My final, successful, configuration 
    
2 Build the toolchain

  Note that this is likely to be an iterative process as the environment
  in which you are trying to build your toolchain is likely to have
  versions of make, automake, etc that are not compatible with the
  compiler version you are building.

  ```
  build-dir:$ wget http://crosstool-ng.org/download/crosstool-ng/crosstool-ng-1.22.0.tar.bz2
  build-dir:$ tar xjvf crosstool-ng-1.22.0.tar.bz2 
  build-dir:$ cd crosstool-ng
  crosstool-ng:$ ./configure --prefix=`pwd`/../crosstool
  crosstool-ng:$ make install
  crosstool-ng:$ PATH=$PATH:`pwd`/../crosstool/bin
  crosstool-ng:$ ct-ng menuconfig
  
  ```

  This generates a `.config` file, which is used by the build step that
  follows.  My config file can be found in `rune/crosstools.config`.  You
  could choose to use that rather than building your own.  Proceding...

  `crosstool-ng:$ ct-ng build`

  This will take some time.  If it fails, you should look at the log
  files.  As well as the master log file, there are specific log files
  for each stage of the build process.  Typically errors will occur at
  the `configure` stage of building a specific tool, so the interesting
  logfile will be called `config.log` and will be somewhere below your
  build directory.  Use find to find the appropriate file and then
  examine it carefully:

  `build-dir:$ find . -name config.log`

  This will probably find a large number of files.  The one you want is
  in a directory named for whatever was being built when it failed.

  Usually the last `error` string in the file will give you some useful
  clues (sometimes the preceding error is even more helpful).

  In my case, I had failures because:

  - the hosts automake was more up to date than the version ct-ng wanted;
  - the hosts make was more up to date.

  For automake, I removed automake and build and installed an old
  version from sources.

  For make, I hacked the `configure` script (in the same directory as
  `config.log` to allow a newer version of make.

3 Clone Needed files from target

  In order to cross-compile, the local environment must have copies of
  all headers, libraries and package information that are needed.  These
  must come from the target machine.

  Libraries and headers will be copied into the cross-compiler tools
  sysroot directory:

  ```
  MPD:$ export SYSROOT=${HOME}/x-tools/armv6-rpi-linux-gnueabihf/armv6-rpi-linux-gnueabihf/sysroot
  MPD:$ cd ${SYSROOT}/usr
  usr:$ scp -r root@runeaudio.local:/usr/lib .
  usr:$ scp -r root@runeaudio.local:/usr/include .
  ```

  We also need package configuration information:

  ```
  MPD:$ mkdir pkgconfig
  MPD:$ scp -r root@runeaudio.local:/usr/lib/pkgconfig pkgconfig
  ```

  We will later need to tell pkg-config where to look by setting
  `PKG_CONFIG_PATH`.

  Finally, we need to modify pkg-config so that it returns paths with
  the `${SYSROOT}` prefix.  I created a simple shell script
  `mypkg-config` to do this, and told `configure` to use this script by
  setting `PKG_CONFIG`.

4 Build MPD

  After cloning MPD from [github](http://github.org) I set up the
  necessary environment for building:

  ```
   MPD:$ PATH=$PATH:${HOME}/x-tools/armv6-rpi-linux-gnueabihf/bin
   MPD:$ export CC=armv6-rpi-linux-gnueabihf-gcc
   MPD:$ export CXX=armv6-rpi-linux-gnueabihf-g++
   MPD:$ export CCLD=armv6-rpi-linux-gnueabihf-ld
   MPD:$ export CXXLD=armv6-rpi-linux-gnueabihf-ld
   MPD:$ export RANLIB=armv6-rpi-linux-gnueabihf-ranlib
   MPD:$ export SYSROOT=$HOME/x-tools/armv6-rpi-linux-gnueabihf/armv6-rpi-linux-gnueabihf/sysroot/usr
   MPD:$ export LDFLAGS="-L${SYSROOT}/lib"
   export PKG_CONFIG_PATH=${HOME}/proj/pi/pkgconfig
   export PKG_CONFIG="${HOME}/proj/pi/MPD/mypkg-config"

   ```

   And then ran `autogen.sh` and `configure`:

   ```
   MPD:$ cd <MPD dir>
   MPD:$ ./autogen.sh --host=armv6-rpi-linux-gnueabihf
   MPD:$ ./configure --build amd64 --host armv6-rpi-linux-gnueabihf \
   --prefix=/usr --sysconfdir=/etc --disable-ao --disable-pulse \
   --disable-shout --disable-sidplay --disable-modplug \
   --disable-soundcloud --disable-wavpack --disable-opus \
   --disable-lame-encoder --disable-ipv6 --disable-recorder-output \
   --disable-iso9660 --disable-zzip --disable-wildmidi  --disable-oss \
   --disable-fluidsynth --disable-bzip2 --disable-gme --disable-mpg123 \
   --enable-mad=yes  -enable-libmpdclient --enable-jack \
   --disable-smbclient --disable-nfs --disable-cdio-paranoia \
   --disable-adplug --disable-roar --disable-openal \
   --disable-icu --disable-soxr --disable-twolame-encoder \
   --disable-shine-encoder --disable-ffmpeg \
   --with-zeroconf=avahi \
   --with-systemdsystemunitdir=/usr/lib/systemd/system --with-boost=no \
   LIBMPDCLIENT_CFLAGS=" " LIBMPDCLIENT_LIBS="-lmpdclient" \
   LDFLAGS=$LDFLAGS
   ```

   If this succeeds, there is a good chance that all will go well:

   `MPD:$ make`

   One problem I had was with lib boost.  The headers for boost mostly
   create inline functions rather than requiring a library.  This meant
   that I could safely install libboost on the host and simply copy the
   headers into `${SYSROOT}/include`:

   ```
   MPD:$ cd ${SYSROOT}/usr/include
   include:$ ln -s /usr/include/boost .
   ```

   The mpd executable is created as `src/mpd`.  You can examine it with
   the `file` command to ensure that it looks good.  Once you have
   copied it to your target machine, use `ldd` to check it.  If `ldd`
   does not complain, then all should be well.

5 strip the executable

  The final step before installing your executable for real is to strip
  it.  This removes a whole bunch of symbols which you aren't going to
  need unless you want to try interactive debugging on the pi:

  `MPD:$ armv6-rpi-linux-gnueabihf-strip src/mpd`

