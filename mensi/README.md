
Playing around with the Rigol MSO5074
Date: 2021-01-10 | permalink | Tags: mso5000

I'm going to do this on a fresh Ubuntu 20.10 install - mainly because most tools are just an apt-get install away.

But first, we have to find a firmware to look at. Rigol seems to have various websites targeted at different markets. But it seems they all carry the same version as the most recent: 00.01.03.00.01:

    rigol.eu
    rigolna.com

Both contain a file called DS5000Update.GEL with an MD5 checksum of c85c5f4a64a8c9d435b589835225d527.
What is a GEL file?

I've had a Rigol MSO5074 oscilloscope for a while now, and while unlocking all options has already been achieved over at the eevblog forums, I've always wanted to poke around a bit myself.

A good thing to try first is trying file to get an idea what format it could be:

$ file DS5000Update.GEL
DS5000Update.GEL: POSIX tar archive (GNU)

A tar file is of course nice to start with. Let's unpack it:

$ tar -xvf DS5000Update.GEL
fw4linux.sh
fw4uboot.sh
logo.hex.gz
zynq.bit.gz
system.img.gz
app.img.gz

We get a couple of shell scripts as well as some compressed data files. We can decompress the data files with gunzip *.gz. The shell scripts however seem to be compressed or encrypted:

$ hd fw4linux.sh 
00000000  1e 36 66 da f3 a5 41 d4  de f1 95 ab 09 0f 52 1c  |.6f...A.......R.|
00000010  07 99 0f 2e 35 0f b8 85  6b 95 6e e3 b2 fb 0a aa  |....5...k.n.....|
[...]

So let's ignore those for now. logo.hex and zynq.bit don't sound too interesting - they are most likely some image and the Zynq FPGA bitstream.
app.img

file is our friend once more:

$ file app.img 
app.img: UBI image, version 1

Googling for "UBI image" will likely be a bit misleading since RedHat's announcement for their container image ranks higher than the result we're interested in: UBIFS on Wikipedia. From there, we learn that UBI and UBIFS are a filesystem and something that looks similar to LVM specifically built for flash devices - doing nice things like wear leveling and bad block management.

You might be tempted to mount this image on a loop-back device - but that doesn't work since UBI runs on top of MTD, which is a different kind of device than a block device.

There's however a Python implementation called "ubi_reader" we can use. Let's set up a python venv and install it:

$ sudo apt-get install python3-venv python3-dev liblzo2-dev build-essential
[.. installing stuff ..]
$ python3 -m venv ../venv
$ . ../venv/bin/activate
$ pip install ubi_reader python-lzo

We can get the parameters used for this UBI image:

$ ubireader_utils_info app.img

Volume app
    alignment   -a 1
    default_compr   -x lzo
    fanout      -f 8
    image_seq   -Q 2016671535
    key_hash    -k r5
    leb_size    -e 126976
    log_lebs    -l 5
    max_bud_bytes   -j 8388608
    max_leb_cnt -c 825
    min_io_size -m 2048
    name        -N app
    orph_lebs   -p 1
    peb_size    -p 131072
    sub_page_size   -s 2048
    version     -x 1
    vid_hdr_offset  -O 2048
    vol_id      -n 0

    #ubinize.ini#
    [app]
    vol_type=dynamic
    vol_flags=autoresize
    vol_id=0
    vol_name=app
    vol_alignment=1
    vol_size=98660352

These should come in handy in case we want to use nandsim to mount the UBI image. We can also extract all files:

$ ubireader_extract_files app.img
Extracting files to: ubifs-root/2016671535/app
$ ls ubifs-root/2016671535/app/
appEntry  cups  default  drivers  K160M_TOP.bit  mail  Qt5.5  resource  shell  tools  webcontrol
$ ls ubifs-root/2016671535/app/shell/
format_disk.sh  load_setup.sh  mount_user_space.sh  print_page.sh  send_mail.sh  start.sh  update.sh  wifi.sh

We could probably already have guessed that is is an ARM Linux environment since we saw a Zynq bitstream earlier and Zynq contains ARM cores - but we can confirm by taking a look at the appEntry binary:

$ file ubifs-root/2016671535/app/appEntry 
ubifs-root/2016671535/app/appEntry: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.3, for GNU/Linux 2.6.16, stripped

update.sh also looks interesting as it mentions fw4linux.sh we saw earlier. It seems to pass it through /rigol/tools/cfger -d where the -d could stand for "decrypt".

This whole thing is only application code though - let's go look at the system as well.
system.img

This time around we don't get an easy life with file:

$ file system.img 
system.img: data

But we have a great tool to get further: binwalk. Since we already have a Python venv, let's just clone from Github and install:

$ git clone https://github.com/ReFirmLabs/binwalk.git
$ cd binwalk
$ python setup.py install

And run it against system.img:

