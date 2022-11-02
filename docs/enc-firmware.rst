Encrypted Firware
#################

Overview
********

Zephyr and TFM use the MCUboot_ bootloader to enable secure boot, as
well as secure image updates.  More information can be found at it's
documentation__.

.. _MCUboot: https://www.mcuboot.com/index.html

.. __ : https://docs.mcuboot.com/

The default configuration of MCUboot uses digital signature to sign
the image.  It may be desired to have the firmware image encrypted
when performing upgrades.  In addition, MCUboot supports a mode where
in order to do A/B upgrades, an external SPI flash can be used to hold
the upgrade image.  In this case, it can be desired to have this image
remain encrypted, as the contents are easier to read out than the
internal flash on the MCU.

Walkthrough
***********

To begin with, we will perform a basic walkthrough of performing
flash upgrades and using encrypted images.  This doesn't seem to be
well supported by the tools, so some of these steps will be very
manual, and we will update the instructions as we get support in the
tools.

..
   The single image steps here are just being documented, and will
   probably be removed from the final documentation, since they aren't
   all that interesting to the project.

We are testing this on the LPC55S69, which has a few issues regarding
MCUboot in general:

* The LPC55S69 flash has a 512-byte minimal write alignment.  There is
  an internal ECC managed over 512 byte blocks, and it is not possible
  to write smaller amounts than this.
* MCUboot supports this with "Overwrite mode", however, there is a
  static assert that has to be disabled, which doesn't work with this
  mode.
* Depending on configuration, MCUboot barely fits into the 32
  partition allocated for it.  Changing the partition layout is a bit
  of an ordeal because, although Zephyr allows the partitions to be
  overridden in an application-specific file, TF-M does not have this,
  and as such, patches will have to be applied to TF-M to change the
  partition layout.

We will start by just making an application that is a single image,
build MCUboot, flash it in to the bootloader slow, and build our
application itself, and flash it, making it available for upgrade.

Apply the following patch to MCUboot to fix the compilation error,
remove debugging, and use ECDSA instead of RSA.  Also, configure it
for the flash on the LPC board, with the larger number of smaller
sectors, and upgrade only mode.

MCUboot for the LPC
===================

