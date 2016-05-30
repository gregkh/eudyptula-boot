eudyptula-boot
==============

`eudyptula-boot` boots a Linux kernel in a VM without a dedicated root
filesystem. The root filesystem is the underlying root filesystem (or
some pre-built chroot). This is a convenient way to do quick tests
with a custom kernel.

The name comes from [Eudyptula][] which is a genus for penguins. This
is also the name of a [challenge for the Linux kernel][].

This utility is aimed at development only. This is a hack. It relies
on AUFS/overlayfs and 9P to build the root filesystem from the running
system.

Unfortunately, a change in overlayfs in Linux 4.2 prevents its use
with 9P (change `4bacc9`). It seems a bug difficult to fix. As a
workaround, see the `--readonly` option.

[Eudyptula]: http://en.wikipedia.org/wiki/Eudyptula
[challenge for the Linux kernel]: http://eudyptula-challenge.org/

Also see
[this blog post](http://vincent.bernat.im/en/blog/2014-eudyptula-boot.html)
for a quick presentation of this tool.

Usage
-----

It is preferable to have a kernel with AUFS or OverlayFS
enabled. Ubuntu and Debian kernels are patched to support AUFS. Since
3.18, vanilla kernels have OverlayFS built-in. Ubuntu kernels also
come with OverlayFS support. Check you have one of those options:

    CONFIG_AUFS_FS=y
    CONFIG_OVERLAY_FS=y
    CONFIG_OVERLAYFS_FS=y

Ensure you have the following options enabled (as a module or builtin):

    CONFIG_9P_FS=y
    CONFIG_NET_9P=y
    CONFIG_NET_9P_VIRTIO=y
    CONFIG_VIRTIO=y
    CONFIG_VIRTIO_PCI=y

A minimal configuration should be obtained with:

    $ cat <<EOF > .config.needed
    CONFIG_SHMEM=y
    CONFIG_TMPFS=y
    CONFIG_OVERLAY_FS=y
    CONFIG_SYSFS=y
    CONFIG_PROC_FS=y
    CONFIG_DEVTMPFS=y
    CONFIG_BLK_DEV_INITRD=y
    CONFIG_RD_GZIP=y
    CONFIG_ISA=y
    CONFIG_PRINTK=y
    CONFIG_EARLY_PRINTK=y
    CONFIG_BINFMT_SCRIPT=y
    CONFIG_64BIT=y
    CONFIG_x86_64=y
    CONFIG_UNIX=y
    CONFIG_MAGIC_SYSRQ=y
    CONFIG_FUTEX=y
    CONFIG_MULTIUSER=y
    EOF
    $ make tinyconfig
    $ ./scripts/kconfig/merge_config.sh -m .config .config.needed
    $ make kvmconfig

Once compiled, the kernel needs to be installed in some work directory:

    $ make modules_install install INSTALL_MOD_PATH=$WORK INSTALL_PATH=$WORK

Then, boot your kernel with:

    $ eudyptula-boot --kernel vmlinuz-3.15.0~rc5-02950-g7e61329b0c26

Use `--help` to get additional available options.

Before booting the kernel, the path to GDB socket will be
displayed. You can use it by running gdb on `vmlinux` (which is
somewhere in the source tree):

    $ gdb vmlinux
    GNU gdb (GDB) 7.4.1-debian
    Reading symbols from /home/bernat/src/linux/vmlinux...done.
    (gdb) target remote | socat UNIX:/path/to/vm-eudyptula-gdb.pipe -
    Remote debugging using | socat UNIX:/path/to/vm-eudyptula-gdb.pipe -
    native_safe_halt () at /home/bernat/src/linux/arch/x86/include/asm/irqflags.h:50
    50  }
    (gdb)

A serial port is also exported. It can be convenient for remote
debugging of userland processes. More details can be found in this
[blog post][] (which also covers debugging the kernel).

[blog post]: http://vincent.bernat.im/en/blog/2012-network-lab-kvm.html

QEMU monitor is also attached to a UNIX socket. You can use the
following command to interact with it:

    $ socat - UNIX:/path/to/vm-eudyptula-console.pipe
    QEMU 2.0.0 monitor - type 'help' for more information
    (qemu)

You can also get something similar to [guestfish][]:

    $ eudyptula-boot --qemu="-drive file=someimage.qcow2,media=disk,if=virtio"

[guestfish]: http://libguestfs.org/guestfish.1.html

Alternatives
------------

Similar projects exist:

 - https://github.com/g2p/vido
 - https://git.kernel.org/cgit/utils/kernel/virtme/virtme.git/
