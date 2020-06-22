---
title: "GSoC Week 3"
date: 2020-06-20T10:53:56+05:30
draft: false
---

As gcc needed glibc version 2.29, I started emerging glibc   

It said ```invalid: REDEPEND: USE flag 'kernel_linux' referenced in conditional 'kernel_linux?' is not in use```

So, I added kernel_linux under IUSE_IMPLICIT variable to /etc/portage/make.profile/make.defaults

Similarly as above, ```Missing USE flag 'amd64'``` So, added amd64 use flag under IUSE_IMPLICIT variable to /etc/portage/make.profile/make.defaults

```bash
IUSE_IMPLICIT="abi_x86_64 prefix prefix-guest kernel_linux amd64"
```  

Then, it complained about saying ```Your /etc/nsswitch.conf is out of date```. So, I had to update it. [My nsswitch.conf](http://ix.io/2oaK)

As glibc needed sys-devel/bison, I commented it from make.provided and emerged
```bash
sudo emerge -a sys-devel/bison
```

Also Installed linux-headers as glibc's dependencies 
```bash
sudo emerge -a sys-kernel/linux-headers
```

I had to add sys-devel/gcc to package.provided (as it was dependencies for glibc). Continued with emerging, ```ERROR : in function iconv_open': (.text+0x55a): undefined reference to __stack_chk_guard'```

Referring to one of stackoverflow post, I added -fno-stack-protector to CFLAGS in  make.conf and used -ssp during emerging.

Then, ```Segmentation fault (core dumped) LC_ALL=C ./ld-*.so --library-path . ${x} > /dev/null```

When i asked about this in the IRC channel, they said that I managed to build a glibc that doesn't work and it failed in sanity check. Also, they commented on ```__stack_chk_guard``` error saying that it was dangerous to add -fno-stack-protector flag and it also tends to breaks things

I thought, instead of adding use flags manually(caused a lot of problems), I would change the profile to default/linux/amd64/17.1/desktop and add gcc, glibc to packages.provided which overcame the problem of circular dependencies.

But, again falls on world update due to circular dependencies of perl package. So, I decided to build binaries for perl.

I read about [crossdev](https://gitweb.gentoo.org/proj/crossdev.git/tree/README) again and started building toolchain for crOS.
- This would build the glibc greater than v2.29 and perl binaries also

In order to build toolchain for x86_64-cros-linux-gnu, I needed a [custom repository](https://wiki.gentoo.org/wiki/Custom_repository#Crossdev) which i had setup previously and also updated my system. I built the toolchain for crOS and tested by building busy box package with static keyword.
```bash
sudo crossdev -t x86_64-cros-linux-gnu
sudo USE=static x86_64-cros-linux-gnu-emerge -av1 busybox
```

Then, I built gcc-9.3 for crOS (Was not to cross compile gcc-10.1.0-r1)
```bash
sudo x86_64-cros-linux-gnu-emerge -av1 =sys-devel/gcc-9.3.0
```

Before building perl binaries, I wanted make sure of toolchain. So, I zipped the directory /usr/x86_64-cros-linux-gnu and moved that to crOS. 

I symlinked the gcc's and binutils's binaries to toolchain's usr/bin and added to PATH variable. Then, I added  toolchain's usr/lib to LD_LIBRARY_PATH variable. Also, symlinked header files from toolchain's  sys-include to /usr/include. Finally, "Hello world" program successfully executed on crOS

Now, I tried to emerge dev-lang/perl but had to add ncurse to toolchain's package.use

But got stuck with error 
```error
The following REQUIRED_USE flag constraints are unsatisfied:
  python_single_target_python2_7 python_single_target_python3_8 python_single_target_python3_8
The above constraints are a subset of the following complete expression:
    exactly-one-of ( python_single_target_python2_7 python_single_target_python3_7 python_single_target_python3_8) 
```

While installing python-3.8.3, I had to add PYTHON_TARGETS PYTHON_TARGETS="python3_8" to toolchain's make.conf and add PYTHON_TARGETS to toolchain's make.profile/make.default line USE_EXPAND

One of the python dependencies, python-exec failed to install. So, I Added to toolchains's profile/package.provided (removed after the python installation completed and emerged again)

Also, I had to emerge a unstable package, setup_tools. So, I added it to system's system's package.use with ~amd64 use flag

Then, Error ```Couldn't satisfy ebuild dev-python/certifi```. I added it to package.provided as it was firefox's CA Bundle. Python successfully emerged

After all this, perl package failed to emerge by giving an error that it couldn't cross-compile perl. I asked on IRC Channel and they asked me to find the right version of perl to emerge which is compatible  perl-cross
```bash
x86_64-cros-linux-gnu-emerge -av1 =dev-lang/perl-5.30.1
```

Finally, I had build all the packages that was stuck in circular dependencies
```
sys-devel/gcc-9.3.0
sys-libs/glibc-2.30-r8
dev-lang/perl-5.30.3
sys-libs/db-5.3.28-r2
sys-devel/autoconf-2.69-r4
```
I zipped the directory /usr/x86_64-cros-linux-gnu using, copied to crOS and unzipped it
```
tar -czf x86_64-cros-linux-gnu.tar /usr/x86_64-cros-linux-gnu # On gentoo system
sudo tar -xzf x86_64-cros-linux-gnu.tar # On crOS root directory
```

But it failed as it had less space on its root partition i.e 2GB

While I was googling around on how to extend the partition, I found a script called [chromeos-resize](https://github.com/ethanmad/chromeos-resize)

I downloaded the script and executed it.
```bash
curl https://raw.githubusercontent.com/ethanmad/chromeos-resize/master/cros-resize.sh
sudo chmod 755 cros-resize.sh
sudo ./cros-resize.sh
```

Script failed as it needed developer mode to be enabled. As i don't own a chromebook, I couldn't. So commented those lines and re-ran it again.

I followed as it instructed me. After it was done, it rebooted and repaired itself. I was supposed to run it again with the same values.

It successfully completed but the when i executed ```df -h```, it was of same size. I rechecked with ```fdisk -l``` and it extended to rootfs C which was minimal-size partition for future third kernel.

I went through the script and learnt how to extend a crOS partition

- As crOS repaired itself, it wipes out all the data on the device

According to phyical partition of crOS ([Layout](https://www.chromium.org/chromium-os/chromiumos-design-docs/disk-format/layout.png?attredirects=0)), I took 6gb from state partition and added to rootfs A. It is easier as they are beside each other

Live boot a OS, list the partition and extend it
```bash
cgpt show /dev/sda
cgpt add -i 1 -b <startblock> -s <size> /dev/sda # Moved the starting point by 6gb and reduced that 6gb from size also
cgpt add -i 3 -b <startblock> -s <size> /dev/sda # kept starting point same but added + 6gb for size
```
Then, check consistency and resize the file system
```bash
e2fsck -f /dev/sda3 
resize2fs /dev/sda3
```

Reboot and repeat the above 2 steps again 

Finally, I was able to extend the root partition to 8GB

### Summary
As gcc needed glibc version 2.29, I started emerging it with its dependencies and also by adding required use flags. But it built a glibc that doesn't work and failed sanity checks. So, I changed the profile to default/linux/amd64/17.1/desktop and added gcc, glibc to packages.provided which overcame the problem of circular dependencies.

Then, It needed perl package, So I had to cross compile it using crossdev. At last, I zipped the toolchain directory, copied to crOS and unzipped it. But it failed as it had less space on its root partition. I referred a script called chromeos-resize.sh and learnt things that was required. It successfully extended the root partition.
