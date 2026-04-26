+++
title = "RTL8127ATF Driver"
date = "2026-04-26T01:34:05-05:00"

description = "Getting RTL8127ATF to work on my linux box"

tags = ["fun", "linux","self-hosting"]

+++

The amazing crab company made this nice card that has both 10Gbps SFP+ port and PCIe 4.0 x1 port.

Theoretically, if you are running kernel 6.16+, `r8169` kernel module already supports this card, but I am still on `6.12.74+deb13+1-amd64` and there are [posts](https://forum.archlinuxcn.org/t/topic/15497) claiming it didn't work as expected.

## How I got it working

The driver can be downloaded via RealTek's [website](https://www.realtek.com/Download/List?cate_id=584), but this does not really work (at least the version). So instead I found this on [github](https://github.com/minisforum-repo/r8127-dkms), at the time of writing they do not have the latest `11.016.00` driver, but the `11.015.00-1` worked fine. 

There's also [r8127-dkms] on [AUR](https://aur.archlinux.org/packages/r8127-dkms)

```fish
wget https://github.com/minisforum-repo/r8127-dkms/releases/download/11.015.00-1/r8127-dkms_11.015.00-1_all.deb
apt install r8127-dkms_11.015.00-1_all.deb
```

some stuff:
1. it does the kernel module signing for you, to enroll your key, check [this post](/homelab-finally/#signing-kernel-modules) out

2. check if the kernel module is loaded/signed: `modinfo r8127 | grep -E 'filename|signer|sig_hashalgo|vermagic'`:

    ```sh
    modinfo r8127 | grep -E 'filename|signer|sig_hashalgo|vermagic'
    filename:       /lib/modules/6.12.74+deb13+1-amd64/updates/dkms/r8127.ko.xz
    vermagic:       6.12.74+deb13+1-amd64 SMP preempt mod_unload modversions
    signer:         DKMS module signing key
    sig_hashalgo:   sha256
    ```

3. check if the driver is being used by the nic: `lspci -k | grep -i "realtek" -A2`

    ```sh
    25:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. Device 8127 (rev 08)
        Subsystem: Realtek Semiconductor Co., Ltd. Device 0123
        Kernel driver in use: r8127
        Kernel modules: r8127
    ```

4. To find the name of the new NIC:  `ls -l /sys/bus/pci/devices/0000:25:00.0/net`

## The official version 

The driver can be downloaded via RealTek's [website](https://www.realtek.com/Download/List?cate_id=584MightaswelltrythenewestversionrightfromRealtekanyway).

If you want to just `wget` it directly, you can use [this repo](https://github.com/openwrt/rtl8127) provided by the openwrt community.

If you want to use the official version, here's how you can manually sign it:

```fish
cd ~/r8127-11.016.00

# compile the module
make clean
make modules

mkdir -p /lib/modules/(uname -r)/kernel/drivers/net/ethernet/realtek
cp src/r8127.ko /lib/modules/(uname -r)/kernel/drivers/net/ethernet/realtek/r8127.ko

# sign the module
/usr/src/linux-headers-(uname -r)/scripts/sign-file \
    sha256 \
    /var/lib/dkms/mok.key \
    /var/lib/dkms/mok.pub \
    /lib/modules/(uname -r)/kernel/drivers/net/ethernet/realtek/r8127.ko

# load it!
depmod -a
modprobe r8127
```

Unfortunately this did not work for me, the card got stuck in `<NO-CARRIER,BROADCAST,MULTICAST,UP>` 

I tried the following and none of them worked:

- `ethtool -s enp37s0 speed 10000 duplex full autoneg on`
- `ethtool -s enp37s0 speed 1000 duplex full autoneg on`
- `ethtool -s enp37s0 autoneg on advertise 0x1000`

## Other interesting stuff

```sh
Settings for enp37s0:
	Supported ports: [ TP ]
	Supported link modes:   1000baseT/Full
	                        10000baseT/Full
	Supported pause frame use: No
	Supports auto-negotiation: No
	Supported FEC modes: Not reported
	Advertised link modes:  1000baseT/Full
	                        10000baseT/Full
	Advertised pause frame use: No
	Advertised auto-negotiation: No
	Advertised FEC modes: Not reported
	Speed: 10000Mb/s
	Duplex: Full
	Auto-negotiation: off
	Port: Twisted Pair
	PHYAD: 0
	Transceiver: internal
	MDI-X: on
	Supports Wake-on: pumbg
	Wake-on: d
        Current message level: 0x00000033 (51)
                               drv probe ifdown ifup
	Link detected: yes
```

notice it says `baseT/Full`, but it is the carb company, im not complaining...

bugtik does recognize it correctly `/interface ethernet monitor sfp-sfpplus1 once`

```sh
                      name: sfp-sfpplus1                                                             
                    status: link-ok                                                                  
          auto-negotiation: done                                                                     
                      rate: 10Gbps                                                                   
               full-duplex: yes                                                                      
           tx-flow-control: no                                                                       
           rx-flow-control: no                                                                       
                       fec: off                                                                      
                 supported: 10M-baseT-half                                                           
                            10M-baseT-full                                                           
                            100M-baseT-half                                                          
                            100M-baseT-full                                                          
                            1G-baseT-half                                                            
                            1G-baseT-full                                                            
                            1G-baseX                                                                 
                            10G-baseT                                                                
                            10G-baseSR-LR                                                            
                            10G-baseCR                                                               
             sfp-supported: 1G-baseX                                                                 
                            10G-baseSR-LR                                                            
               advertising: 1G-baseX                                                                 
                            10G-baseSR-LR                                                            
        sfp-module-present: yes                                                                      
               sfp-rx-loss: no                                                                       
              sfp-tx-fault: no                                                                       
                  sfp-type: SFP/SFP+/SFP28/SFP56                                                     
        sfp-connector-type: LC                                                                       
              sfp-encoding: 64B/66B                                                                  
        sfp-link-length-sm: 10km                                                                     
           sfp-vendor-name: FINISAR CORP.
```

{{< giscus "Mr-Sheep/blog" "MDEwOlJlcG9zaXRvcnkzNDQ4NjQ1MTQ=" "preferred_color_scheme" >}}