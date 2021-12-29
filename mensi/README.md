
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

PART    2
--------------------------------------------------------------------------------

Playing around with the Rigol MSO5074 - Part 2
Date: 2021-01-27 | permalink | Tags: mso5000

After getting SSH access in the last post. We can now explore the running system a bit more.
Running processes

Looking at the running processes with ps ax yields only a few:

    /rigol/appEntry -run
    rpcbind
    /rigol/cups/sbin/cupsd -C /rigol/cups/etc/cups/cupsd.conf
    /rigol/webcontrol/sbin/lighttpd -f /rigol/webcontrol/config/lighttpd.conf
    ... and of course sshd and our login shell, but that's less interesting at this point

So it looks like we have the main app (appEntry), CUPS for printing and lighthttpd for the web interface. Slightly more interesting is that we have rpcbind, which could hint at NFS being available to us. Let's give it a try.
Using NFS to exchange files

Still using the Ubuntu machine from the last post, we can install NFS with apt-get install nfs-server and add a share in /etc/exports:

/home/youruser/nfsserver 192.168.1.0/24(rw,sync,all_squash,anonuid=1000,anongid=1000)

and reload with exportfs -ra. The share is available to the whole subnet (change it if you use different IP addresses) and squashes all users to the Ubuntu user (replace "youruser" and the UID/GID to match your setup).

On the scope, we can then mount this:

<root@rigol>mkdir /media/nfs
<root@rigol>mount -t nfs 192.168.1.1:/home/youruser/nfsserver /media/nfs
<root@rigol>cd /media/nfs/
<root@rigol>ls
hello  world

and we can see the hello and world files I put in the shared directory.
Mounts

Looking at /proc/mounts, we can see what else is mounted:

rootfs / rootfs rw 0 0
/dev/root / ext2 rw,relatime,errors=continue 0 0
devtmpfs /dev devtmpfs rw,relatime,size=218708k,nr_inodes=54677,mode=755 0 0
none /proc proc rw,relatime 0 0
none /sys sysfs rw,relatime 0 0
none /tmp tmpfs rw,relatime,size=102400k 0 0
devpts /dev/pts devpts rw,relatime,mode=600 0 0
/dev/ubi6_0 /rigol ubifs rw,relatime 0 0
/dev/ubi1_0 /rigol/data ubifs rw,sync,relatime 0 0
/dev/ubi12_0 /user ubifs rw,sync,relatime 0 0

In addition to /rigol we already discovered last time, there seem to be two additional UBIFS mounts:

    /user: /user/data appears to be the location that the scope UI calls C:\
    /rigol/data: Seems to contain calibration data and license keys

Various other things

<root@rigol>cat /proc/cpuinfo
processor       : 0
model name      : ARMv7 Processor rev 0 (v7l)
Features        : swp half thumb fastmult vfp edsp neon vfpv3 tls vfpd32
CPU implementer : 0x41
CPU architecture: 7
CPU variant     : 0x3
CPU part        : 0xc09
CPU revision    : 0

processor       : 1
model name      : ARMv7 Processor rev 0 (v7l)
Features        : swp half thumb fastmult vfp edsp neon vfpv3 tls vfpd32
CPU implementer : 0x41
CPU architecture: 7
CPU variant     : 0x3
CPU part        : 0xc09
CPU revision    : 0

Hardware        : Xilinx Zynq Platform
Revision        : 0000
Serial          : 0000000000000000

<root@rigol>cat /proc/meminfo
MemTotal:         448236 kB
MemFree:          288680 kB
[...]

<root@rigol>cat /proc/cmdline
console=ttyPS0,115200 no_console_suspend, root=/dev/ram rw

<root@rigol>uname -an
Linux (none) 3.12.0-xilinx #48 SMP PREEMPT Wed Dec 12 15:26:15 CST 2018 armv7l GNU/Linux

<root@rigol>lsmod
usbtmc 16092 0 - Live 0xbf026000
usbtmc_dev 12637 1 - Live 0xbf01d000
libcomposite 38365 1 usbtmc_dev, Live 0xbf00c000
tmp421 3786 0 - Live 0xbf008000
devIRQ 2618 2 - Live 0xbf004000 (O)
axi 2540 1 - Live 0xbf000000 (O)

Looks like we know everything we need to try and compile/run our own programs!

LAST
-------------------------------------------------


From poking around earlier, we can make an educated guess that the graphical output works via frame buffer as there is a device node /dev/fb0. fbset can tell us a bit more about its format:

<root@rigol>fbset

mode "1024x600-0"
        # D: 0.000 MHz, H: 0.000 kHz, V: 0.000 Hz
        geometry 1024 600 1024 600 16
        timings 0 0 0 0 0 0 0
        accel false
        rgba 5/11,6/5,5/0,0/0
