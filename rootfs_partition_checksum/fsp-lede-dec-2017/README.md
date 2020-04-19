**Compatibility:** The Mkimage utility has undergone radical restructuring since the LEDE OpenWrt merge. The patch works with LEDE builds from December 2017, but does not with the most recent clone of OpenWrt (April 2020). For those using new builds of OpenWrt, the kernel patch may apply, but the mkimage patch will need to be refactored significantly.

**Things you must change:** 
1. The should_do_crc() method includes infrastructure to check for gpio pin input to verify whether we should or should not run the crc check. This can all be removed if it does not apply. 
2. Since we are early in the kernel init process, I do not trust the `/proc` filesystem to have been mounted yet. Therefore, this patch refers the mtd partions by their numerical name (i.e. "/dev/mtd1") instead of by their string mapping (i.e. "rootfs"). For the patch to work, you must swap out all instances of "/dev/firmware" with the full path of your "firmware" partition. You also must swap out all instances of "/dev/oops" with the full path to your "oops" partition. If you are not using an oops partition, you can change all instances to be the name of any nonexistent file (meaning you can likely leave it as /dev/oops). 

**To apply and use:** 
