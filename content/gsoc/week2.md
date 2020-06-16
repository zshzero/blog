---
title: "GSoC Week 2"
date: 2020-06-14T09:52:56+05:30
draft: false
---

After Building and Installing crOS for amd64, i just needed a google account to sign in and was ready to go

[crosh](https://chromium.googlesource.com/chromiumos/platform2/+/master/crosh/README.md#Crosh-The-Chromium-OS-shell) is the terminal prompt used by crOS which can be opened by crtl+alt+t and the first warning the shell gave was:  

```bash: warning: /home/chronos/user/.bash_profile: warning: script from noexec mount; see https://chromium.googlesource.com/chromiumos/docs/+/master/security/noexec_shell_scripts.md```  

```bash: warning: /home/chronos/user/.bashrc: warning: script from noexec mount; see https://chromium.googlesource.com/chromiumos/docs/+/master/security/noexec_shell_scripts.md```

CrOS has noexec option in order to reject attempts to execute code as a part of its extremely security. So, I had to remount the partition with exec(permits to execute binaries) and rw(permits to read/write filesystem)

```bash
sudo mount -o rw,exec,remount /
sudo mount -o rw,exec,remount /var
sudo mount -o rw,exec,remount /home/chronos/user
```

I cloned the previously built toolchain from [gitlab](https://gitlab.com/zshzero/cross-x86_64-cros-linux-gnu), renamed the folder as x86_64-cros-linux-gnu and moved to /usr/local. Also, I set path in ~/.bashrc
```bash
export PATH=/usr/local/x86_64-cros-linux-gnu/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/x86_64-cros-linux-gnu/x86_64-cros-linux-gnu/lib64:$LD_LIBRARY_PATH
```

Symlink toolchain binaries to /usr/bin/
```bash
sudo ln -s /usr/local/x86_64-cros-linux-gnu/bin/x86_64-cros-linux-gnu-cc /usr/bin/cc
sudo ln -s /usr/local/x86_64-cros-linux-gnu/bin/x86_64-cros-linux-gnu-gcc /usr/bin/gcc
```
- Create symlink like above if something complains about missing commands

Configuring the repository can be done in a few simple steps. First, if it does not exist, create the repos.conf directory, then copy the Gentoo repository configuration file provided [here](https://wiki.gentoo.org/wiki//etc/portage/repos.conf/gentoo.conf) to /etc/portage/repos.conf/gentoo.conf
```bash
sudo mkdir -p /etc/portage/repos.conf
sudo nano /etc/portage/repos.conf/gentoo.conf #paste the content
```

Fetch the latest snapshot (which is released on a daily basis) from one of Gentoo's mirrors and install it onto the system
```bash
sudo emerge-webrsync
```

- (Optional) ```emerge --sync``` command will use the rsync protocol to update the Gentoo ebuild repository(up to 1 hour) which was fetched earlier on through emerge-webrsync

Now, build make tool which helps in generation of executables from program's source files

Download the latest tar from [Link](https://ftp.gnu.org/gnu/make/), extract and build it:
```bash
tar -xzf make-4.3.tar.gz
sudo ./configure --disable-dependency-tracking --prefix=/usr/local
sudo ./build.sh
sudo ./make
sudo ./make install 
```

And patch tool for producing patched versions of the package

Download the latest tar from [Link](https://ftp.gnu.org/gnu/patch/), extract and build it:
```bash
tar -xzf patch-2.7.tar.gz
sudo ./configure --prefix=/usr/local
sudo make
sudo make install 
```

Now the toolchain is set, we need to instruct portage to compile and build the package. But crOS downloads packages from where binhost is pointing and only emerges them. So, we need to remove that option out from /etc/portage/make.profile/make.defaults
```bash
#Comment the line below
EMERGE_DEFAULT="--getbinpkgs --usepkgonly"
```

Then, I setup the make.conf for portage ([My make.conf](https://pastebin.com/fvqL684P))

I wanted to update the profile to latest(17.1) as CrOS uses a old profile(10.0). I made a backup of existing profile (because it was not a symlink) and set it to ```default/linux/amd64/17.1/desktop```
```bash
sudo emerge -a app-admin/eselect
sudo mv /etc/portage/make.profile /etc/portage/make.profile.bak 
sudo eselect profile set default/linux/amd64/17.1/desktop
```

While trying to update the world, it gave a circular dependencies of gcc and glibc. So removed the symlink and used the backup(default profile). 

I decided to emerge both and started with sys-devel/binutils ```sudo emerge -a sys-devel/binutils``` and the first warning was ``` ```

Commenting app-portage/elt-patches and sys-develel/m4 in /etc/portage/make.profile/package.provided/package.provided

- During error, ```Could not find source /lib/gentoo/functions.sh!```  
I Added [/lib/gentoo/functions.sh](https://github.com/gentoo/gentoo-functions/blob/master/functions.sh)
```bash
sudo mkdir /lib/gentoo
sudo nano /lib/gentoo/functions.sh # Paste the content from the link
```

And again tried to emerge binutils, ```Error: zlib.h : No such file or directory```

So, i installed that library
```bash
sudo emerge -a sys-libs/zlib
```
But it still complained about it.

After referring from stackoverflow, [Link 1](https://stackoverflow.com/questions/4980819/what-are-the-gcc-default-include-directories) and [Link 2](https://stackoverflow.com/questions/16710047/usr-bin-ld-cannot-find-lnameofthelibrary/21647591#21647591)

I had to symlink zconf.h and zlib.h to toolchain's include directory (according to link 1) and libz.so to toolchain's lib directory(according to link 2)
```bash
sudo ln -s  /usr/include/z*.h /usr/local/x86_64-cros-linux-gnu/lib/gcc/x86_64-cros-linux-gnu/8.3.0/include/
sudo ln -s /usr/lib/libz* /usr/local/x86_64-cros-linux-gnu/x86_64-cros-linux-gnu/sysroot/usr/lib/
```

Then, binutils emerged successfully

Next, I tried to emerge sys-devel/gcc. And it said library gmp.h was missing. So, installed it and followed the same as with zlib's error (same in case of mpfr and mpc)

Afterwards, it complained about not having the right verion of glibc ```Error: Needed GLIBC Version 2.29```

### Summary

I remounted the paritions with read/write and execute option. Then downloaded the toolchain repository, moved it to the right directory, set the path and symlinked them to /usr/bin. Later, I configured portage to pull form the gentoo repository, fetched the latest snapshot, setup make.conf and ask it to compile and build the package. Also, i did manually build some packages from source that were necessary for portage.

I tried to update the profile but got struck in circular dependencies. So, I am emerging those packages now. I was able to emerge sys-devel/binutils but now struck with emerging sys-devel/gcc.
