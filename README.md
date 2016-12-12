# Marc's Hack on Music Player Daemon

This is a hack specifically for users of the Hifiberry DAC+ which has a
rather lovely hardware volume control.  Out of the box behaviour with
this volume control is that levels are crazy low until around 45, and
then get very loud very quickly as the level is raised.

This hack is to make the volume control more progressive (and usable).
I have not folded it back into the original MPD out of pure laziness.
My hack really is a hack, and is very specific to this piece of
hardware.  If you'd like to get it into the real, upstream, MPD by all
means go for it.  The changed files are:

 - `src/decoder/plugins/FfmpegDecoderPlugin.cxx`
 - `src/mixer/plugins/AlsaMixerPlugin.cxx`

I have implemented this hack on a [runeaudio
device](http://www.runeaudio.com/) , and I include a cross-compiled
executable for an original model Pi running runeaudio 0.3  The hacked
executable is in: 

- `rune/mpd`

The hard part of the whole exercise was building a cross-compiler and
getting that working.  If you'd like to try to do the same, please view
[the Cross-Compiling For Rune notes](rune/XCOMPILE.md).

# Installation on the Rune

1) Download a copy of the file: `rune/mpd`

2) Copy it to your rune device

   `myhost$ scp mpd root@runeaudio.local:.`

3) Backup the orginal version of the file

   This is in case you break anything irreperably.  If you do, you
   should be able to recover by copying the original file back.

   ```
    myhost$ ssh root@runeaudio.local
    ===============  RuneOS distribution  ===============
      ____                      _             _ _       
     |  _ \ _   _ _ __   ___   / \  _   _  __| (_) ___  
     | |_) | | | | '_ \ / _ \ / _ \| | | |/ _` | |/ _ \ 
     |  _ <| |_| | | | |  __// ___ \ |_| | (_| | | (_) |
     |_| \_\\__,_|_| |_|\___/_/   \_\__,_|\__,_|_|\___/ 
                                                        
    ================  www.runeaudio.com  ================
    RuneOs: 0.3-beta (build 20141029)
    RuneUI: 1.3
    Hw-env: RaspberryPi

    Last login: Sun Dec 11 12:11:38 2016 from 192.168.2.102
    [root@runeaudio ~]# 
    [root@runeaudio ~]# cp /usr/bin/mpd /usr/bin/mpd.orig
    ```

4) Install the new version of the file

    `[root@runeaudio ~]# cp mpd /usr/bin/mpd`

5) Restart the machine

   The easiest thing is to simply power down and power back up again.
   You will probably need to re-scan your music sources as this
   version of mpd is a little newer than the runeaudio version and seems
   to have an incompatible database version.  It worked for me.

# The Original Stuff

http://www.musicpd.org

A daemon for playing music of various formats.  Music is played through the 
server's audio device.  The daemon stores info about all available music, 
and this info can be easily searched and retrieved.  Player control, info
retrieval, and playlist management can all be managed remotely.

For basic installation information see the INSTALL file.

# Users

- [Manual](http://www.musicpd.org/doc/user/)
- [Forum](http://forum.musicpd.org/)
- [IRC](irc://chat.freenode.net/#mpd)
- [Bug tracker](http://bugs.musicpd.org/)

# Developers

- [Protocol specification](http://www.musicpd.org/doc/protocol/)
- [Developer manual](http://www.musicpd.org/doc/developer/)

# Legal

MPD is released under the
[GNU General Public License version 2](https://www.gnu.org/licenses/gpl-2.0.txt),
which is distributed in the COPYING file.