.. code-block:: diff

    diff --git a/boot/bootutil/include/bootutil/bootutil_public.h b/boot/bootutil/include/bootutil/bootutil_public.h
    index a2bf9b23..e174c1d6 100644
    --- a/boot/bootutil/include/bootutil/bootutil_public.h
    +++ b/boot/bootutil/include/bootutil/bootutil_public.h
    @@ -83,8 +83,10 @@ extern "C" {
     
     #ifdef MCUBOOT_BOOT_MAX_ALIGN
     
    +#ifndef CONFIG_BOOT_UPGRADE_ONLY
     _Static_assert(MCUBOOT_BOOT_MAX_ALIGN >= 8 && MCUBOOT_BOOT_MAX_ALIGN <= 32,
                    "Unsupported value for MCUBOOT_BOOT_MAX_ALIGN");
    +#endif
     
     #define BOOT_MAX_ALIGN          MCUBOOT_BOOT_MAX_ALIGN
     #define BOOT_MAGIC_ALIGN_SIZE   ALIGN_UP(BOOT_MAGIC_SZ, BOOT_MAX_ALIGN)
    diff --git a/boot/zephyr/prj.conf b/boot/zephyr/prj.conf
    index e4c01294..fb70572b 100644
    --- a/boot/zephyr/prj.conf
    +++ b/boot/zephyr/prj.conf
    @@ -1,6 +1,9 @@
    -CONFIG_DEBUG=y
    +CONFIG_DEBUG=n
     CONFIG_PM=n
     
    +### Use ECDSA P256 signatures instead of RSA.
    +CONFIG_BOOT_SIGNATURE_TYPE_ECDSA_P256=y
    +
     CONFIG_MAIN_STACK_SIZE=10240
     CONFIG_MBEDTLS_CFG_FILE="mcuboot-mbedtls-cfg.h"
     
    @@ -23,6 +26,9 @@ CONFIG_BOOT_BOOTSTRAP=n
     
     CONFIG_FLASH=y
     
    +# Set this for the LPC boards. It shouldn't be here.
    +CONFIG_BOOT_MAX_IMG_SECTORS=320
    +
     ### Various Zephyr boards enable features that we don't want.
     # CONFIG_BT is not set
     # CONFIG_BT_CTLR is not set
    @@ -34,3 +40,6 @@ CONFIG_LOG_MODE_MINIMAL=y # former CONFIG_MODE_MINIMAL
     CONFIG_LOG_DEFAULT_LEVEL=0
     ### Decrease footprint by ~4 KB in comparison to CBPRINTF_COMPLETE=y
     CONFIG_CBPRINTF_NANO=y
    +
    +# For LPC55S69, overwrite only
    +CONFIG_BOOT_UPGRADE_ONLY=y

We can now build this MCUboot for the LPC:

.. code-block:: shell

   # Build MCUboot
   west build -p auto -d build_mcuboot -b lpcxpresso55s69_cpu0 \
       ../bootloader/mcuboot/boot/zephr

   # Flash it to the boot loader partition.
   west flash --bin-file build/zephyr/zephyr.signed.bin

At this point, booting the target should show a message about being
unable to find a signed image.

An unencrypted image
--------------------

Edit the ``samples/hello_world/prj.conf`` file in the Zephyr tree, and
add a line with ``CONFIG_BOOTLOADER_MCUBOOT=y``, and just build
this application:

.. code-block:: shell

   west build -p auto -b lpcxpresso55s69_cpu0 \
       samples/hello

For this case, the ``west sign`` is almost sufficient for what we need.
However there is a bug in the imgtool command, which rejects the
alignment on the LPC device.  We can invoke imgtool ourselves, and
just pass in a bogus alignment (it isn't used in overwrite mode):

.. code-block:: shell

   # west sign -t imgtool -- --key ../bootloader/mcuboot/root-rsa-2048.pem
   imgtool sign -v 0.0.0 --header-size 512 --slot-size 98304 \
       --align 8 \
       --key ../bootloader/mcuboot/root-ec-p256.pem \
       build/zephyr.bin build/zephyr/zephyr.signed.bin

This will sign the image, using the developer key from the MCUboot
tree.

We can flash this image into slot0 using west:

.. code-block:: shell

   west flash --bin-file build/zephyr/zephyr.signed.bin

West will know from the build configuration that this image needs to
be placed into slot 0.  This should then boot, printing the hello
world message.

MCUmgr
======

The mcumgr tool allows for firmware upgrades over various
communication channels.  With a board like the LPC, that doesn't have
wifi or ethernet, using a UART for this is quite helpful.  The LPC
board has a Click board on it, and there are several options available
that make a secondary UART available as a USB device.  This allows us
to still have debug messages and the shell available, while being able
to use mcumgr.  The mcumgr protocol was designed to be able to be
muxed, and you can probably get away with it on the same port, but
keep in mind that depending on timing, debug messages may go to the
mcumgr program, which will simply discard them.

The following patch reconfigures the sample application to use this
second UART for debugging and shell, and the primary one for mcumgr:

.. code-block:: diff

   commit 0aa1301d244723435b85839bed16806391913482
   Author: David Brown <david.brown@linaro.org>
   Date:   Mon Oct 31 16:29:51 2022 -0600
   
       Rearrange the uarts for split debugging
       
       Move the console UART onto flexcomm2, which is the UART on the Chip
       board.  This can be accessed via a small Chip board with a UART to USB
       adaptor on it.
   
   diff --git a/boards/arm/lpcxpresso55s69/lpcxpresso55s69_cpu0.dts b/boards/arm/lpcxpresso55s69/lpcxpresso55s69_cpu0.dts
   index 1da0fb86332..79b98bf47cc 100644
   --- a/boards/arm/lpcxpresso55s69/lpcxpresso55s69_cpu0.dts
   +++ b/boards/arm/lpcxpresso55s69/lpcxpresso55s69_cpu0.dts
   @@ -38,9 +38,10 @@
    		zephyr,code-partition = &slot0_partition;
    		zephyr,code-cpu1-partition = &slot1_partition;
    		zephyr,sram-cpu1-partition = &sram3;
   -		zephyr,console = &flexcomm0;
   -		zephyr,shell-uart = &flexcomm0;
   +		zephyr,console = &flexcomm2;
   +		zephyr,shell-uart = &flexcomm2;
    		zephyr,entropy = &rng;
   +		zephyr,uart-mcumgr = &flexcomm0;
    	};
    
    	gpio_keys {
   @@ -105,6 +106,10 @@
    	status = "okay";
    };
    
   +&flexcomm2 {
   +	status = "okay";
   +};
   +
    &flexcomm4 {
    	status = "okay";
    };

We can enable MCUmgr in the hello world application with this patch:

.. code-block:: diff

   commit c52ba4fbc96f72e03cc60f6c57827a488e3e75e7
   Author: David Brown <david.brown@linaro.org>
   Date:   Mon Oct 31 16:30:40 2022 -0600

       Enable the mcumgr calls

       Enables these calls using the UART defined by the DTS file.

   diff --git a/samples/hello_world/prj.conf b/samples/hello_world/prj.conf
   index b2a4ba59104..82856e2d95f 100644
   --- a/samples/hello_world/prj.conf
   +++ b/samples/hello_world/prj.conf
   @@ -1 +1,22 @@
    # nothing here
   +CONFIG_SHELL=y
   +CONFIG_KERNEL_SHELL=y
   +
   +CONFIG_DEBUG=y
   +
   +CONFIG_MCUMGR=y
   +# CONFIG_MCUMGR_SMP_SHELL=y
   +CONFIG_MCUMGR_SMP_UART=y
   +
   +CONFIG_BOOTLOADER_MCUBOOT=y
   +
   +CONFIG_FLASH=y
   +
   +CONFIG_STATS=y
   +CONFIG_STATS_NAMES=y
   +
   +#CONFIG_MCUMGR_CMD_IMG_MGMT=y
   +CONFIG_MCUMGR_CMD_OS_MGMT=y
   +CONFIG_MCUMGR_CMD_STAT_MGMT=y
   +CONFIG_OS_MGMT_TASKSTAT=y
   +CONFIG_OS_MGMT_RESET_HOOK=y
   diff --git a/samples/hello_world/src/main.c b/samples/hello_world/src/main.c
   index 9a90c6a6a0a..a9f1c40cf2a 100644
   --- a/samples/hello_world/src/main.c
   +++ b/samples/hello_world/src/main.c
   @@ -6,7 +6,15 @@

    #include <zephyr/kernel.h>

   +/* This is an ugly mess. */
   +#ifdef CONFIG_MCUMGR_CMD_OS_MGMT
   +  #include <os_mgmt/os_mgmt.h>
   +#endif
   +
    void main(void)
    {
   +#ifdef CONFIG_MCUMGR_CMD_OS_MGMT
   +	os_mgmt_register_group();
   +#endif
    	printk("Hello World! %s\n", CONFIG_BOARD);
    }

Building and flashing this image should work as above, but on the new
secondary UART, you should see, in addition to the hello world
message, a shell prompt.

We can try talking to this with MCUmgr.

.. code-block:: shell

   # Setup the connection, only needs to be done once.  Use the
   # appropriate serial device for your setup.
   $ mcumgr conn add serial dev=/dev/ttyACM0,baud=115200

   # Sent a an echo request
   $ mcumgr -c serial echo 'Ping message'
   Ping message

You should see the message echoed back to you if everything is
working.