endmode

So we have 1024 * 600 pixels formatted as 16bit RGB values with 5, 6 and 5 bits respectively dedicated for the colors.

Let's try dumping the frame buffer (cp /dev/fb0 /tmp/screenshot) and transfer it over to our more beefy Ubuntu VM. There, we can use ffmpeg to convert it to PNG:

$ ffmpeg -vcodec rawvideo -f rawvideo -pix_fmt rgb565 -s 1024x600 \
         -i screenshot -f image2 -vcodec png screenshot.png

Looking at the resulting file, we however just see the RIGOL logo with the loading progress bar at 100%, so there seems to be more to it. The easiest way to figure out how the frame buffer works would be to look at the source - fortunately, Olliver Schinagl managed to get kernel sources from Rigol and put them on GitLab.
The frame buffer driver

Looking at .config we find that the only enabled frame buffer driver is CONFIG_FB_XILINX. This driver can be found in /drivers/video/xilinxfb.c. After the normal header includes, this driver also seems to just straight up include /drivers/video/dpu.c.

An interesting place to start is probably the ioctls this driver supports as there might be some special ones Rigol put in. And we can indeed find some custom commands in xilinx_fb_ioctl:

    DPU_SET_LAYER_RST: Resets the driver
    DPU_SET_LAYER_ID: Changes the layer ID
    DPU_GET_LAYER_ID: Returns the current layer ID
    DPU_SET_TRACE_MASK: ???
    DPU_SET_TRACE_COLOR: ???
    DPU_SET_TRACE_DATA: ???
    DPU_SET_LAYER_ALPHA: ???
    DPU_SET_LAYER_OPEN: ???
    DPU_SET_LAYER_CLOSE: ???
    DPU_SET_WAVE_XY: Set the X and Y coordinate of the wave layer
    DPU_SET_HDMI: ???
    DPU_SCR_PRINT: Instruct the hardware to do a printscreen

The command numbers start with 0x0F000000 for DPU_SET_LAYER_ID and then just increment in the order defined in the enum in /drivers/video/dpu.h (note that the order is slightly different than above / in the ioctl code!).
Layers

So it seems we have a stack of layers. drvDPUInit provides a good idea of what these layers are. The layer numbers are defined via an enum:

enum
{
    DPU_Layer_Back,  // layer 0
    DPU_Layer_Wave,  // layer 1
    DPU_Layer_Eye,   // layer 2
    DPU_Layer_Fore,  // layer 3
    DPU_Layer_Print, // layer 4
    DPU_Layer_Comp,  // layer 5
    DPU_Layer_All,
    DPU_Layer_Logo = DPU_Layer_All
};

DPU_Layer_All does not seem to be a layer itself but is used as the number of layers (of which there are 6).

How do we get those layers? The memory that is mapped in xilinx_fb_mmap is chosen based on drvdata->opAddr which indexes to the layer currently chosen via the DPU_SET_LAYER_ID ioctl. We can thus use said ioctl to select which layer we want to mmap.
Dumping layers

To dump layers, we have to switch the layer via ioctl, mmap the file and then just write out the bytes. We can again use ffmpeg to convert the raw pixel array to a PNG file. You can find my rust implementation for the dumper on Github. It also contains code to directly output PNG, so you can even skip ffmpeg.

The background (0) and foreground (3) layers contain about what we'd expect, namely the backing grid and the UI elements respectively. The wave layer has a different color format and dimension - the code indicates 1000 * 480 in R8G8B8 format. The reality seems to look different though if we look at an excerpt from hexdump:

000e6b90  cc cc cc 00 00 45 45 00  cc cc cc 00 00 45 45 00  |.....EE......EE.|
000e6ba0  00 6a 6a 00 cc cc cc 00  cc cc cc 00 00 53 53 00  |.jj..........SS.|

We know that the transparency color is 0xCCCCCC so we have 4 bytes with the 4th always being zero. Additionally, the trace is yellowish, but the first byte is zero for opaque pixels. So this looks more like a BGR format with 1 byte each + a zero-byte per pixel.
Taking screenshots

From the code, we can also deduct that there is a hardware-assisted print screen function. The implementation can be found in the printSCR function. It seems to set a register to request the operation and then wait until completion is signalled via another register or the timeout expires.

We can trigger this by using command 0x0F00000C and passing a pointer to an integer - this will contain 0 to signal success and 1 for failure.
Putting it all together

With the separate layers, we can do custom stackups. The following image is the back layer with the wave layer on top as two images. Since we have transparency preserved in the PNG files, this works as intended and shows the background grid under the traces:
Using Rust on the Rigol MSO5074
Date: 2021-03-06 | permalink | Tags: mso5000

