# Hello-world
# Android Booting Shenanigans

## Terminologies

- **rootdir**: the root directory (`/`). All files/folders/filesystems are stored in or mounted under rootdir. On Android, the filesystem may be either `rootfs` or the `system` partition.
- **`initramfs`**: a section in Android's boot image that the Linux kernel will use as `rootfs`. People also use the term **ramdisk** interchangeably
- **`recovery` and `boot` partition**: these 2 are actually very similar: both are Android boot images containing ramdisk and Linux kernel (plus some other stuff). The only difference is that booting `boot` partition will bring us to Android, while `recovery` has a minimalist self contained Linux environment for repairing and upgrading the device.
- **SAR**: System-as-root. That is, the device uses `system` as rootdir instead of `rootfs`
- **A/B, A-only**: For devices supporting [Seamless System Updates](https://source.android.com/devices/tech/ota/ab), it will have 2 slots of all read-only partitions; we call these **A/B devices**. To differentiate, non A/B devices will be called **A-only**
- **2SI**: Two Stage Init. The way Android 10+ boots. More info later.

Here are a few parameters to more precisely define a device's Android version:

- **LV**: Launch Version. The Android version the device is **launched** with. That is, the Android version pre-installed when the device first hit the market.
- **RV**: Running Version. The Android version the device is currently running on.

We will use **Android API level** to represent LV and RV. The mapping between API level and Android versions can be seen in [this table](https://source.android.com/setup/start/build-numbers#platform-code-names-versions-api-levels-and-ndk-releases). For example: Pixel XL is released with Android 7.1, and is running Android 10, these parameters will be `(LV = 25, RV = 29)`

## Boot Methods

Android booting can be roughly categorized into 3 major different methods. We provide a general rule of thumb to determine which method your device is most likely using, with exceptions listed separately.

Method | Initial rootdir | Final rootdir
:---: | --- | ---
**A** | `rootfs` | `rootfs`
**B** | `system` | `system`
**C** | `rootfs` | `system`

- **Method A - Legacy ramdisk**: This is how *all* Android devices used to boot (good old days). The kernel uses `initramfs` as rootdir, and exec `/init` to boot.
	- Devices that does not fall in any of Method B and C's criteria
- **Method B - Legacy SAR**: This method was first seen on Pixel 1. The kernel directly mounts the `system` partition as rootdir and exec `/init` to boot.
	- Devices with `(LV = 28)`
	- Google: Pixel 1 and 2. Pixel 3 and 3a when `(RV = 28)`.
	- OnePlus: 6 - 7
	- Maybe some `(LV < 29)` Android Go devices?
- **Method C - 2SI ramdisk SAR**: This method was first seen on Pixel 3 Android 10 developer preview. The kernel uses `initramfs` as rootdir and exec `/init` in `rootfs`. This `init` is responsible to mount the `system` partition and use it as the new rootdir, then finally exec `/system/bin/init` to boot.
	- Devices with `(LV >= 29)`
	- Devices with `(LV < 28, RV >= 29)`, excluding those that were already using Method B
	- Google: Pixel 3 and 3a with `(RV >= 29)`

### Discussion

From documents online, Google's definition of SAR only considers how the kernel boots the device (**Initial rootdir** in the table above), meaning that only devices using **Method B** is *officially* considered an SAR device from Google's standpoint.

However for Magisk, the real difference lies in what the device ends up using when fully booted (**Final rootdir** in the table above), meaning that **as far as Magisk's concern, both Method B and C is a form of SAR**, but just implemented differently. Every instance of SAR later mentioned in this document will refer to **Magisk's definition** unless specifically says otherwise.

The criteria for Method C is a little complicated, in layman's words: either your device is modern enough to launch with
