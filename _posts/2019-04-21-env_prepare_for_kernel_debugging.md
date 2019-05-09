---
layout: post
title: use gdb kgdb and two virtual machines to debug linux kernel
date: 2019-02-22 17:30:00
tags: gdb kgdb kernel debugging

---
how to build environment for debugging linux kernel with gdb and kgdb 

1. make menu config - to enable KGDB and build the vmlinux file from linux source files. and then make modules and make install to install the new vmlinux file to the current os and replace the kernel with your builded one.

2. update-grub to update the boot file, boot the new kernel when system boot.

3. the upper two steps are all on the target virtual machines

4. on the host machine, you must ensure the sshd service is enabled, and copy the vmlinux file you build on the target virtual machine to it.

5. on the target machine, edit /boot/grub/grub.cfg, add the following to the new version of boot you update before

   kgdbwait kgdboc=ttyS0, 115200

6. disable GRUB_HIDDEN_TIMEOUT=0 and GRUB_HIDDEN_TIMEOUT_QUIET=true in file /etc/default/grub on the target machine.

   and this time, if you reboot the target machine, the os will froze and wait the host machine to connect it with gdb

**NOTE:** pay attention to the two arguments in `/boot/grub/grub.cfg`, GRUB_HIDDEN_TIMEOUT and GRUB_HIDDEN_TIMEOUT_QUIET, if you don't disable them, grub will not show selections and directly reboot with the old kernel version when you restart.

**To enable CONFIG_KGDB you should first turn on "Prompy for development and/or incomplete code/drivers" (CONFIG_EXPERIMENTAL) in "General setup", then under the "Kernel debugging" select "KGDB: kernel debugging with remote gdb". You can see this by "make menuconfig".**

set serial port on virtualbox. settings->serial ports, on the target virtual machine:

![serial port settings for target](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/serial_ports_settings.png)

on the host virtual machine:

![serial port setting for host](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/serial_port_setting_for_host.png)

NOTE: you should start the target machine first, and then to start host machine, or you will occur an error that can not connect to serial port because we have selected "Connected to existing pipe/socket", the "/tmp/serial" will be created on the target machine. 

At the end, of course you could have other ways to build your debugging environment.