In the last post, we got NFS set up - that should make it quite easy to experiment with cross-compiling our own code for the scope and run it directly off of an NFS share.

I'm using the same Ubuntu 20.10 VM as always for this. The goal this time is to see if we can write some code in rust and get it running. To follow along, you should have rustup installed. You can find instructions here.
Creating a hello world binary

First, we're going to need the ARM target for Rust as well as the ARM cross compiler:

$ sudo apt install gcc-arm-linux-gnueabihf
$ rustup target install armv7-unknown-linux-musleabihf

Note that the target is using musl (a small C standard library) instead of glibc. The reason for this is mainly that on the scope, we have glibc 2.4 and we'd need a toolchain built against the same version in order to support dynamic linking. It therefore seems easier to use musl and just link everything statically for now.

Let's create a new rust project:

$ cargo new hello
     Created binary (application) `hello` package
$ cd hello

Feel free to use your favorite editor to change the hello world message into whatever you like in src/main.rs. We can tell cargo what linker to use and build a binary for our target:

$ export CARGO_TARGET_ARMV7_UNKNOWN_LINUX_MUSLEABIHF_LINKER=/usr/bin/arm-linux-gnueabihf-gcc
$ cargo build --target=armv7-unknown-linux-musleabihf --release

Unfortunately this results in a 3.6 MB executable for me, which seems a bit excessive. We can squash this in half: In Cargo.toml, add a [profile.release] section with the option lto = true and build again.
A slight detour: Let's make it smaller

1.5 MB still seems excessive, so let's see if we can't make this even smaller. The nightly rust releases seem to have some more features:

    Install nightly: rustup default nightly
    Install the rust source for nightly: rustup component add rust-src --toolchain nightly
    You might have to install the target again for nightly: rustup target install armv7-unknown-linux-musleabihf

Applying some more tricks, we get this Cargo.toml:

cargo-features = ["strip"]

[package]
name = "hello"
version = "0.1.0"
edition = "2018"

[dependencies]

[profile.release]
lto = true
strip = "symbols"
codegen-units = 1
opt-level = "z"

for a binary size of about 260 KB. Still large for just printing "hello world" but much better.
Running it on the actual scope

You will need SSH access to your scope to test it - refer to the fist post in this series on how to get that.

You can either use the NFS share from last time or copy our binary over to the scope:

$ scp target/armv7-unknown-linux-musleabihf/release/hello root@IPOFYOURSCOPE:/tmp/

and login via SSH and run it:

<root@rigol>/tmp/hello
Hello, world!

Playing around with the Rigol MSO5074 - Part 2
Date: 2021-01-27 | permalink | Tags: mso5000

After getting SSH access in the last post. We can now explore the running system a bit more.
Running processes

Looking at the running processes with ps ax yields only a few:

    /rigol/appEntry -run
    rpcbind
    /rigol/cups/sbin/cupsd -C /rigol/cups/etc/cups/cupsd.conf
    /rigol/webcontrol/sbin/lighttpd -f /rigol/webcontrol/config/lighttpd.conf
    ... and of course sshd and our login shell, but that's less interesting at this point

So it looks like we have the main app (appEntry), CUPS for printing and lighthttpd for the web interface. Slightly more interesting is that we have rpcbind, which could hint at NFS being available to us. Let's give it a try.
Using NFS to exchange files

Still using the Ubuntu machine from the last post, we can install NFS with apt-get install nfs-server and add a share in /etc/exports:

/home/youruser/nfsserver 192.168.1.0/24(rw,sync,all_squash,anonuid=1000,anongid=1000)

and reload with exportfs -ra. The share is available to the whole subnet (change it if you use different IP addresses) and squashes all users to the Ubuntu user (replace "youruser" and the UID/GID to match your setup).

On the scope, we can then mount this:

<root@rigol>mkdir /media/nfs
<root@rigol>mount -t nfs 192.168.1.1:/home/youruser/nfsserver /media/nfs
<root@rigol>cd /media/nfs/
<root@rigol>ls
hello  world

and we can see the hello and world files I put in the shared directory.
Mounts

Looking at /proc/mounts, we can see what else is mounted:

rootfs / rootfs rw 0 0
/dev/root / ext2 rw,relatime,errors=continue 0 0
devtmpfs /dev devtmpfs rw,relatime,size=218708k,nr_inodes=54677,mode=755 0 0
none /proc proc rw,relatime 0 0
none /sys sysfs rw,relatime 0 0
none /tmp tmpfs rw,relatime,size=102400k 0 0
devpts /dev/pts devpts rw,relatime,mode=600 0 0
/dev/ubi6_0 /rigol ubifs rw,relatime 0 0
/dev/ubi1_0 /rigol/data ubifs rw,sync,relatime 0 0
/dev/ubi12_0 /user ubifs rw,sync,relatime 0 0

