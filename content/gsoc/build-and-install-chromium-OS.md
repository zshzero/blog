---
title: "Build and Install Chromium OS dev"
date: 2020-06-05T06:47:12+05:30
draft: false
---

This post is a quick start guide to get the Chromium OS source code, build it and try it out for BOARD=arm-generic. 

The official guide is at: [Developer Guide](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md)

These are the steps and commands I used while following the official guide:

##### Prerequisite: You must have any up-to-date Linux distribution on a x86_64 64-bit system with sudo access to develop Chromium OS (They recommend and offer support to Ubuntu 16.04 - Xenial)

1. Install git revision control system, the curl download helper, and lvm tools
```bash
sudo apt-get install git-core gitk git-gui curl lvm2 \ 
     thin-provisioning-tools python-pkg-resources python-virtualenv \
     python-oauth2client xz-utils python3.6
```

2. Clone depot_tools and add them to your PATH
```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
# Add depot_tools to your PATH(probably in your ~/.bashrc or ~/.zshrc):
export PATH=/path/to/depot_tools:$PATH
```

3. Tweak your sudoers configuration to turn off the tty_tickets option
```bash
cd /tmp
cat > ./sudo_editor <<EOF
#!/bin/sh
echo Defaults \!tty_tickets > \$1          
echo Defaults timestamp_timeout=180 >> \$1
EOF
chmod +x ./sudo_editor
sudo EDITOR=./sudo_editor visudo -f /etc/sudoers.d/relax_requirements
```

4. Setup git now
```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

5. Set default file permissions (umask) in your ~/.bashrc or or ~/.zshrc file 
```bash 
umask 022 
``` 

6. Create the directory for your source code with this command:
```bash
mkdir -p ~/chromiumos
```
If you are not willing to create it in your home directory, refer : [Link](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md#decide-where-your-source-will-live)

7. Get the public source code

You will have a much nicer time if you also have a fast multi-processor machine with lots of memory and good Internet connection
```bash
cd ~/chromiumos
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git
repo sync
```

8. Install the chroot (assuming you’re in chromiumos directory) 
```bash
cros_sdk
```
- Above command is also used to enter the chroot
- I will use the label, "(chroot)" which indicates that you should run commands inside the chroot


9. Choose the board for your build
```bash
(chroot) export BOARD=amd64-generic
```
If you are building for different board, [refer](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md#select-a-board)

10. Initialize the build for your board
```bash
(chroot) setup_board --board=${BOARD}
```

11. Set a password for your default user, chronos
```bash
(chroot) ./set_shared_user_password.sh
```

12. Build all the packages for your board
```bash
(chroot) ./build_packages --board=${BOARD}
```

13. Build the Chromium OS developer image
```bash
(chroot) ./build_image --board=${BOARD} --noenable_rootfs_verification dev
```

You can adapt the previously built image so that it is usable by kvm 
```bash
(chroot) ./image_to_vm.sh --board=${BOARD}
```
- This command creates the file ~/trunk/src/build/images/${BOARD}/latest/chromiumos_qemu_image.bin

14. Flash the image on a USB disk
```bash
(chroot) cros flash usb:// ${BOARD}/latest

#IF your USB is not recognized by cros
(chroot) cros flash usb:///dev/sdxY ${BOARD}/latest # replace with your USB parition
```

15. Boot with your USB disk and install Chromium OS on your hard disk 

##### NOTE: Installing Chromium OS onto your hard disk will Wipe Your Hard Disk Clean.
```bash
/usr/sbin/chromeos-install --dst /dev/sdxY # replace with your Hard disk parition
```


This guide is just the creamly layer on what is possible – it is best to read the Official Developer guide
