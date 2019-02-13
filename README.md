# Linksys EA8500 Bring-Up

This branch includes "snapshot in time" commits of the state of my local development branch as I work towards bringing up a Linksys EA8500 ("Dallas") under OpenWrt (Linux 4.14) and making it generally functional.

This is an **experimental** branch at this time. 

***Breakage, including "bricking" of the device, is possible at any time.***

For some additional details on the Linksys EA8300, see [https://openwrt.org/inbox/linksys/linksys_ea8300](https://openwrt.org/inbox/linksys/linksys_ea8300)

The OpenWrt sources incorporated herein are maintained at [https://git.openwrt.org/openwrt/openwrt.git](https://git.openwrt.org/openwrt/openwrt.git) and are subject to the ownership, licensing, and terms of use of that project. This repo does *not* include the OpenWrt `master` branch.

## Current Status

<span style="font-size:larger;">***Experimental -- Booting initramfs from memory using U-Boot***</span>


### Mainly Operational

* Kernel -- boots and runs SMP from memory (initramfs)
* serial0 -- available, "console" output and login
* NAND -- read confirmed, write not tested
* eth0, eth1 -- available, MAC addresses need work
* Networking -- available, including SSH access
* LEDs -- operational
* USB -- operational, but resets under heavy load

### Limited Functionality

* Switch -- available, re-configuration seems to fail
* Wireless
  * IPQ4019 x2, QCA9888 recognized
  * Firmware download seems to work ("classic" and "CT" ath10k)
    * Automated extraction of config data from 0:ART has "timing" issue
    	* Seems to happen *after* drivers load
    	* Manual inclusion of the three config files required for memory-based boot
  * No AP beacons seen, monitor functionality not yet examined
  * *Note: The second 5 GHz radio, QCA9888 on PCI 0000:01:00.0, is limited to ch. 100 (5.5 GHz) and above by the ART data and the data in the OEM firmware's cal data. This is perhaps due to RF design optimization and/or interoperation with 2.4 GHz, such as the two, shared antennas.*

### Not Functional or Not Seen

* serial1 -- not seen (OEM appears as `ttyQHS0`, different than serial0)
* Bluetooth -- not seen, though no Bluetooth drivers in build

### Untested

* Flashing of image to NAND
* ROM/overlay images (only initramfs tested at this time)


## Reference Data in ./ea8300

`config.seed` of the moment. Includes several "debug" tools.

Output of a locally "decompiled" copy of the device tree from running OEM firmware, v1.1.3.184925, generated with 
```
dtc -I fs -O dts ~/devel/ea8300/2019-02-08/proc/device-tree > OEM-decompiled-device-tree.dts 2> OEM-decompiled-device-tree.dts.stderr
```

## Branch Point off OpenWrt `master`

At this time, this work is based off

```
commit 26fcc937f7e0b8b40297c2d63ae7a17d996f30b1 (tag: ea8300-branch-point, openwrt/master, openwrt/HEAD)
Author: Stijn Tintel <redacted>
Date:   Tue Feb 5 04:34:01 2019 +0200
```
I have yet to decide how to handle any future OpenWrt commits that directly impact this work. 

## TODO

### Functionality

#### High Priority

* Switch configuration
* Primary MAC address extraction from `devinfo` partition
* MAC address generation for additional interfaces
* Get wireless *working* for both IPQ4019 and QCA9888
* "sysupgrade" image generation and testing
* "Factory" image generation and testing
	* Examine OEM firmware to determine if `scripts/linksys-image.sh` is applicable ("footer" from OEM images seem to suggest "close enough" to work)
* Add QCA9888 firmware to default images

#### Lower Priority

* Examine USB reset/hang behavior seen with `nanddump -f /mnt/some.bin /dev/mtd10 2>& 1 | tee /mnt/some.log` (and other large partitions; not seen with `dd` from `/dev/zero`)
* Install Bluetooth drivers and evaluate (see OEM boot-log segment below)
* OEM enables serial1 as `ttyQHS0`
* Examine OEM for additional devices worth implementing
* Determine if hardware RNG and crypto facilities can be / are being used
* Add "panic" LED to DTS
* USB hang/reset under load
* OEM firmware resets USB power and Ethernet in early run-time
	* Where are the regulator or reset controls?
	* Why is this being done?
* Check "key" operation and DTS
* What is `Dakota Chip version 0x1401` (OEM boot log)
* Consider adding U-Boot environment tools and config to image



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

#### Building Image

Configure and build as any other OpenWrt image.

#### Booting Image

Without additional arguments the TFTP server is expected at 192.168.1.254 with a file name of `C0A80101.img`. Default configuration of `loadaddr` of 8400000 is functional.

Copy `bin/targets/ipq40xx/generic/openwrt-ipq40xx-linksys_ea8300-initramfs-fit-zImage.itb` to `C0A80101.img` in your TFTP server's directory (or to the target of a symlink of that name). 

Access the serial console (see the OpenWrt wiki page, referenced near the top of this page) and rrestart the router.

At least for my serial adapter, I can hit [space] before `Hit any key to stop autoboot` appears and it will stop, waiting for commands.

```
Hit any key to stop autoboot:  0 
(IPQ40xx) # tftp
eth0 PHY0 up Speed :1000 Full duplex
eth0 PHY1 Down Speed :10 Half duplex
eth0 PHY2 Down Speed :10 Half duplex
eth0 PHY3 Down Speed :10 Half duplex
eth0 PHY4 Down Speed :10 Half duplex
*** Warning: no boot file name; using 'C0A80101.img'
Using eth0 device
TFTP from server 192.168.1.254; our IP address is 192.168.1.1
Filename 'C0A80101.img'.
Load address: 0x84000000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         ########################################################
done
Bytes transferred = 6533904 (63b310 hex)
(IPQ40xx) # bootm

```

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