In addition to /rigol we already discovered last time, there seem to be two additional UBIFS mounts:

    /user: /user/data appears to be the location that the scope UI calls C:\
    /rigol/data: Seems to contain calibration data and license keys

Various other things

<root@rigol>cat /proc/cpuinfo
processor       : 0
model name      : ARMv7 Processor rev 0 (v7l)
Features        : swp half thumb fastmult vfp edsp neon vfpv3 tls vfpd32
CPU implementer : 0x41
CPU architecture: 7
CPU variant     : 0x3
CPU part        : 0xc09
CPU revision    : 0

processor       : 1
model name      : ARMv7 Processor rev 0 (v7l)
Features        : swp half thumb fastmult vfp edsp neon vfpv3 tls vfpd32
CPU implementer : 0x41
CPU architecture: 7
CPU variant     : 0x3
CPU part        : 0xc09
CPU revision    : 0

Hardware        : Xilinx Zynq Platform
Revision        : 0000
Serial          : 0000000000000000

<root@rigol>cat /proc/meminfo
MemTotal:         448236 kB
MemFree:          288680 kB
[...]

<root@rigol>cat /proc/cmdline
console=ttyPS0,115200 no_console_suspend, root=/dev/ram rw

<root@rigol>uname -an
Linux (none) 3.12.0-xilinx #48 SMP PREEMPT Wed Dec 12 15:26:15 CST 2018 armv7l GNU/Linux

<root@rigol>lsmod
usbtmc 16092 0 - Live 0xbf026000
usbtmc_dev 12637 1 - Live 0xbf01d000
libcomposite 38365 1 usbtmc_dev, Live 0xbf00c000
tmp421 3786 0 - Live 0xbf008000
devIRQ 2618 2 - Live 0xbf004000 (O)
axi 2540 1 - Live 0xbf000000 (O)

Looks like we know everything we need to try and compile/run our own programs!
Bare Metal C Programming the Blue Pill - Part 2
Date: 2021-01-22 | permalink | Tags: stm32

In part 1, we came up with a minimal program and a linker script to compile a binary we could load onto the Blue Pill. We used OpenOCD to program the binary into the chip's embedded flash memory. OpenOCD can however do much more than what we did so far.
Programming Flash

To change things up a little bit, we're going to use an ST-LINK V2 adapter this time. Compared to the the Sipeed JTAG adapter used in part 1, it has less features; there is no embedded serial port and it can't do JTAG.

On the other hand, the ST-LINK provides power for our Blue Pill and uses SWD to talk to the chip - the necessary SWDIO and SWCLK pins are conveniently accessible on the end of the Blue Pill. For a smooth experience, you also want to connect the RST pin of the ST-LINK with the reset pin on the Blue Pill (next to the USB connector, marked "R" or "B2").

Since we're using a different adapter (or "interface" in OpenOCD terminology), we need a new config file. OpenOCD ships with one for the ST-LINK, so our config can be as simple as this (stlink_bluepill.cfg):

source [find interface/stlink-v2.cfg]
transport select hla_swd
source [find target/stm32f1x.cfg]

We're just using the shipped interface config, the SWD transport and the STM32F1x target (remember, the Blue Pill has an STM32F103 chip). We can again program our ELF binary from part 1:

openocd -f stlink_bluepill.cfg -c "program main.elf verify reset exit"

Debugging

OCD stands for "On-Chip Debugger", so programming could be seen more as a side business than the core purpose of OpenOCD. ARM cores contain a debug port (DP) that can be access via JTAG or SWD. Through it, we can read and manipulate the CPU's state and memory.

Let's start OpenOCD again, this time without programming anything:

$ openocd -f stlink_bluepill.cfg
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v29 API v2 SWIM v7 VID 0x0483 PID 0x3748
Info : using stlink api v2
Info : Target voltage: 3.145419
Info : stm32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints

In this mode, OpenOCD will be listening on ports 3333 and 4444. Let's try telnet on port 3333:

$ telnet localhost 4444
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Open On-Chip Debugger
>

We get a prompt, where we can for example halt the CPU. If we do this, our blinking LED should stop:

> halt
target halted due to debug-request, current mode: Thread
xPSR: 0x21000000 pc: 0x08000132 msp: 0x20004ff0

You can resume the CPU (and the blinking) with resume. To disconnect, simply use exit.
GDB

On port 3333, OpenOCD runs a GDB server. We can run GDB and connect to it with target remote :3333 like this:

