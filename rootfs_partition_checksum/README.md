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

**Compatibility:** The Mkimage utility has undergone radical restructuring since the LEDE OpenWrt merge in January 2018. The patch works with LEDE builds from December 2017, but does not with the most recent clone of OpenWrt (April 2020). For those using new builds of OpenWrt, the kernel patch may apply, but the mkimage patch will need to be refactored significantly.

**Things you must change:** 
1. The should_do_crc() method includes infrastructure to check for an active low gpio pin input to verify whether we should or should not run the crc check. This can all be removed if it does not apply. 
2. Since we are early in the kernel init process, I do not trust the `/proc` filesystem to have been mounted yet. Therefore, this patch refers the mtd partions by their numerical name (i.e. "/dev/mtd1") instead of by their string mapping (i.e. "rootfs"). For the patch to work, you must swap out all instances of "/dev/firmware" with the full path of your "firmware" partition. You also must swap out all instances of "/dev/oops" with the full path to your "oops" partition. If you are not using an oops partition, you can change all instances of "/dev/oops" to be the name of any nonexistent file (meaning you can likely leave it as "/dev/oops"). 

**To apply and use:**
1. Add 'squashfs_crc=1' to the bootargs in your target .dts file. The checksum will only be verified if this bootarg is present.
2. In your firmware image builder, modify your mkimage call to include the -S flag, followed by the name of the sqaushfs blob.
`mkimage ... -S /path/to/root.squashfs ...`
2. clone this repo and change directory to `OpenWrt-fsp-lede-patches/rootfs_partition_checksum/fsp-lede-dec-2017`, and run:<br/>
`mv mkimage-adds-rootfs-partition-squashfs-size-and-crc-to-uImage-header.patch /path_to_your_OpenWrt_trunk/tools/mkimage/patches/`<br/>
`mv 0860-check-rootfs-partition-squashfs-crc.patch /path_to_your_OpenWrt_trunk/target/linux/your_target/patches-3.18/`<br/>
`cd /path_to_your_OpenWrt_trunk/`<br/>
`make`<br/>
