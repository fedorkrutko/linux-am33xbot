# linux-am33xbot
Arch Linux ARM Kernel with botic patches (for Beaglebone Black / Wireless). Please refer to [DiyAudio thread](https://www.diyaudio.com/forums/twisted-pear/258254-support-botic-linux-driver.html) for more information on the driver and its development and checkout Cronus and Hermes-BBB from [Twistedpearaudio](http://twistedpearaudio.com/landing.aspx) for the background on the required cape for the Beaglebone board. Credit for the development of the cape and driver to Russ White from TPA and [Miero](https://github.com/miero).

Follow these steps to build a boticized kernel for recent Arch Linux ARM:

1. Make sure you have either a VM or a native install of Arch Linux (x86-64) and that you are able to build packages: `pacman -S --needed base-devel`
2. Install the cross-compiler required to build the kernel for the armv7h architecture (AUR: [arm-linux-gnueabihf-gcc-linaro-bin](https://aur.archlinux.org/packages/arm-linux-gnueabihf-gcc-linaro-bin/))
3. Clone this repository and delete the `.git`hidden folder created by the git tool (otherwise some of the patches will not apply)
4. Run `CARCH=armv7h makepkg -A` in the created folder to build the package
5. Copy the created package (pkg.tar.xz file) to the BBB / BBBW
6. Uninstall the standard kernel package (`pacman -R linux-am33x`). ATTENTION: You need to install the new kernel before rebooting / power off
7. Install the package with `pacman -U`

Before rebooting ensure that Arch Linux ARM is configured to load the required botic overlay and edit /boot/boot.txt accordingly, see following example (which disables also HDMI and the onboard oscillator):
```
# After modifying, run ./mkscr

if test -n ${distro_bootpart}; then setenv bootpart ${distro_bootpart}; else setenv bootpart 1; fi
part uuid ${devtype} ${devnum}:${bootpart} uuid

setenv bootargs "console=tty0 console=${console} root=PARTUUID=${uuid} rw rootwait"

if load ${devtype} ${devnum}:${bootpart} ${kernel_addr_r} /boot/zImage; then
  gpio set 54
  echo fdt: ${fdtfile}
  if load ${devtype} ${devnum}:${bootpart} ${fdt_addr_r} /boot/dtbs/${fdtfile}; then
    gpio set 55

    fdt addr ${fdt_addr_r}
    #take out the framer
    fdt rm /ocp/i2c@44e0b000/tda19988
    # disable the lcdc and remove its pins
    fdt set /ocp/lcdc@4830e000 status "disabled"
    fdt rm /ocp/l4_wkup@44c00000/scm@210000/pinmux@800/nxp_hdmi_bonelt_pins
    # disable the sound and remove pins as well as clock
    #fdt set /ocp/mcasp@48038000 status "disabled"
    fdt rm /ocp/l4_wkup@44c00000/scm@210000/pinmux@800/mcasp0_pins
    fdt rm /clk_mcasp0_fixed
    fdt rm /clk_mcasp0
    # enable Botic overlay
    # use below line only if you have ES9018 DAC connected to the I2C header of the Hermes-BBB module
    load ${devtype} ${devnum}:${bootpart} 0x88060000 /lib/firmware/BOTIC-SABRE32-00A0.dtbo
    # in case you want to use ES9018K2M based DAC, you can use the following overlay line
    #load ${devtype} ${devnum}:${bootpart} 0x88060000 /lib/firmware/BOTIC-ES9018K2M-00A0.dtbo
    # for a generic I2S DAC connected to the Cronus board you can use the following dummy codec overlay
    #load ${devtype} ${devnum}:${bootpart} 0x88060000 /lib/firmware/BOTIC-00A0.dtbo
    fdt resize ${filesize}
    fdt apply 0x88060000
    
    if load ${devtype} ${devnum}:${bootpart} ${ramdisk_addr_r} /boot/initramfs-linux.img; then
      gpio set 56
      bootz ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r};
    else
      gpio set 56
      bootz ${kernel_addr_r} - ${fdt_addr_r};
    fi;
  fi;
fi
```

Afterwards run `./mkscr` in the `/boot` directory and reboot. Enjoy the boticized kernel and listen to some good music!
