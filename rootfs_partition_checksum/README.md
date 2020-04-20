**Prerequisites:**
* flash layout (and firmware image) in the following order: kernel, rootfs, rootfs_data
* failsafe partition that is loaded when a reboot count is exceeded
* boot latency is not critical
* last 8 bytes of uImage name not being used

**Short Bug Description:** By default, the rootfs partition (aka the squashfs region) is NOT checksummed after being written during a firmware upgrade. If the mtd write during the upgrade is interrupted (i.e. power off), it is possible (and very likely) that you will then boot into a firmware image with corrupted rom.

**Long Bug Description:** This bug is concerned with upgrade systems that use the "legacy" u-boot bootloader format, which includes several partitions of instructions concatenated together, preceeded by a 64-byte uImage header. Using this format,  flash layouts for boards running OpenWrt/lede [look like this](https://openwrt.org/docs/techref/flash.layout), with the "firmware" partition consisting of linux kernel instructions followed by the /rom squashfs instructions, followed by the jffs2 overlay region. During the creation of the uImage using the [mkimage utility](https://linux.die.net/man/1/mkimage), a checksum is run on the "kernel" partition and included in the uImage header. Then, the "rootfs" blob is appended to the kernel instructions to make the firmware image. Upon first boot after the upgrade, u-boot runs a crc check on the kernel partition, and will print the following at boot if all goes well:
`Verifying Checksum ... OK`
However, if the upgrade is interrupted during the writing of the rootfs partition, the corrupted firmware goes unchecked, and when Procd attempts to launch those userspace processes, you might see the following message sprinkled throughout the kernel message buffer:
`[ ... ] SQUASHFS error: Unable to read page, block ..., size ...` and/or
`[ ... ] SQUASHFS error: Unable to read data cache entry [ ... ]`
and the processes will fail to launch. This undefined behavior is certainly dangerous, and should be avoided at all costs.

**The Patch:** This patch adds functionality to mkimage to calculate the crc32 of a rootfs blob whose filename will also be passed into the mkimage call. The uImage header includes a 32-byte "name" field, and this patch commandeers the last 8 bytes of that field for the rootfs file size and checksum. This means that with this patch, the uImage name can only be 24 bytes in length.  Then, on the read side, a hook is injected into the kernel init sequence (which must have passed the u-boot checksum) that reads the crc and filesize from the uImage header and verifys that the calculated checksum of the current rootfs partition matches the expected checksum. If not, the kernel panics and (if applicable) writes a note to the "oops" partition. With this patch, we have ensured that both the kernel and rootfs partitions are verified as error free.

**Reasons for not Submitting to official Repo:** This patch would break any system that requires the last 8 bytes of the uImage name. The other alternative would have been to change the entire format of the uImage header, which would require reflashing the u-boot partition, which is to be avoided at all costs.