$ binwalk system.img

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Flattened device tree, size: 14216248 bytes, version: 17
248           0xF8            Linux kernel ARM boot executable zImage (little-endian)
16611         0x40E3          gzip compressed data, maximum compression, from Unix, last modified: 1970-01-01 00:00:00 (null date)
3303420       0x3267FC        Flattened device tree, size: 9597 bytes, version: 17
3313212       0x328E3C        gzip compressed data, has original file name: "rootfs.img", from Unix, last modified: 2019-01-22 08:41:09
12627549      0xC0AE5D        MySQL MISAM index file Version 6

Running it with -e extracts anything it can extract into _system.img.extracted where we find rootfs.img. Falling back to file:

$ file rootfs.img 
rootfs.img: Linux rev 1.0 ext2 filesystem data, UUID=dba05baa-0271-4f62-92a1-3a6f75eecf53

This time around, we can use a loop-back mount:

$ sudo losetup -f _system.img.extracted/rootfs.img
$ losetup -l
NAME        SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                                                           DIO LOG-SEC
[...]
/dev/loop10         0      0         0  0 /path/to/rootfs.img   0     512
$ mkdir rootfs_initrd
$ sudo mount /dev/loop10 rootfs_initrd/

And in there, we have a Linux root filesystem.
Combining system.img and app.img

Based on what we saw in update.sh and the fact that there is an empty directory called rigol in the root filesystem, it is likely that the contents of app.img are mounted under /rigol. We can emulate this with a bind mount:

$ sudo mount --bind ubifs-root/2016671535/app rootfs_initrd/rigol

And with this, we have what we would expect to see on the scope.
Decrypting fw4linux.sh

Now that we have the user-space, can we somehow use it to decrypt the fw4linux script? It would be great if we could just run cfger but that is an ARM binary and my VM is amd64. We can try QEMU to get around that:

$ sudo apt-get install qemu-user
$ cd rootfs_initrd/
$ chmod +x rigol/tools/cfger
$ LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger
cfger: loadlocale.c:130: _nl_intern_locale_data: Assertion `cnt < (sizeof (_nl_value_type_LC_TIME) / sizeof (_nl_value_type_LC_TIME[0]))' failed.
qemu: uncaught target signal 6 (Aborted) - core dumped
Aborted (core dumped)

With LD_LIBRARY_PATH, we're telling the linker where to look for shared libraries - which it should look for in our extracted rootfs instead of on the host system. Unfortunately the binary crashed, but it looks like it happened due to locale-related code. Let's just set the local to C and try again:

$ LC_ALL=C LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger
/tmp/env.bin not exist
$ touch /tmp/env.bin
$ LC_ALL=C LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger
qemu: uncaught target signal 11 (Segmentation fault) - core dumped
Segmentation fault (core dumped)
$ dd if=/dev/zero of=/tmp/env.bin bs=100 count=1
1+0 records in
1+0 records out
100 bytes copied, 0.000193575 s, 517 kB/s
$ LC_ALL=C LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger
crc error

So with the locale set, the binary runs but complains about a missing /tmp/env.bin file. Just creating an empty one leads to a segmentation fault, hinting at an unchecked read in the file. An all-zeros file works, but leads to a CRC error.

Since the CRC of 0xff is 0xff we can play around a bit:

$ printf '\xff\x00\x00\x00\xff' > /tmp/env.bin 
$ LC_ALL=C LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger
crc error
$ printf '\x00\x00\x00\xff\xff' > /tmp/env.bin 
$ LC_ALL=C LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger
"\uFFFD"
"UTF-8"
"\u0019"

We got some output, yay! Let's try the -d flag from update.sh:

$ LC_ALL=C LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger -d ../fw4linux.sh /tmp/fw4linux.sh
$ head /tmp/fw4linux.sh 
#!/bin/sh

model=MSO5074
softver=00.01.03.00.01
builddate="2020-03-30 15:56:36
[...]

Nice! We can also try to see if the binary has help with -h and it turns out it does! And there is even a -e flag we can use to encrypt our own fw4linux.sh.
Running SSH

Further examination of the startup (etc/inittab and etc/init.d/rcS) tells us that there is an sshd- but it's commented out and thus will not start by default. Additionally, we don't know the root password.

But since we can create our own GEL file with an fw4linux.sh script, we can fix that:

#!/bin/sh
mkdir -p "/root/.ssh/"
echo "ssh-rsa YOURRSAPUBKEY your-key-comment" >> "/root/.ssh/authorized_keys"
/etc/init.d/S50sshd restart
exit 1

I saved this as /tmp/runssh.sh, so we can encrypt it and pack it up as a tar file:

$ LC_ALL=C LD_LIBRARY_PATH=./lib:./rigol/Qt5.5/lib qemu-arm -L ./ rigol/tools/cfger -e /tmp/runssh.sh /tmp/fw4linux.sh
$ cd /tmp
$ tar -cf DS5000Update.GEL fw4linux.sh

We can put the resulting GEL file onto a USB stick and give it a try on the scope. My scope is running an older firmware, but it's unlikely Rigol has changed a lot, so there's a good chance it will work.

Insert the USB stick, press the "Utility"-button -> "System" -> "Help" -> "Local upgrade" -> "OK". It will say the upgrade failed, but trying to connect via SSH should succeed. Login with root and your private key.

Check out part 2 where we poke around the running system a bit.
