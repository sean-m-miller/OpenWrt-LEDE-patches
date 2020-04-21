**The Bug:** If a config is preserved across a firmware upgrade via the sysupgrade utility (`sysupgrade -c`), the /tmp RAM overlay is skipped and the jffs2 overlay is prepared and mounted while the preinit_main script hangs. This is a bug and can lockup boards with large "rootfs_data" partitions. 

**Description:** In 2008, the [ability to preserve a config across sysupgrades](https://github.com/bmork/OpenWrt/blob/master/package/system/mtd/src/jffs2.c) was added (see mtd_replace_jffs2()). The mechanism, which is still the behavior in the offical OpenWrt repo as of April 2020, writes the tar file to be preserved at the beginning of the rootfs_data partition, and moves the [0xdeadc0de flag that signifies the rootfs/rootfs_data boundary](https://openwrt.org/docs/techref/filesystems) to be at the start of the next erase block after the tar file. The file is written as a valid jffs2 node, which requires that it is preceded by a jffs2 file header. jffs2 file headers have 0x1985 as the first 2 bytes, which matches the first 2 bytes of the jffs2 ["CLEANMARKER"](https://github.com/m-labs/openwrt-milkymist/blob/master/package/mtd/src/jffs2.c), which is defined as 0x198520030000000cf060dc98.

Then, on reboot, the [80_mount_root](https://github.com/openwrt/openwrt/blob/master/package/base-files/files/lib/preinit/80_mount_root) script calls the [mount_root](https://git.openwrt.org/?p=project/fstools.git;a=blob;f=mount_root.c) utility, which examines the first bytes of the rootfs_data partition using [this](https://lxr.openwrt.org/source/fstools/libfstools/mtd.c) logic. It examines the first four bytes of the partition to try to match with 0xdeadc0de to set the FS_DEADCODE case, but instead matches with the 2 byte 0x1985 (due to the jffs2 file header format), setting the FS_JFFS2 case. Therefore, the mount_root utility gets "tricked" into thinking that the rootfs_data partition has already been cleaned, with every erase block being set to the jffs2 cleanmarker followed by 0xff until the next block boundary. This causes mount_root to immediately launch the jffs2 driver to mount the jffs2 overlay with the assumption that there is no more cleaning or formatting work to be done. [This behaviour is undesired](https://openwrt.org/docs/techref/preinit_mount) (see steps 3-5 in the **Mount Root Filesystem** section), since it hangs the preinit main while the jffs2 driver cleans and formats the rootfs_data partition. 
The desired behavior is to have mount_root fall into the FS_DEADCODE case during the 80_mount_root script, which would mount the /tmp RAM overlay, and defer the switch to the jffs2 overlay until the [/etc/init.d/done](https://github.com/openwrt/openwrt/blob/master/package/base-files/files/etc/init.d/done) script gets called at the end of the init sequence. The intended behaviour is reiterated in a comment found in the FS_DEADCODE case in [mount_root.c](https://github.com/ianclegg/openwrt-fsutils/blob/master/mount_root.c), "Filesystem isn't ready yet and we are in the preinit, so we can't afford waiting for it. Use tmpfs for now and handle it properly in the 'done' call".

The current design is also "sloppy" because the jffs2 driver looks for the 0xdeadc0de marker as the starting point to begin cleaning and reformatting blocks for the jffs2 overlay, and since the 0xdeadc0de marker was moved to be after the tar file, the jffs2 driver never cleans the blocks at the beginning of the rootfs_data partition that were used to store the tar file. Those blocks get cleaned before use at some point during run time. Certainly, we should be launching the jffs2 overlay with a totally clean rootfs_data partition.

To verify that this bug exists on your processor, after rebooting from a sysupgrade that preserves a config, you will find the following in `dmesg`:<br/>
`[   11.600000] jffs2_scan_eraseblock(): End of filesystem marker found at 0x10000`<br/>
`[   11.600000] jffs2_build_filesystem(): unlocking the mtd device... done`<br/>
`[   11.600000] jffs2_build_filesystem(): erasing all blocks after the end marker... done`<br/>
These messages are from the jffs2 driver, which is getting called during preinit main and is failing to erase the blocks used to store the tar file (the blocks from 0x0 - 0x10000).

The desired behavior would instead print the following: <br/>
`[   11.480000] mount_root: jffs2 not ready yet, using temporary tmpfs overlay`<br/>
This shows mount_root falling into the FS_DEADCODE case, kicking off the RAM overlay, and deferring the rootfs_data cleaning until after preinit main has finished. Then Later in the boot sequence:<br/>
`[   44.040000] jffs2_scan_eraseblock(): End of filesystem marker found at 0x0`<br/>
`[   44.040000] jffs2_build_filesystem(): unlocking the mtd device... done.`<br/>
`[   44.040000] jffs2_build_filesystem(): erasing all blocks after the end marker...`<br/>
This signifies that jffs2 driver was called about 30 seconds later (during the done script), and that all rootfs_data blocks were successfully erased before the mounting of the jffs2 overlay.

**The Patch:** The patch changes the behavior of sysupgrade to keep the 0xdeadc0de marker at the rootfs/rootfs_data boundary, and writes the file to raw flash in the following erase blocks. Since the 0xdeadc0de is kept where it belongs, upon reboot after the upgrade, mount_root falls into the FS_DEADCODE case and launches the /tmp RAM overlay. Then it reads the file from flash and untars the file into the new RAM root directory. When the `/etc/init.d/done` script calls `mount_root done`, the entire rootfs_data partition (including the region used to store the config file) is cleaned, and then the jffs2 overlay is mounted and some files are copied back to the rootfs_data partition as jffs2 files.

**Reasons for not Submitting to official Repo:** This patch is compatible with all versions of OpenWrt released after 2008. Unfortunately, to downgrade from firmware including this patch to an earlier version would require creating an intermediate custom firmware consisting of the target firmware with only the read-side .patch file applied and sysupgrading to that firmware, followed by a sysupgrade to the target firmware without either of the .patch files applied. Otherwise, the preserved config file would be corrupted and lost during the downgrade. This makes this patch unusable for some OpenWrt users.

**Compatibility:** A patch was added in August of 2018 that made all OpenWrt distributions thereafter require a one line change to this patch. If you pulled OpenWrt after August 2018, use the patch files contained in OpenWrt-apr-2020/. If you are using an OpenWrt version from before August 2018 or LEDE, use the patch files contained in LEDE-dec-2017/.

**Things in this Patch You Must Change** None.

**To Apply and Use:** Clone this repo and change directory to `OpenWrt-fsp-lede-patches/sysupgrade_ram_overlay_when_preserving_config/your_version`, and run:<br/>
`mkdir /your_OpenWrt_trunk_path/package/system/fstools/patches/`<br/>
`mv 010-restore_tar_from_rootfs_data.patch /your_OpenWrt_trunk_path/package/system/fstools/patches/`<br/>
`mkdir /your_OpenWrt_trunk_path/package/system/mtd/patches/`<br/>
`mv 010-write_tar_to_rootfs_data.patch /your_OpenWrt_trunk_path/package/system/mtd/patches/`<br/>
`cd /your_OpenWrt_trunk_path/`<br/>
`make`<br/>
