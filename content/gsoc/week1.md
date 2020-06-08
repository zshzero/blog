---
title: "GSoC Week 1"
date: 2020-06-07T12:00:09+05:30
draft: false 
---

This week, I built toolchain for crOS (An acronmyn which I will be referring to ChromiumOS this entire series)

My first attempt was to create binary package for toolchain packages using crossdev

- Create an overlay specifically for crossdev's use
```bash
mkdir -p /var/db/repos/crossdev
mkdir -p /var/db/repos/crossdev/{profiles,metadata}
echo 'crossdev' > /var/db/repos/crossdev/profiles/repo_name 
echo 'masters = gentoo' > /var/db/repos/crossdev/metadata/layout.conf
chown -R portage:portage /var/db/repos/crossdev
```

- Then instruct portage and crossdev to use this overlay:
```bash
#FILE /etc/portage/repos.conf/crossdev.conf
[crossdev]
location = /var/db/repos/crossdev
priority = 10
masters = gentoo
auto-sync = no
```

- Install crossdev and build toolchain for crOS:
```bash
emerge --ask crossdev
crossdev --stable -t x86_64-cros-linux-gnu
```

- After building toolchain, use quickpkg to create binary packages of them:
```bash
quickpkg cross-x86-cros-linux-gnu/binutils
quickpkg cross-x86-cros-linux-gnu/gcc
quickpkg cross-x86-cros-linux-gnu/gdb
quickpkg cross-x86-cros-linux-gnu/glibc
quickpkg cross-x86-cros-linux-gnu/linux-headers
```

- Created binary packages are pointed by the PKGDIR variable which defaults to /var/cache/binpkgs 

- On crOS, unpack the packages using [quickunpkg](https://github.com/zoobab/quickunpkg):
```bash
./quickunpkg binutils-2.33.1-r1.tbz2
```

But emerge command gave an error saying that it was missing make and patch. Even adding make package would not have helped as it also requires make

So, I built the the toolchain using [crosstool-ng](https://crosstool-ng.github.io/)

crosstool-NG targets at building toolchains, and only toolchains. They are ready to use and quite easy to install and setup. 

They can be general purpose and pre-configured for the majority targets


- Install ct-ng by referring the [docs](https://crosstool-ng.github.io/docs/) or by:
```bash
emerge -a sys-devel/ct-ng
```

- Choose one out of several sample configurations shipped by ct-ng:
```bash
ct-ng list-samples
ct-ng show-x86-unknown-linux-gnu
```

- Load the sample configuration as a base and fine-tune using ct-ng menuconfig
```bash
ct-ng x86-unknown-linux-gnu
```

- ct-ng is configured with a configurator presenting a menu-structured set of options like Gentoo's kernel configurator
```bash
ct-ng menuconfig
```

My configuration for the toolchain : [Github gist](https://gist.github.com/zshzero/eafcbdb283c4629881ad03e3d70fd6bf)

- Build the toolchain as user:
```bash
ct-ng build
```

My toolchain build : [Gitlab](https://gitlab.com/zshzero/cross-x86_64-cros-linux-gnu)

- After building it, Add the toolchain's bin directory to PATH
```bash
export PATH="${PATH}:/your/toolchain/path/bin"
```

### Summary

I tried building toolchain using gentoo's crossdev but failed due to the above reason. So, I built  it using crosstool-ng as it was ready to use and quite easy to install and setup. 

Next, I will be setting it up on crOS which will now help portage compile and build packages. Also, need to add necessary libraries that are missing in toolchain and configure portage to use it.