$ arm-none-eabi-gdb main.elf
GNU gdb (7.10-1+9) 7.10
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=arm-linux-gnueabihf --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from main.elf...done.
(gdb) target remote :3333
Remote debugging using :3333
wait () at main.c:22
22        for (unsigned int i = 0; i < 2000000; ++i) __asm__ volatile ("nop");
(gdb)

We compiled the binary with -ggdb, so GDB was able to load debug symbols and after connecting to the target can directly tell us what code the CPU was executing. Since we're waiting most of the time, it's highly likely that we see the for loop in the wait() function doing NOPs.

Let's reset the CPU and single-step through our code (you can just press enter to repeat the previous command):

(gdb) monitor reset halt
target halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x08000140 msp: 0x20005000
(gdb) s
25      void main() {
(gdb)
36          *((volatile unsigned int *)0x40011010) = (1U << 13);
(gdb)
39          *((volatile unsigned int *)0x40011010) = (1U << 29);
(gdb)
27        *((volatile unsigned int *)0x40021018) |= (1 << 4);
(gdb)
30        *((volatile unsigned int *)0x40011004) = ((0x44444444 // The reset value
(gdb)
36          *((volatile unsigned int *)0x40011010) = (1U << 13);
(gdb)

The LED should now be turned on - but what follows is a boring 2 million NOP loops. If we want to go straight to where we turn the LED off, we can use a break point. Resetting the LED is on line 39, so:

(gdb) break main.c:39
Breakpoint 1 at 0x8000146: file main.c, line 39.
(gdb) c
Continuing.

Unfortunately, at least with my GCC and GDB versions, this did not work.
Interlude: Why is the breakpoint not working?

We can disassemble the code of our main function:

(gdb) disass
Dump of assembler code for function main:
0x08000140 <+0>:     movs    r1, #16
0x08000142 <+2>:     push    {r4, r5, r6, lr}
0x08000144 <+4>:     movs    r6, #128        ; 0x80
0x08000146 <+6>:     movs    r5, #128        ; 0x80
0x08000148 <+8>:     ldr     r2, [pc, #32]   ; (0x800016c <main+44>)
0x0800014a <+10>:    ldr     r3, [r2, #0]
0x0800014c <+12>:    orrs    r3, r1
0x0800014e <+14>:    str     r3, [r2, #0]
0x08000150 <+16>:    ldr     r2, [pc, #28]   ; (0x8000170 <main+48>)
0x08000152 <+18>:    ldr     r3, [pc, #32]   ; (0x8000174 <main+52>)
0x08000154 <+20>:    str     r2, [r3, #0]
0x08000156 <+22>:    lsls    r6, r6, #6
0x08000158 <+24>:    lsls    r5, r5, #22
0x0800015a <+26>:    ldr     r4, [pc, #28]   ; (0x8000178 <main+56>)
0x0800015c <+28>:    str     r6, [r4, #0]
0x0800015e <+30>:    bl      0x8000130 <wait>
=> 0x08000162 <+34>:    str     r5, [r4, #0]
0x08000164 <+36>:    bl      0x8000130 <wait>
0x08000168 <+40>:    b.n     0x800015a <main+26>
0x0800016a <+42>:    nop                     ; (mov r8, r8)
0x0800016c <+44>:    asrs    r0, r3, #32
0x0800016e <+46>:    ands    r2, r0
0x08000170 <+48>:    add     r4, r8
0x08000172 <+50>:    add     r4, r6
0x08000174 <+52>:    asrs    r4, r0, #32
0x08000176 <+54>:    ands    r1, r0
0x08000178 <+56>:    asrs    r0, r2, #32
0x0800017a <+58>:    ands    r1, r0
End of assembler dump.

Looking at the code, we wanted to break between the two call to main - this would be at address 0x08000162. However, the breakpoint was set at 0x8000146. If we stare at the code a bit, we can see that GCC optimized it for speed quite a bit.

Instead of computing the bitshift in each loop iteration, it's only doing str r5, [r4, #0], so just storing the value from register r5. And if we look at the instruction where GDB set the breakpoint, movs r5, #128, this is actually the first step in computing the shifted (1U << 29) value. The second part is at 0x08000158 where the already set 0x80 is shifted by 22.

So it looks like GDB set the breakpoint on the first instruction belonging to the line of code we wanted to break on - unfortunately this is outside the loop and we won't hit it.

We can set a breakpoint on the correct address manually:

(gdb) break *0x08000162
Breakpoint 7 at 0x8000162: file main.c, line 39.
(gdb) c
Continuing.

Breakpoint 2, main () at main.c:39
39          *((volatile unsigned int *)0x40011010) = (1U << 29);

Looking at CPU state

After hitting the (fixed) breakpoint from above, we can take a look at the CPU registers:

(gdb) info registers
r0             0x4a5dead2       1247668946
r1             0x10     16
r2             0x44344444       1144276036
r3             0x0      0
r4             0x40011010       1073811472
r5             0x20000000       536870912
r6             0x2000   8192
r7             0xf23d2b64       -230872220
r8             0xff3ffffe       -12582914
r9             0xefbf9ddd       -272654883
r10            0x2ad17842       718370882
r11            0x63dac8c9       1675282633
r12            0xf9ffeffb       -100667397
sp             0x20004ff0       0x20004ff0
lr             0x8000163        134218083
pc             0x8000162        0x8000162 <main+34>
xPSR           0x61000000       1627389952

Most interesting are r5 and r6, where we figured out our shifted values for turning on/off the LED are. pc is the program counter - which unsurprisingly points to the instruction we set the breakpoint at.

sp is the stack pointer. If we compare it to where we started, we didn't get too far:

(gdb) print &_end_of_ram
$1 = (unsigned long *) 0x20005000

But since our program is not that big, that's to be expected. We could also print memory, but again, due to the simplicity of our minimal program, everything ends up in registers anyway.
Playing around with the Rigol MSO5074
Date: 2021-01-10 | permalink | Tags: mso5000

I've had a Rigol MSO5074 oscilloscope for a while now, and while unlocking all options has already been achieved over at the eevblog forums, I've always wanted to poke around a bit myself.

I'm going to do this on a fresh Ubuntu 20.10 install - mainly because most tools are just an apt-get install away.

But first, we have to find a firmware to look at. Rigol seems to have various websites targeted at different markets. But it seems they all carry the same version as the most recent: 00.01.03.00.01:

    rigol.eu
    rigolna.com

Both contain a file called DS5000Update.GEL with an MD5 checksum of c85c5f4a64a8c9d435b589835225d527.
What is a GEL file?

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
Bare Metal C Programming the Blue Pill - Part 1
Date: 2020-11-21 | permalink | Tags: stm32

These days, there are many easy to use environments for microcontroller programming. You have the popular Arduino platform but also the IDEs and tools provided by the different chip manufacturers as well as several open source efforts.

While these are certainly great for getting your project going, they often gloss over the actual "bare metal" details of the chip you work with. So let's explore a bit and peek under the hood.
Hardware

For this post, I'm going to use a little STM32 dev board that is generally advertised as the "Blue Pill" and can be found on eBay, Aliexpress or your other favorite source of cheap gadgets from China.

The board is based around an STM32F103C8T6 chip. It contains a 32bit ARM Cortex M3 microcontroller and lots of peripherals. You can typically get the boards for less than 2$ per piece. While the identifier "STM32F103C8T6" might look scary at first, it's actually quite simple: It's a part from STMicroelectronics, has a 32bit CPU, is part of the F1 family, specifically the F103 model. C8T6 encodes things like which chip package is used and how much flash memory is included (in this case 64kB).

Since the chip features an ARM 32bit CPU, we will need a compiler that can target tthat architecture. I'll use GCC - so the compiler we want is gcc-arm-none-eabi (targeting ARM, no OS, embedded ABI). If you're using Ubuntu, Debian or Raspbian, you can install the compiler via apt-get install gcc-arm-none-eabi.
How do you program a micro controller?

Contrary to your typical $FAVORITE_OS program, you do not (generally) have any libraries or environment around your code. Instead, the instructions your code is compiled into just run directly, one after each other.

Another important aspect is that you do not have virtual memory - so you do not have a private large memory space you can use as you like and you talk to a kernel when you want to do anything with hardware. Instead, you are directly accessing the physical memory (RAM).

The physical memory you can use for your program however does not start at address 0. It is actually a relatively small chunk among other things at are made to look like they are part of memory. So what else do we have there? Let's have a look at the datasheet. In section 4, we can see how memory is organized. The interesting bits are:
First Byte 	Last Byte 	Length 	Contents
0x0000 0000 	Depends 	2kB or 64kB 	Either Flash memory or hardwired system memory depending on boot pins
0x0800 0000 	0x0800 FFFF 	64kB 	Flash memory (the C8T6 variant has 64kB. The datasheet shows 128kB)
0x1FFF F000 	0x1FFF F7FF 	2kB 	System memory with the hardwired bootloader from ST
0x2000 0000 	0x2000 4FFF 	20kB 	SRAM - our main RAM
0x4000 0000 	0x4000 33FF 	141kB 	Peripherals (your ADCs, GPIOs, SPIs, UARTs, ... )

So we can't just write our data / variables to arbitrary memory locations - they might not be backed by RAM. Or even worse, they might be backed by some memory mapped hardware and we could get interesting side effects.

We now know that address 0 actually either maps to the embedded flash memory or the hardwired system memory. Which one we get is determined by the boot pins (BOOT0 and BOOT1). If we select flash, whatever we put in there appears starting at address 0.
The Linker

While the compiler takes care of translating our C code to machine instructions (for ARM in our case), the linker is responsible for resolving references between functions, globals or in more generic terms "symbols". It also lays out the final binary image.

We can control this process with linker scripts. Here's a minimal example (linker.ld):

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}

SECTIONS
{
    .text : {
        /* Start at 0 for the code in flash */
        . = 0;

        /* At the very beginning, we need the interrupt vectors.
        * We need to mark this KEEP to make sure it doesn't get garbage collected.
        * The syntax *(.foo) means take the .foo sections from all files. */
        KEEP(*(.interrupt_vectors))

        /* The whole interrupt table is 304 bytes long. Advance to that position in case
        * the supplied table was shorter. */
        . = 304;

        /* And here comes the rest of the code, ie. all symbols starting with .text */
        *(.text*)
    } > FLASH = 0xFF /* Put this in the flash memory region, default to 0xFF */

    _end_of_ram = ORIGIN(RAM) + LENGTH(RAM); /* Define a symbol for the end of ram */
}

MEMORY defines the location an size of flash and ram. We start laying out the text section at position 0, first putting the interrupt_vectors section, ensuring it is at least 304 bytes long and then we collect all .text sections with the code. A special symbol _end_of_ram is defined to point at the byte just after the last usable ram byte - we will need this to initialize the stack pointer.

The flash memory's default state is 0xFF and not 0x00, so we'll set this as the fill value. Also note that there would be two options for the flash ORIGIN address: We could use either 0x0000 0000 or 0x0800 0000, but remember that it only appears at at 0x0000 0000 if we set the boot configuration as such. So if we chose to boot differently, it will not be there.
Writing some code

With the linker script prepared, let's write some code. The hello world of micro controllers is to blink an LED. The Blue Pill has one on board which is connected to pin PC13.

So far, we've only looked at the datasheet. In order to write code that accesses peripherals like the GPIO module to blink the LED, we'll need more information. We can find the necessary details in the document RM0008: The reference manual for the STM32F1xx parts.

Don't get scared by the over 1100 pages of text. We will reference specific sections to figure out what we're after. A good place to start is section 3.1: System architecture. Here we can see that there are 1 CPU core and 2 DMA engines accessing flash, ram and peripherals through a bus matrix. The peripherals sit behind the AHB on two peripheral buses: APB1 and APB2. GPIOC which we need for controlling our LED at PC13 is on APB2.
Clock gating

One thing that might not immediately be obvious is that the clock for these peripherals can be turned off to conserve power. And they do default to off. So we first need to turn the clock for port C on. Where we do that is described in section 7.3.7: APB2 peripheral clock enable register (RCC_APB2ENR)

So we want to set bit 4 of this register to 1 to enable port C, but where is this register in memory? We're only being told that it is at address 0x18 within the memory for RCC configuration. So let's go back to section 3.3 to consult the memory map. There, "Reset and clock control RCC" is listed at 0x4002 1000. Adding 0x18 we get 0x4002 1018.
GPIO config

So what else do we need to do? Section 9 covers GPIO. Looks like have quite some options. Let's just do a simple push-pull output. Table 20 and 21 tell us that we need CNF=00 and MODE=11 for a max. 50 MHz speed push-pull.

Section 9.2.2 documents the register we need. The bits for pin 13 are 20-23. Note the reset value 0x4444 4444: We'll have to leave the other bits with those values to avoid changing the config of any other pins. The section specifies an address offset of 0x04, but what is the base for this? Our trusty memory map in section 3.3 points us at 0x4001 1000 for GPIO Port C, so we get 0x4001 1004.
Toggling the GPIO pin

Finally, we need to toggle the pin between high and low to blink the LED. We can use the trick described in section 9.1.2 to atomically set/reset only the pin we're interested in. The BSRR register for that purpose is at offset 0x10 as described in section 9.2.5. We need to set bit 13 to make PC13 high and bit 29 to make it low respectively.
Putting it all together

With this information, we can put together our main.c:

// This is the symbol defined in the linker script.
extern unsigned long _end_of_ram;

// We'll implement main a bit further down but we need the symbol for
// the initial program counter / start address.
void main();

// This is our interrupt vector table that goes in the dedicated section.
__attribute__ ((section(".interrupt_vectors"), used))
void (* const interrupt_vectors[])(void) = {
    // The first 32bit value is actually the initial stack pointer.
    // We have to cast it to a function pointer since the rest of the array
    // would all be function pointers to interrupt handlers.
    (void (*)(void))((unsigned long)&_end_of_ram),

    // The second 32bit value is the initial program counter aka. the
    // function you want to run first.
    main
};

void wait() {
    // Do some NOPs for a while to pass some time.
    for (unsigned int i = 0; i < 2000000; ++i) __asm__ volatile ("nop");
}

void main() {
    // Enable port C clock gate.
    *((volatile unsigned int *)0x40021018) |= (1 << 4);

    // Configure GPIO C pin 13 as output.
    *((volatile unsigned int *)0x40011004) = ((0x44444444 // The reset value
        & ~(0xfU << 20))  // Clear out the bits for pin 13
        |  (0x3U << 20)); // Set both MODE bits, leave CNF at 0

    while (1) {
        // Set the output bit.
        *((volatile unsigned int *)0x40011010) = (1U << 13);
        wait();
        // Reset it again.
        *((volatile unsigned int *)0x40011010) = (1U << 29);
        wait();
    }
}

This program essentially only consists of two parts: - The interrupt vector table that specifies the initial stack pointer and our entrypoint - A super tiny main function that sets up the GPIO pin and toggles it in a loop

If you're wondering how I came up with 2000000 loop iterations: By default, the CPU runs off a 8MHz internal oscillator. Each loop iteration takes a few instructions, so if we count to 2 million, it's more or less going to take about a second.

We can compile:

arm-none-eabi-gcc -ggdb -Wall -Os -mthumb -nostdlib -o main.o -c main.c

link:

arm-none-eabi-gcc -ggdb -Os -Wl,--gc-sections -mthumb -mcpu=cortex-m3 -Tlinker.ld -o main.elf main.o

and flash (using the JTAG adapter described in this post):

openocd -f sipeed.cfg -c "program main.elf verify reset exit"

and you should see the onboard LED start blinking.

Check out part 2 to do more with OpenOCD.
Using the Sipeed JTAG Debugger with a Blue Pill
Date: 2020-11-15 | permalink | Tags: stm32 jtag

I recently got a Sipeed USB-JTAG/TTL debugger - mainly because I was looking for a JTAG adapter and it looked compact and somewhat sturdy.

You can find these in the usual places (Seedstudio, Aliexpress, etc.) for around 10$. The documentation seems a bit sparse or rather non-existent though. There is a schematic on Github that is as far as I can tell reasonably accurate.

Plugging it in reveals:

$ lsusb
Bus 001 Device 006: ID 0403:6010 Future Technology Devices International, Ltd FT2232C Dual USB-UART/FIFO IC

which matches the schematic linked above as well.
So what's this FTDI chip?

The FT2232C (datasheet) the debugger uses seems like an interesting chip: It's a USB to serial bridge with 2 channels. The first channel has some extra features: You can configure it in "Multi-Protocol Synchronous Serial Engine (MPSSE)"-mode (application note). With that, you can implement serial protocols with clocking like SPI or JTAG.

Additionally, there are GPIO pins that can be used together with the fixed data in/out and clock pins. The Sipeed debugger connects GPIOL1 aka ADBUS5 to the pin labelled "RST". You could in principle use this pin for anything - but it's of course ideal for controlling the reset signal of your chip.
OpenOCD

OpenOCD supports several different FTDI-based adapters with the ftdi interface type but does not define any presets for the Sipeed adapter in particular. It is however configurable and we can make it work for us. To talk to a "Blue Pill" (STM32 F1 family MCU) dev board, we can use a config like this:

# Tell OpenOCD to use the ftdi interface
interface ftdi

# Find the device based on the USB vendor/product ID
ftdi_vid_pid 0x0403 0x6010

# FT2232C IO bits per schematic:
# 0: TCK, Output
# 1: TDO, Output
# 2: TDO, Input
# 3: TMS, Output
# 4: Not connected
# 5: RST, Output
#
# The first 16bit value is the initial IO state. Just make TMS and RST high
# The second 16bit value is data direction for each pin, 1 = Output
ftdi_layout_init 0x0028 0x2b

# We'll use RST for the system reset, not the JTAG reset. Thus, disable nTRST
# by setting data and enable mask to 0. If you want to use
# the RST pin for nTRST instead, switch this and the nSRST line.
ftdi_layout_signal nTRST -data 0x0 -oe 0x0

# RST is on bit 5, so the mask is 0x20. The pin is directly connected, so
# we don't have an output-enable pin -> set the same mask
ftdi_layout_signal nSRST -data 0x0020 -oe 0x0020

# JTAG mode
transport select jtag

# And finally include the config for the target we're using. You could
# also skip this here and instead pass it via commandline arguments to openocd
source [find target/stm32f1x.cfg]

Finally launching OpenOCD: openocd -f sipeed.cfg. Or to flash an ELF binary: openocd -f sipeed.cfg -c "program thebinarytoflash.elf verify reset exit"

In parallel, you can still use the second serial port as usual.
