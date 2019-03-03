# Linksys EA8500 Bring-Up

This branch includes "snapshot in time" commits of the state of my local development branch as I work towards bringing up a Linksys EA8500 ("Dallas") under OpenWrt (Linux 4.14) and making it generally functional.

This is an **experimental** branch at this time. 

***Breakage, including "bricking" of the device, is possible at any time.***

For some additional details on the Linksys EA8300, see [https://openwrt.org/inbox/linksys/linksys_ea8300](https://openwrt.org/inbox/linksys/linksys_ea8300)

The OpenWrt sources incorporated herein are maintained at [https://git.openwrt.org/openwrt/openwrt.git](https://git.openwrt.org/openwrt/openwrt.git) and are subject to the ownership, licensing, and terms of use of that project. This repo does *not* include the OpenWrt `master` branch.

## Current Status

<span style="font-size:larger;">***Experimental -- Requires U-Boot Environment Changes***</span>


### Mainly Operational

* Kernel
* serial0 and console
* NAND
* eth0, eth1
* Networking -- switch re-configuration has challenges
* Wireless
* LEDs
* USB
* sysupgrade -- some strangeness around next-boot partition
* factory images -- can be loaded through OEM GUI; U-Boot envirinment changes still needed


### Limited Functionality

* Switch -- available, re-configuration seems to fail
* Easy install -- still requires serial access to 

*Note: The second 5 GHz radio, QCA9888 on PCI 0000:01:00.0, is limited to ch. 100 (5.5 GHz) and above by the ART data and the data in the OEM firmware's cal data. This is perhaps due to RF design optimization and/or interoperation with 2.4 GHz, such as the two, shared antennas.*


### Not Functional or Not Seen

* serial1 -- not seen (OEM appears as `ttyQHS0`, different than serial0)
* Bluetooth -- not seen, though no Bluetooth drivers in build



## Reference Data in ./ea8300

`config.seed` of the moment. Includes several "debug" tools.

Output of a locally "decompiled" copy of the device tree from running OEM firmware, v1.1.3.184925, generated with 
```
dtc -I fs -O dts ~/devel/ea8300/2019-02-08/proc/device-tree > OEM-decompiled-device-tree.dts 2> OEM-decompiled-device-tree.dts.stderr
```


## TODO

### Functionality

#### High Priority

* Switch configuration
* Evaluate ART and ART + board approaches
	* Two board files for IPQ4019
	* Board file for QCA9888
	* What about multiple jurisdictions???
* Limit QCA9888 frequencies in DTS, consistent with ART/board
* firstboot doesn't seem to erase the overlay
* Strangeness with not switching boot choice on sysupgrade
  * mtd13 ==> 11, OK
  * mtd11 ==> 13, fails first time, then works
* bootarg overrides


#### Lower Priority

* Tune wireless performance
* Initialize /etc/config/wireless "better" (ch. 100 or higher for QCA9888)
* Install Bluetooth drivers and evaluate (see OEM boot-log segment below)
* OEM enables serial1 as `ttyQHS0`
* Examine OEM for additional devices worth implementing
* Determine if hardware RNG and crypto facilities can be / are being used
* Consider moving ART extraction to early procd
* OEM firmware resets USB power and Ethernet in early run-time
	* Where are the regulator or reset controls?
	* Why is this being done?
* What is `Dakota Chip version 0x1401` (OEM boot log)
* Figure out how to handle multiple jurisdiction's board files
* USB LED trigger
* Default LED triggers
* "f" on console doesn't enter failsafe
* Resolve why "fail-over" boot isn't working (even with OEM firmware present)


### Cleanliness / Style

* Determine if DT can have nodes removed
* Clean up .dts
	* Organization
	* Node references
		* Use existing
		* Add "missing" 
	* Examine warnings on .dtb generation/decompilation and resolve, if appropriae
* Determine appropriate use of "dallas" vs. "ea8300"
	* *As described at [1], please don't use "linksys,ea8300", but "linksys,dallas". The ea6350 should have used linksys,civic as well -- will likely get around poking people about these again.*
		* [1] [https://patchwork.ozlabs.org/patch/731469/](https://patchwork.ozlabs.org/patch/731469/) 

### Crazy Talk

* Custom U-Boot with web interface
* Use "native" extraction of ART data

---

A few "interesting" segments of the OEM boot log

```
Bluetooth: HCI device and connection manager initialized
Bluetooth: HCI socket layer initialized
Bluetooth: L2CAP socket layer initialized
Bluetooth: SCO socket layer initialized
[...]
Bluetooth: HCI UART driver ver 2.2
Bluetooth: HCI H4 protocol initialized
Bluetooth: HCI BCSP protocol initialized
```
  
```
78b0000.uart: ttyQHS0 at MMIO 0x78b0000 (irq = 140, base_baud = 460800) is a MSM HS UART
```
  
---

## Change Log

### 2019-03-02

* IPQ4019 and QCA9888 radios functional
* Primary MAC address extraction from `devinfo` partition
* MAC address generation for additional Ethernet interface
* MAC address generation for wireless interfaces
* Added "panic" LED to DTS
* Added U-Boot environment tools and config to image `/dev/mtd7 0x0 0x40000 0x20000`
* "sysupgrade" image generation and sysupgrade work
* "Factory" image generation and flashing work (needs U-Boot environment at this time)
* Button functionality confirmed for reset and WPS
* Merged changes in `master`

The issues with QCA9888 stability seem to have been resolved upstream around Feb 28 - Mar 1

The previous USB reset/hang behavior seen with `nanddump -f /mnt/some.bin /dev/mtd10 2>& 1 | tee /mnt/some.log` (and other large partitions; not seen with `dd` from `/dev/zero`) appears to have been a media problem. Debian is hanging on `sync` after write of a large file to same media. `badblocks` on that media reports numerous errors.

Based off:

```
commit bc97257ffefd560d7e77fec8c6ac9d3745ea9f11
Author: Daniel Golle <daniel@makrotopia.org>
Date:   Sat Mar 2 19:24:22 2019 +0100
```


### 2019-02-12 -- Inital Revision

Device boots and runs initramfs image from U-Boot `tftp` and `bootm`.

See above for level of functionality and known issues.

See `./EA8300/` for OpenWrt configuration in use.

The ath10k cal files seem to be extracted *after* the drivers try to load. They can be pulled from a booted device and built into the next initramfs build.

```
pre-cal-ahb-a000000.wifi.bin
pre-cal-ahb-a800000.wifi.bin
pre-cal-pci-0000:01:00.0.bin
```

Tested with both at10k and ath10k-ct drivers. Drivers appear to initialize, but not yet seeing APs output on another device's monitor interfaces.

See also [Firmware lacks feature flag indicating a retry limit of > 2 is OK, requested limit: 4](https://forum.openwrt.org/t/ath10k-firmware-lacks-feature-flag/31198?u=jeff) with respect to the ath10k-ct driver/firmware.

Based off:

```
commit 26fcc937f7e0b8b40297c2d63ae7a17d996f30b1
Author: Stijn Tintel <redacted>
Date:   Tue Feb 5 04:34:01 2019 +0200
```


## Building Image

Configure and build as any other OpenWrt image.

## Installing Image

Without additional arguments the TFTP server is expected at 192.168.1.254 with a file name of `C0A80101.img`. Default configuration of `loadaddr` of `8400000` is functional.

Copy `bin/targets/ipq40xx/generic/openwrt-ipq40xx-linksys_ea8300-squashfs-factory.bin` to `C0A80101.img` in your TFTP server's directory (or to the target of a symlink of that name). 

Access the serial console (see the OpenWrt wiki page, referenced near the top of this page) and restart the router, being ready to stop when U-Boot begind to run.

At least for my serial adapter, I can hit [space] once U-Boot starts and before `Hit any key to stop autoboot` appears and it will stop, waiting for commands.

Flash the image to the primary firmware partition with `run flashimg`. Either confirm/adjust `boot_part` to be persisted as `1` or also flash the same image to the secondary firmware partition with `run flashimg2`.

Take note of the current values of `partbootargs` and `partbootargs2` to allow a later return to OEM firmware. Set and persist the U-Boot environment the new bootargs

```
partbootargs=ubi.mtd=11 root=/dev/ubiblock0_0
partbootargs2=ubi.mtd=13 root=/dev/ubiblock0_0
```

Boot into the partitition selected by `boot_part` with `run bootcmd`

Further firmware changes (flip-flop flashing) can be performed using `sysupgrade`, with the caveat of the somewhat puzzling behavior when expecting to transition from mtd11 (primary) to mtd13 (secondary) noted 


## References

This is a "from-scratch" implementation. However, the work that has gone before provides insight into the IPQ4019 and IPQ40xx-based devices. The following references may be of interest:

### Device-Specific

* OEM [GPL drop](http://downloads.linksys.com/downloads/gpl/EA8300_v1.1.3.184925_GPL.tar.gz)
* OEM firmware (`binwalk` and `pip install ubi_reader` helpful)
	* [v1.1.3.184925 -- 2017/11/15](http://downloads.linksys.com/downloads/firmware/FW_EA8300_1.1.3.184925_prod.img)
	* [v1.1.4.191539 -- 2018/11/08](http://downloads.linksys.com/downloads/firmware/FW_EA8300_1.1.4.191539_prod.img)
* `./ea8300/OEM-decompiled-device-tree.dts`
* [http://wiki.dreamrunner.org/public_html/Embedded-System/Qcom-ipq40xx/ipq40xx-device-tree-overview.html](http://wiki.dreamrunner.org/public_html/Embedded-System/Qcom-ipq40xx/ipq40xx-device-tree-overview.html)
* From OpenWrt `master` in `target/linux/ipq40xx/files-4.14/arch/arm/boot/dts/`
	* `qcom-ipq4019-a62.dts`
	* `qcom-ipq4029-mr33.dts`
* From Linux 4.14 and OpenWrt patches
	* `arch/arm/boot/dts/qcom-ipq4019.dtsi`
* From Linux 4.19
	*  `arch/arm/boot/dts/qcom-ipq4019-ap.dk07.1-c1.dts`
	*  `arch/arm/boot/dts/qcom-ipq4019-ap.dk07.1.dtsi`
	*  `arch/arm/boot/dts/qcom-ipq4019.dtsi`
*  OpenWrt [PR #1229](https://github.com/openwrt/openwrt/pull/1229)
*  OpenWrt [PR #1216](https://github.com/openwrt/openwrt/pull/1216)
*  [https://github.com/Bunkerschild/openwrt](https://github.com/Bunkerschild/openwrt)
*  OpenWrt Forum: [Linksys EA8300](https://forum.openwrt.org/t/linksys-ea8300/16906?u-=jeff) thread (many thanks to the contributors there)

### General

#### Device Tree Usage and Conventions
* [https://elinux.org/Device\_Tree\_Reference](https://elinux.org/Device_Tree_Reference)
* [https://elinux.org/Device\_Tree\_Mysteries](https://elinux.org/Device_Tree_Mysteries)
* [https://elinux.org/Device\_Tree\_Source\_Undocumented](https://elinux.org/Device_Tree_Source_Undocumented)
* [https://developer.toradex.com/device-tree-customization](https://developer.toradex.com/device-tree-customization)
* [https://events.static.linuxfound.org/sites/events/files/slides/petazzoni-device-tree-dummies.pdf](https://events.static.linuxfound.org/sites/events/files/slides/petazzoni-device-tree-dummies.pdf)
* Linux binding defintions, in source or online at [https://www.kernel.org/doc/Documentation/devicetree/bindings/](https://www.kernel.org/doc/Documentation/devicetree/bindings/)
* OpenWrt wiki on [Device Tree Usage in OpenWrt](https://openwrt.org/docs/guide-developer/defining-firmware-partitions)
* [https://devicetree-specification.readthedocs.io/en/latest/source-language.html](https://devicetree-specification.readthedocs.io/en/latest/source-language.html)
* [https://github.com/devicetree-org/devicetree-specification/blob/master/source/source-language.rst](https://github.com/devicetree-org/devicetree-specification/blob/master/source/source-language.rst)

#### U-Boot

* [http://processors.wiki.ti.com/index.php/Booting\_Linux\_kernel\_using\_U-Boot](http://processors.wiki.ti.com/index.php/Booting_Linux_kernel_using_U-Boot)
* [https://github.com/pepe2k/u-boot_mod](https://github.com/pepe2k/u-boot_mod)
* [http://www.denx.de/wiki/DULG/Manual](http://www.denx.de/wiki/DULG/Manual)
