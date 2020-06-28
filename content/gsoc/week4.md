---
title: "GSoC Week 4"
date: 2020-06-27T20:26:02+05:30
draft: false
---

After expanding the root partition, I moved the toolchain built using crossdev to crOS. I extracted it in /usr/local/x86_64-cros-linux-gnu, force symlinked binutils and gcc's binaries  to toolchain's /usr/bin, added PATH and LD_LIBRARY_PATH bash variable to .bashrc.I also symlinked toolchain's sys-include to /usr/include
```bash
sudo tar -xzf cross-x86_64-cros-linux-gnu.tar -C /usr/local/x86_64-cros-linux-gnu/
sudo ln -sf /usr/local/x86_64-cros-linux-gnu/usr/x86_64-cros-linux-gnu/gcc-bin/9.3.0/*  /usr/local/x86_64-cros-linux-gnu/usr/bin/
sudo ln -sf /usr/local/x86_64-cros-linux-gnu/usr/x86_64-cros-linux-gnu/binutils-bin/2.33.1/*  /usr/local/x86_64-cros-linux-gnu/usr/bin/
sudo export PATH=/usr/local/x86_64-cros-linux-gnu/usr/bin:$PATH
sudo export LD_LIBRARY_PATH=/usr/local/x86_64-cros-linux-gnu/usr/lib:$LD_LIBRARY_PATH
sudo ln -s /usr/local/x86_64-cros-linux-gnu/sys-include/* /usr/include/
```

- If you get inconsistency in STATE_PARTITION or ```Error: wrong fs type, bad option, bad superblock on /dev/sda1, missing codepage or helper program, or other error```, Do follow the previous steps  again after reboot for the  check

When i tried to execute a hello world program, It gave me a error ```Error: ld: cannot find /usr/lib64/libc_nonshared.a```. So, symlinked the file and it successufully executed hello.c. I managed to end up the same with perl
```bash
sudo ln -s /usr/local/x86_64-cros-linux-gnu/usr/lib/libc_nonshared.a /usr/lib64/libc_nonshared.a
```

But the catch is that, The crOS core packages are compiled with older version of gcc and glibc which is holding me back to compile new packages using portage. If i force to do so, then the core packages with breaks badly.

Then, I came across this guide to upgrade old system gentoo ([link](https://wiki.gentoo.org/wiki/Upgrading_Gentoo)). I thought, it is better to use stage 3 tarball to install basic packages

So, I followed the guide above. I created a intermediate build chroot location (/mnt/build) and extracted a recent stage3 archive into it. Next, I setup the network access and created a mount point inside this chroot environment, on which I then bind-mount the old environment. 
```bash
mkdir -p /mnt/build
tar -xf /path/to/stage3-somearch-somedate.tar.bz2 -C /mnt/build
mount --rbind /dev /mnt/build/dev
mount --rbind /proc /mnt/build/proc
mount --rbind /sys /mnt/build/sys
mkdir -p /mnt/build/mnt/host
mount --rbind / /mnt/build/mnt/host
cp -L /etc/resolv.conf /mnt/build/etc/
chroot /mnt/build
```

I removed the make.profile directory from /usr/local/etc/portage and instead symlinked it from /etc/portage
```bash
sudo ln -s /etc/portage/make.profile  /usr/local/etc/portage/ #On crOS and not the build chroot
```

Chrooting into the intermediate build location allows me to start updating packages for the old system, So i started with portage. Then, I tried to build the @world set, but it had no track of what packages is installed. I also referred the dev_install script and it just moved the binary packages to /usr/local. To overcome the problem of breaking core packages, I installed packages at /usr/local. 
```bash
source /etc/profile
export PS1="(chroot) ${PS1}"
(chroot) root # emerge --sync
(chroot) root # emerge --root=/mnt/host/usr/local --config-root=/mnt/host/usr/local --verbose --oneshot sys-apps/portage
(chroot) root # emerge --root=/mnt/host/usr/local --config-root=/mnt/host/usr/local -av1 sys-devel/gcc
```

But while Compiled gcc, It said no space left in root partition(8GB). So, I extended the partition to 32GB and re-did it. I successfully installed it but for some libraries, it was referring the /. Again, It broke the core packages. I also tried to emerge with --prefix option, but it persisted.

I was able to install some binary packages successfully, but some gave me a saying ```keyerror:'/usr/local'```. I tried to update the packages that dev_install script installed using portage, but it gave me a error saying it had internal file collisions. 

So, I tried with FEATURES="-collision-detect -protect-owned" option but it still complained. Also, i removed the package and tried to re-emerge them, but it failed saying ```One or more symlinks to directories have been preserved in order to ensure that files installed via these symlinks remain accessible```

### Summary

After expanding the root partition, I moved the toolchain built using crossdev to crOS and configured it. But crOS core packages are compiled with older version of gcc and glibc which is holding back to compile new packages using portage. So, I thought to upgrade it to using the Upgrading_Gentoo guide. But again, It broke the core packages. I also tried to install to /usr/local but the problem persisted.

So, next week, I will be trying to setup a prefix for crOS. I have brief understanding of it, so i would be following the [manual bootstrap](https://wiki.gentoo.org/wiki/Project:Prefix/Manual_Bootstrap) process and also setting it up stage wise would make debugging easy. As a prerequisite, I should have a toolchain packages which i already have built using crosstool-ng
