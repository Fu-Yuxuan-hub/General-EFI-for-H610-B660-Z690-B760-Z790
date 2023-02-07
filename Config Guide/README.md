# Desktop Alder Lake

| Support               | Version              |
| :-------------------- | :------------------- |
| Initial macOS Support | macOS 12.3, Monterey |

## Starting Point

So making a config.plist may seem hard, it's not. It just takes some time but this guide will tell you how to configure everything, you won't be left in the cold. This also means if you have issues, review your config settings to make sure they're correct. Main things to note with OpenCore:

* **All properties must be defined**, there are no default OpenCore will fall back on so **do not delete sections unless told explicitly so**. If the guide doesn't mention the option, leave it at default.
* **The Sample.plist cannot be used As-Is**, you must configure it to your system
* **DO NOT USE CONFIGURATORS**, these rarely respect OpenCore's configuration and even some like Mackie's will add Clover properties and corrupt plists!

Now with all that, a quick reminder of the tools we need

* [ProperTree](https://github.com/corpnewt/ProperTree)
  * Universal plist editor
* [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)
  * For generating our SMBIOS data

**And read this guide more than once before setting up OpenCore and make sure you have it set up correctly. Do note that images will not always be the most up-to-date so please read the text below them, if nothing's mentioned then leave as default.**

## ACPI

![ACPI](../images/config/config.plist/cometlake/acpi.png)

### Add

::: tip Info

This is where you'll add SSDTs for your system, these are very important to **booting macOS** and have many uses like [USB maps](https://dortania.github.io/OpenCore-Post-Install/usb/), [disabling unsupported GPUs](../extras/spoof.md) and such. And with our system, **it's even required to boot**. Guide on making them found here: [**Getting started with ACPI**](https://dortania.github.io/Getting-Started-With-ACPI/)

For us we'll need a couple of SSDTs:

| Required SSDTs                                               | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| **[SSDT-PLUG-ALT](https://dortania.github.io/Getting-Started-With-ACPI/)** | Adds ACPI `Processor` objects since macOS does not support `Device` objects used on newer boards, see [Getting Started with ACPI](https://dortania.github.io/Getting-Started-With-ACPI/) for more details. |
| **[SSDT-EC-USBX](https://dortania.github.io/Getting-Started-With-ACPI/)** | Fixes both the embedded controller and USB power, see [Getting Started with ACPI](https://dortania.github.io/Getting-Started-With-ACPI/) for more details. |
| **[SSDT-AWAC](https://dortania.github.io/Getting-Started-With-ACPI/)** | This the 300 and 600 series RTC patch, required for all B660 and Z690 boards. The alternative is SSDT-RTC0 or SSDT-AWAC-DISABLE, depending on the ACPI definitions in your DSDT, see [Getting Started with ACPI](https://dortania.github.io/Getting-Started-With-ACPI/) for more details. |
| **[SSDT-SBUS-MCHC](https://dortania.github.io/Getting-Started-With-ACPI/)** | May not be needed for everyone. Fixes AppleSMBus support, see [Getting Started with ACPI](https://dortania.github.io/Getting-Started-With-ACPI/) for more details. |
| **[SSDT-BRG0](https://dortania.github.io/Getting-Started-With-ACPI/)** | Needed on some systems to restore missing devices to your ACPI tables, which fixes DeviceProperties injection issues. See [Getting Started with ACPI](https://dortania.github.io/Getting-Started-With-ACPI) for more details. |
| **[SSDT-USBW](https://dortania.github.io/Getting-Started-With-ACPI/)** | Companion to the USBWakeFixup kext, see [Getting Started with ACPI](https://dortania.github.io/Getting-Started-With-ACPI/) for more details. |
| **SSDT-DMAC**                                                | As on a real MacPro 7,1 : "the DMAC Direct Memory Access Controller provides an interface between the bus and the input-output devices , share the bus with the processor to make the data transfer, speedups the memory operations by bypassing the involvement of the CPU ". |
| **SSDT-HPET**                                                | Patches out IRQ conflicts. Check: [SSDTs: The easy way](https://dortania.github.io/Getting-Started-With-ACPI/ssdt-methods/ssdt-easy.html#running-ssdttime) (SSDTTime > HPET). |
| **SSDT-DTPG**                                                | Implements DTGP method that is needed by other SSDTs. Related to Thunderbolt. |
| **SSDT-DMAR**                                                | Intel i225/226-V Ethernet |

Note that you **should not** add your generated `DSDT.aml` here, it is already in your firmware. So if present, remove the entry for it in your `config.plist` and under EFI/OC/ACPI.

For those wanting a deeper dive into dumping your DSDT, how to make these SSDTs, and compiling them, please see the [**Getting started with ACPI**](https://dortania.github.io/Getting-Started-With-ACPI/) **page.** Compiled SSDTs have a **.aml** extension(Assembled) and will go into the `EFI/OC/ACPI` folder and **must** be specified in your config under `ACPI -> Add` as well.

:::

### Delete

This blocks certain ACPI tables from loading.

Drive Intel i225/226-V with SSDT-DMAR.Apply as needed:

```
TableSignature  OemTableId        TableLength  Comment 
DMAR            45444B32 20202020 0            Drop OEM DMAR Table
```

### Patch(optional)

This section allows us to dynamically modify parts of the ACPI (DSDT, SSDT, etc.) via OpenCore. For us, our patches are handled by our SSDTs. This is a much cleaner solution as this will allow us to boot Windows and other OSes with OpenCore

I see many configurations with various ACPI patches. Other Alder Lake systems use none of these patches. (Reddit does not format the table properly). Apply as needed:

```
TableSignature  OemTableId        TableLength  Find              Replace           Count  Comment 
44534454                          0            4D435F5F          4D434843          0      Change MC__ to MCHC
53534454        4967667853736474  0            4D435F5F          4D434843          0      Change MC__ to MCHC
53534454        475357417070      0            4303141941444247  4303141958444247  1      Change ADBG to XDBG
```

- Enable *Change MC__ to MCHC* and possibly *Change ADBG to XDBG* as shown above, if you encounter relevant ACPI Errors:
  - See: [fix wake from sleep issue on ***Gigabyte*** *Z690* boards](https://www.tonymacx86.com/threads/z690-chipset-and-alder-lake-cpus.316618/page-132#post-2291256).
  - *Change ADBG to XDBG* is related to an [ACPI error](https://www.tonymacx86.com/threads/gigabyte-z690-aero-g-i5-12600k-amd-rx-6800-xt.317179/page-25#post-2291723) on **Gigabyte** Z690 boards.
  - *Change MC__ to MCHC* is also used on **ASUS** Z690 boards.
- *HPET _CRS to XCRS Rename*, *RTC IRQ 8 Patch*, *TIMR IRQ 0 Patch.* Check: [SSDTs: The easy way](https://dortania.github.io/Getting-Started-With-ACPI/ssdt-methods/ssdt-easy.html#running-ssdttime) (SSDTTime > HPET).
- *Fix RTC _STA bug* (seems to be an old fix previously used in Clover which should not be necessary in OpenCore). Try instead: [SSDTs: The easy way](https://dortania.github.io/Getting-Started-With-ACPI/ssdt-methods/ssdt-easy.html#running-ssdttime) (SSDTTime > AWAC)

### Quirks

Settings relating to ACPI, leave everything here as default as we have no use for these quirks.

## Booter

![Booter](../images/config/config-universal/hedt-booter.png)

This section is dedicated to quirks relating to boot.efi patching with OpenRuntime, the replacement for AptioMemoryFix.efi

### MmioWhitelist

This section is allowing devices to be passthrough to macOS that are generally ignored, for us we can ignore this section.

### Quirks

::: tip Info
Settings relating to boot.efi patching and firmware fixes, for us, we need to change the following:

| Quirk                  | Enabled |
| :--------------------- | :------ |
| DevirtualiseMmio       | YES     |
| EnableWriteUnprotector | NO      |
| ProtectUefiServices    | YES     |
| RebuildAppleMemoryMap  | YES     |
| ResizeAppleGpuBars     | -1      |
| SetupVirtualMap        | NO      |
| SyncRuntimePermissions | YES     |
| :::                    |         |

::: details More in-depth Info

* **AvoidRuntimeDefrag**: YES
  * Fixes UEFI runtime services like date, time, NVRAM, power control, etc.
* **DevirtualiseMmio**: YES
  * Reduces Stolen Memory Footprint, expands options for `slide=N` values and very helpful with fixing Memory Allocation issues , requires `ProtectUefiServices` as well for Z490, Z590, and Z690.
* **EnableSafeModeSlide**: YES
  * Enables slide variables to be used in safe mode.
* **EnableWriteUnprotector**: NO
  * This quirk and RebuildAppleMemoryMap can commonly conflict, recommended to enable the latter on newer platforms and disable this entry.
  * However, due to issues with OEMs not using the latest EDKII builds you may find that the above combo will result in early boot failures. This is due to missing the `MEMORY_ATTRIBUTE_TABLE` and such we recommend disabling RebuildAppleMemoryMap and enabling EnableWriteUnprotector. More info on this is covered in the [troubleshooting section](/troubleshooting/extended/kernel-issues.md#stuck-on-eb-log-exitbs-start).
* **ProtectUefiServices**: YES
  * Protects UEFI services from being overridden by the firmware, required for Z490.
* **ProvideCustomSlide**: YES
  * Used for Slide variable calculation. However the necessity of this quirk is determined by `OCABC: Only N/256 slide values are usable!` message in the debug log. If the message `OCABC: All slides are usable! You can disable ProvideCustomSlide!` is present in your log, you can disable `ProvideCustomSlide`.
* **RebuildAppleMemoryMap**: YES
  * Generates Memory Map compatible with macOS, can break on some laptop OEM firmwares so if you receive early boot failures disable this.
* **ResizeAppleGpuBars**: -1
  * Will reduce the size of GPU PCI Bars if set to zero when booting macOS.
  * Setting other PCI Bar values is possible with this quirk, though can cause instabilities
  * This quirk being set to zero is only necessary if Resizable GPU Bar Support is enabled in your firmware.
* **SetupVirtualMap**: NO
  * Fixes SetVirtualAddresses calls to virtual addresses, however broken due to Comet Lake's memory protections. ASUS, Gigabyte and AsRock boards will not boot with this on.
* **SyncRuntimePermissions**: YES
  * Fixes alignment with MAT tables and required to boot Windows and Linux with MAT tables, also recommended for macOS. Mainly relevant for RebuildAppleMemoryMap users.

:::

## DeviceProperties

![DeviceProperties](../images/config/config.plist/cometlake/DeviceProperties.png)

### Add

Sets device properties from a map.

::: tip PciRoot(0x0)/Pci(0x1b,0x0)

`layout-id`

* Applies AppleALC audio injection, you'll need to do your own research on which codec your motherboard has and match it with AppleALC's layout. [AppleALC Supported Codecs](https://github.com/acidanthera/AppleALC/wiki/Supported-codecs).
* You can delete this property outright as it's unused for us at this time

For us, we'll be using the boot-arg `alcid=xxx` instead to accomplish this. `alcid` will override all other layout-IDs present. More info on this is covered in the [Post-Install Page](https://dortania.github.io/OpenCore-Post-Install/)

:::

### Delete

Removes device properties from the map, for us we can ignore this

## Kernel

![Kernel](../images/config/config-universal/kernel-modern-XCPM.png)

### Add

Here's where we specify which kexts to load, in what specific order to load, and what architectures each kext is meant for. By default we recommend leaving what ProperTree has done.

::: The kexts used are essentially the same as the ones used for Comet Lake:

- Lilu.kext (required)
- WhateverGreen.kext (required)
- VirtualSMC.kext (required)
  - SMCProcessor.kext (optional - monitoring CPU temperature)
  - SMCSuperIO.kext (optional - monitoring fan speed)
- AppleALC.kext (usually required - enable audio)
- NVMeFix.kext (optional - for fixing power management and initialization on non-Apple NVMe)

**Other common kexts used on Alder Lake:**

- [RestrictEvents.kext](https://github.com/acidanthera/RestrictEvents) - Lilu Kernel extension for blocking unwanted processes causing compatibility issues on different hardware. - Is needed when enabling E-cores due to large core count and makes showing the proper CPU name possible.
- [CPUFriend.kext ](https://github.com/acidanthera/CPUFriend)- A Lilu plug-in for dynamic power management data injection. Used with CpuFriendDataProvider.kext which can be created according to the instructions here: [CPUFriend/Instructions](https://github.com/acidanthera/CPUFriend/blob/master/Instructions.md)
  - Partial XCPM compatibility is available, but frequency vector tuning will be [required](https://github.com/dortania/bugtracker/issues/190). *(Vit, 22-01-09)*
- An Ethernet kext. Commonly found on Z690:
  - [LucyRTL8125Ethernet.kext](https://github.com/Mieze/LucyRTL8125Ethernet) - A macOS driver for Realtek RTL8125 2.5GBit Ethernet Controllers.
- [USBWakeFixup](https://github.com/osy/USBWakeFixup) is needed to fix keyboard wakeup support, but may cause [compatibility issues](https://github.com/osy/USBWakeFixup/issues/14) with Bluetooth. Works with SSDT-USBW.
- Kexts for USB mapping, depending on the use of [USBMap](https://github.com/corpnewt/USBMap) or [USBToolBox](https://github.com/USBToolBox/tool)

See [Kexts | OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/ktext.html#kexts) for more details.

::: details More in-depth Info

The main thing you need to keep in mind is:

* Load order
  * Remember that any plugins should load *after* its dependencies
  * This means kexts like Lilu **must** come before VirtualSMC, AppleALC, WhateverGreen, etc

A reminder that [ProperTree](https://github.com/corpnewt/ProperTree) users can run **Cmd/Ctrl + Shift + R** to add all their kexts in the correct order without manually typing each kext out.

* **Arch**
  * Architectures supported by this kext
  * Currently supported values are `Any`, `i386` (32-bit), and `x86_64` (64-bit)
* **BundlePath**
  * Name of the kext
  * ex: `Lilu.kext`
* **Enabled**
  * Self-explanatory, either enables or disables the kext
* **ExecutablePath**
  * Path to the actual executable is hidden within the kext, you can see what path your kext has by right-clicking and selecting `Show Package Contents`. Generally, they'll be `Contents/MacOS/Kext` but some have kexts hidden within under `Plugin` folder. Do note that plist only kexts do not need this filled in.
  * ex: `Contents/MacOS/Lilu`
* **MinKernel**
  * Lowest kernel version your kext will be injected into, see below table for possible values
  * ex. `12.00.00` for OS X 10.8
* **MaxKernel**
  * Highest kernel version your kext will be injected into, see below table for possible values
  * ex. `11.99.99` for OS X 10.7
* **PlistPath**
  * Path to the `info.plist` hidden within the kext
  * ex: `Contents/Info.plist`

::: details Kernel Support Table

| OS X Version | MinKernel | MaxKernel |
| :----------- | :-------- | :-------- |
| 10.4         | 8.0.0     | 8.99.99   |
| 10.5         | 9.0.0     | 9.99.99   |
| 10.6         | 10.0.0    | 10.99.99  |
| 10.7         | 11.0.0    | 11.99.99  |
| 10.8         | 12.0.0    | 12.99.99  |
| 10.9         | 13.0.0    | 13.99.99  |
| 10.10        | 14.0.0    | 14.99.99  |
| 10.11        | 15.0.0    | 15.99.99  |
| 10.12        | 16.0.0    | 16.99.99  |
| 10.13        | 17.0.0    | 17.99.99  |
| 10.14        | 18.0.0    | 18.99.99  |
| 10.15        | 19.0.0    | 19.99.99  |
| 11           | 20.0.0    | 20.99.99  |
| 12           | 21.0.0    | 21.99.99  |

:::

### Emulate

::: tip Info
Settings related to CPUID spoofing.

| Quirk      | Enabled                               |
| :--------- | :------------------------------------ |
| Cpuid1Data | `55060A00 00000000 00000000 00000000` |
| Cpuid1Mask | `FFFFFFFF 00000000 00000000 00000000` |
| :::        |                                       |

### Force

Used for loading kexts off system volume, only relevant for older operating systems where certain kexts are not present in the cache(ie. IONetworkingFamily in 10.6).

For us, we can ignore.

### Block

Blocks certain kexts from loading. Not relevant for us.

### Patch

Patches both the kernel and kexts.

::: tip Disable RTC wake scheduling

This patch is present in the OpenCore sample and will be under `Kernel -> Patch -> 5` if you have not deleted the other sample patches.
In this case, you do not have to enter this patch manually, and can instead simply set the `Enabled` key in the patch to true.
If you do not have this patch in your config for whatever reason, add this entry to your config:

| Key         | Type    | Value                                                        |
| :---------- | :------ | :----------------------------------------------------------- |
| Arch        | String  | Any                                                          |
| Base        | String  | __ZN8AppleRTC18setupDateTimeAlarmEPK11RTCDateTime            |
| Comment     | String  | Disable RTC wake scheduling                                  |
| Count       | Number  | 1                                                            |
| Enabled     | Boolean | YES                                                          |
| Find        | Data    | <>                                                           |
| Identifier  | String  | com.apple.driver.AppleRTC                                    |
| Limit       | Number  | 0                                                            |
| Mask        | Data    | <>                                                           |
| MaxKernel   | String  | (this value should be empty, do not add this string to your config) |
| MinKernel   | String  | 19.0.0                                                       |
| Replace     | Data    | C3                                                           |
| ReplaceMask | Data    | <>                                                           |
| Skip        | Number  | 0                                                            |

:::

### Quirks

::: tip Info

Settings relating to the kernel, for us we'll be enabling the following:

| Quirk                   | Enabled | Comment                                                   |
| :---------------------- | :------ | :-------------------------------------------------------- |
| AppleXcpmCfgLock        | YES     | Disable this quirk if `CFG Lock` is disabled in the BIOS. |
| DisableIoMapper         | YES     | Disable this quirk if `VT-d` is disabled in the BIOS.     |
| LapicKernelPanic        | NO      | HP Machines will require this quirk.                      |
| PanicNoKextDump         | YES     |                                                           |
| PowerTimeoutKernelPanic | YES     |                                                           |
| ProvideCurrentCpuInfo   | YES     | Enable E-Core                                             |
| XhciPortLimit           | YES     | Disable if running macOS 11.3+.                           |

:::

::: details More in-depth Info

* **AppleCpuPmCfgLock**: NO
  * Only needed when CFG-Lock can't be disabled in BIOS
  * Only applicable for Ivy Bridge and older
    * Note: Broadwell and older require this when running 10.10 or older
* **AppleXcpmCfgLock**: YES
  * Only needed when CFG-Lock can't be disabled in BIOS
  * Only applicable for Haswell and newer
    * Note: Ivy Bridge-E is also included as it's XCPM capable
* **CustomSMBIOSGuid**: NO
  * Performs GUID patching for UpdateSMBIOSMode set to `Custom`. Usually relevant for Dell laptops
  * Enabling this quirk with UpdateSMBIOSMode Custom mode can also disable SMBIOS injection into "non-Apple" OSes however we do not endorse this method as it breaks Bootcamp compatibility. Use at your own risk
* **DisableIoMapper**: YES
  * Needed to get around VT-D if either unable to disable in BIOS or needed for other operating systems, much better alternative to `dart=0` as SIP can stay on in Catalina
* **DisableLinkeditJettison**: YES
  * Allows Lilu and others to have more reliable performance without `keepsyms=1`
* **DisableRtcChecksum**: NO
  * Prevents AppleRTC from writing to primary checksum (0x58-0x59), required for users who either receive BIOS reset or are sent into Safe mode after reboot/shutdown
* **ExtendBTFeatureFlags** NO
  * Helpful for those having continuity issues with non-Apple/non-Fenvi cards
* **LapicKernelPanic**: NO
  * Disables kernel panic on AP core lapic interrupt, generally needed for HP systems. Clover equivalent is `Kernel LAPIC`
* **LegacyCommpage**: NO
  * Resolves SSSE3 requirement for 64 Bit CPUs in macOS, mainly relevant for 64-Bit Pentium 4 CPUs(ie. Prescott)
* **PanicNoKextDump**: YES
  * Allows for reading kernel panics logs when kernel panics occur
* **PowerTimeoutKernelPanic**: YES
  * Helps fix kernel panics relating to power changes with Apple drivers in macOS Catalina, most notably with digital audio.
* **ProvideCurrentCpuInfo**: YES
  * More patches are required for XNU when using the efficiency cores, though handled automatically by the `ProvideCurrentCpuInfo` quirk starting with OpenCore 0.7.7. *(Vit, 22-01-09)*

* **SetApfsTrimTimeout**: `-1`
  * Sets trim timeout in microseconds for APFS filesystems on SSDs, only applicable for macOS 10.14 and newer with problematic SSDs.
* **XhciPortLimit**: NO
  * This is actually the 15 port limit patch, don't rely on it as it's not a guaranteed solution for fixing USB. Please create a [USB map](https://dortania.github.io/OpenCore-Post-Install/usb/) when possible.
  * With macOS 11.3+, [XhciPortLimit may not function as intended.](https://github.com/dortania/bugtracker/issues/162) We recommend users either disable this quirk and map before upgrading or [map from Windows](https://github.com/USBToolBox/tool). You may also install macOS 11.2.3 or older.

The reason being is that UsbInjectAll reimplements builtin macOS functionality without proper current tuning. It is much cleaner to just describe your ports in a single plist-only kext, which will not waste runtime memory and such

:::

### Scheme

Settings related to legacy booting(ie. 10.4-10.6), for majority you can skip however for those planning to boot legacy OSes you can see below:

::: details More in-depth Info

* **FuzzyMatch**: True
  * Used for ignoring checksums with kernelcache, instead opting for the latest cache available. Can help improve boot performance on many machines in 10.6
* **KernelArch**: x86_64
  * Set the kernel's arch type, you can choose between `Auto`, `i386` (32-bit), and `x86_64` (64-bit).
  * If you're booting older OSes which require a 32-bit kernel(ie. 10.4 and 10.5) we recommend to set this to `Auto` and let macOS decide based on your SMBIOS. See below table for supported values:
    * 10.4-10.5 — `x86_64`, `i386` or `i386-user32`
      * `i386-user32` refers 32-bit userspace, so 32-bit CPUs must use this(or CPUs missing SSSE3)
      * `x86_64` will still have a 32-bit kernelspace however will ensure 64-bit userspace in 10.4/5
    * 10.6 — `i386`, `i386-user32`, or `x86_64`
    * 10.7 — `i386` or `x86_64`
    * 10.8 or newer — `x86_64`

* **KernelCache**: Auto
  * Set kernel cache type, mainly useful for debugging and so we recommend `Auto` for best support

:::

## Misc

![Misc](../images/config/config-universal/misc.png)

### Boot

Settings for boot screen (Leave everything as default).

### Debug

::: tip Info

Helpful for debugging OpenCore boot issues:

| Quirk           | Enabled |
| :-------------- | :------ |
| AppleDebug      | YES     |
| ApplePanic      | YES     |
| DisableWatchDog | YES     |
| Target          | 67      |

:::

::: details More in-depth Info

* **AppleDebug**: YES
  * Enables boot.efi logging, useful for debugging. Note this is only supported on 10.15.4 and newer
* **ApplePanic**: YES
  * Attempts to log kernel panics to disk
* **DisableWatchDog**: YES
  * Disables the UEFI watchdog, can help with early boot issues
* **DisplayLevel**: `2147483650`
  * Shows even more debug information, requires debug version of OpenCore
* **SerialInit**: NO
  * Needed for setting up serial output with OpenCore
* **SysReport**: NO
  * Helpful for debugging such as dumping ACPI tables
  * Note that this is limited to DEBUG versions of OpenCore
* **Target**: `67`
  * Shows more debug information, requires debug version of OpenCore

These values are based of those calculated in [OpenCore debugging](../troubleshooting/debug.md)

:::

### Security

::: tip Info

Security is pretty self-explanatory, **do not skip**. We'll be changing the following:

| Quirk                | Enabled  | Comment                                                      |
| :------------------- | :------- | :----------------------------------------------------------- |
| AllowNvramReset      | YES      |                                                              |
| AllowSetDefault      | YES      |                                                              |
| BlacklistAppleUpdate | YES      |                                                              |
| ScanPolicy           | 0        |                                                              |
| SecureBootModel      | Default  | Leave this as `Default` if running macOS Big Sur or newer. The next page goes into more detail about this setting. |
| Vault                | Optional | This is a word, it is not optional to omit this setting. You will regret it if you don't set it to Optional, note that it is case-sensitive. |

:::

::: details More in-depth Info

* **AllowNvramReset**: YES
  * Allows for NVRAM reset both in the boot picker and when pressing `Cmd+Opt+P+R`
* **AllowSetDefault**: YES
  * Allow `CTRL+Enter` and `CTRL+Index` to set default boot device in the picker
* **ApECID**: 0
  * Used for netting personalized secure-boot identifiers, currently this quirk is unreliable due to a bug in the macOS installer so we highly encourage you to leave this as default.
* **AuthRestart**: NO
  * Enables Authenticated restart for FileVault 2 so password is not required on reboot. Can be considered a security risk so optional
* **BlacklistAppleUpdate**: YES
  * Used for blocking firmware updates, used as extra level of protection as macOS Big Sur no longer uses `run-efi-updater` variable

* **DmgLoading**: Signed
  * Ensures only signed DMGs load
* **ExposeSensitiveData**: `6`
  * Shows more debug information, requires debug version of OpenCore
* **Vault**: `Optional`
  * We won't be dealing vaulting so we can ignore, **you won't boot with this set to Secure**
  * This is a word, it is not optional to omit this setting. You will regret it if you don't set it to `Optional`, note that it is case-sensitive
* **ScanPolicy**: `0`
  * `0` allows you to see all drives available, please refer to [Security](https://dortania.github.io/OpenCore-Post-Install/universal/security.html) section for further details. **Will not boot USB devices with this set to default**
* **SecureBootModel**: Default
  * Controls Apple's secure boot functionality in macOS, please refer to [Security](https://dortania.github.io/OpenCore-Post-Install/universal/security.html) section for further details.
  * Note: Users may find upgrading OpenCore on an already installed system can result in early boot failures. To resolve this, see here: [Stuck on OCB: LoadImage failed - Security Violation](/troubleshooting/extended/kernel-issues.md#stuck-on-ocb-loadimage-failed-security-violation)

:::

### Tools

Used for running OC debugging tools like the shell, ProperTree's snapshot function will add these for you.

### Entries

Used for specifying irregular boot paths that can't be found naturally with OpenCore.

Won't be covered here, see 8.6 of [Configuration.pdf](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.pdf) for more info

## NVRAM

![NVRAM](../images/config/config-universal/nvram.png)

### Add

::: tip 4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14

Used for OpenCore's UI scaling, default will work for us. See in-depth section for more info

:::

::: details More in-depth Info

Booter Path, mainly used for UI Scaling

* **UIScale**:
  * `01`: Standard resolution
  * `02`: HiDPI (generally required for FileVault to function correctly on smaller displays)

* **DefaultBackgroundColor**: Background color used by boot.efi
  * `00000000`: Syrah Black
  * `BFBFBF00`: Light Gray

:::

::: tip 4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102

OpenCore's NVRAM GUID, mainly relevant for CPU name andRTCMemoryFixup users

:::

::: details More in-depth Info

- **Optionally add your CPU name, for example**:



```
revcpuname    String    10-Core Intel i5-12600K
revcpu        Number    1
```

- this is working together with the [acidanthera/RestrictEvents.kext](https://github.com/acidanthera/RestrictEvents/tree/64c9ea31fa62081f8fcc3076ca96d8d39d8c6ca2)

* **rtc-blacklist**: <>
  * To be used in conjunction with RTCMemoryFixup, see here for more info: [Fixing RTC write issues](https://dortania.github.io/OpenCore-Post-Install/misc/rtc.html#finding-our-bad-rtc-region)
  * Most users can ignore this section

:::

::: tip 7C436110-AB2A-4BBB-A880-FE41995C9F82

System Integrity Protection bitmask

* **General Purpose boot-args**:

| boot-args       | Description                                                  |
| :-------------- | :----------------------------------------------------------- |
| **-v**          | This enables verbose mode, which shows all the behind-the-scenes text that scrolls by as you're booting instead of the Apple logo and progress bar. It's invaluable to any Hackintosher, as it gives you an inside look at the boot process, and can help you identify issues, problem kexts, etc. |
| **debug=0x100** | This disables macOS's watchdog which helps prevents a reboot on a kernel panic. That way you can *hopefully* glean some useful info and follow the breadcrumbs to get past the issues. |
| **keepsyms=1**  | This is a companion setting to debug=0x100 that tells the OS to also print the symbols on a kernel panic. That can give some more helpful insight as to what's causing the panic itself. |

* **Networking-specific boot-args**:
| boot-args   | Description                                                  |
| :---------- | :----------------------------------------------------------- |
| **e1000=0** | Disables `com.apple.DriverKit-AppleEthernetE1000` (Apple's DEXT driver) from matching to the Intel I225-V Ethernet controller found on higher end Comet Lake boards, causing Apple's I225 kext driver to load instead.<br/>This boot argument is optional on most boards as they are compatible with the DEXT driver. However, it is required on Gigabyte and several other boards, which can only use the kext driver, as the DEXT driver causes hangs.<br/>You don't need this if your board didn't ship with the I225-V NIC. |

* **GPU-specific boot-args**:

| boot-args          | Description                                                  |
| :----------------- | :----------------------------------------------------------- |
| **agdpmod=pikera** | Used for disabling board ID checks on Navi GPUs (RX 5000 & 6000 series), without this you'll get a black screen. **Don't use if you don't have Navi** (ie. Polaris and Vega cards shouldn't use this) |
| **-wegnoigpu**     | Disable internal GPU, which is not supported.                |

* **Alder Lake-specific boot-args**:

| boot-args   | Description                                                  |
| :---------- | :----------------------------------------------------------- |
| **-ctrsmt** | Used **only with the CpuTopologyRebuild kext** to edit your CPU topology to potentially improve performance related to the use of E-cores. Use at your own risk. |

* **csr-active-config**: `00000000`
  * Settings for 'System Integrity Protection' (SIP). It is generally recommended to change this with `csrutil` via the recovery partition.
  * csr-active-config by default is set to `00000000` which enables System Integrity Protection. You can choose a number of different values but overall we recommend keeping this enabled for best security practices. More info can be found in our troubleshooting page: [Disabling SIP](../troubleshooting/extended/post-issues.md#disabling-sip)

* **run-efi-updater**: `No`
  * This is used to prevent Apple's firmware update packages from installing and breaking boot order; this is important as these firmware updates (meant for Macs) will not work.

* **prev-lang:kbd**: <>
  * Needed for non-latin keyboards in the format of `lang-COUNTRY:keyboard`, recommended to keep blank though you can specify it (**Default in Sample config is Russian**):
  * American: `en-US:0`(`656e2d55533a30` in HEX)
  * Full list can be found in [AppleKeyboardLayouts.txt](https://github.com/acidanthera/OpenCorePkg/blob/master/Utilities/AppleKeyboardLayouts/AppleKeyboardLayouts.txt)
  * Hint: `prev-lang:kbd` can be changed into a String so you can input `en-US:0` directly instead of converting to HEX

| Key           | Type   | Value   |
| :------------ | :----- | :------ |
| prev-lang:kbd | String | en-US:0 |

:::

### Delete

::: tip Info

Forcibly rewrites NVRAM variables, do note that `Add` **will not overwrite** values already present in NVRAM so values like `boot-args` should be left alone. For us, we'll be changing the following:

| Quirk      | Enabled |
| :--------- | :------ |
| WriteFlash | YES     |

:::

::: details More in-depth Info

* **LegacyEnable**: NO
  * Allows for NVRAM to be stored on nvram.plist, needed for systems without native NVRAM

* **LegacyOverwrite**: NO
  * Permits overwriting firmware variables from nvram.plist, only needed for systems without native NVRAM

* **LegacySchema**
  * Used for assigning NVRAM variables, used with LegacyEnable set to YES

* **WriteFlash**: YES
  * Enables writing to flash memory for all added variables.

:::

## PlatformInfo

![PlatformInfo](../images/config/config.plist/haswell/smbios.png)

::: tip Info

For setting up the SMBIOS info, we'll use CorpNewt's [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) application.

For this example, we'll chose the MacPro7,1 SMBIOS. There are two main SMBIOSes used for Alder Lake:

| SMBIOS     | Hardware                            |
| :--------- | :---------------------------------- |
| MacPro7,1  | Use if installing Catalina or later |
| iMacPro1,1 | Use if installing Mojave            |

Run GenSMBIOS, pick option 1 for downloading MacSerial and Option 3 for selecting our SMBIOS.  This will give us an output similar to the following:

```sh
  #######################################################
 #               MacPro7,1 SMBIOS Info                 #
#######################################################

Type:         MacPro7,1
Serial:       F5KZV0JVP7QM
Board Serial: F5K9518024NK3F7JC
SmUUID:       535B897C-55F7-4D65-A8F4-40F4B96ED394
Apple ROM:    001D4F0D5E22
```

* **Note**: MacSerial currently does not support Linux, so you must grab a Windows or macOS machine to generate the values

The `Type` part gets copied to Generic -> SystemProductName.

The `Serial` part gets copied to Generic -> SystemSerialNumber.

The `Board Serial` part gets copied to Generic -> MLB.

The `SmUUID` part gets copied to Generic -> SystemUUID.

The `Apple ROM` part gets copied to Generic -> ROM.

We set Generic -> ROM to either an Apple ROM (dumped from a real Mac), your NIC MAC address, or any random MAC address (could be just 6 random bytes, for this guide we'll use `11223300 0000`. After install follow the [Fixing iServices](https://dortania.github.io/OpenCore-Post-Install/universal/iservices.html) page on how to find your real MAC Address)

**Reminder that you want either an invalid serial or valid serial numbers but those not in use, you want to get a message back like: "Invalid Serial" or "Purchase Date not Validated"**

[Apple Check Coverage page](https://checkcoverage.apple.com)

**Automatic**: YES

* Generates PlatformInfo based on Generic section instead of DataHub, NVRAM, and SMBIOS sections

:::

### Generic

::: details More in-depth Info

* **AdviseFeatures**: NO
  * Used for when the EFI partition isn't first on the Windows drive

* **MaxBIOSVersion**: NO
  * Sets BIOS version to Max to avoid firmware updates in Big Sur+, mainly applicable for genuine Macs.

* **ProcessorType**: `0`
  * Set to `0` for automatic type detection, however this value can be overridden if desired. See [AppleSmBios.h](https://github.com/acidanthera/OpenCorePkg/blob/master/Include/Apple/IndustryStandard/AppleSmBios.h) for possible values

* **SpoofVendor**: YES
  * Swaps vendor field for Acidanthera, generally not safe to use Apple as a vendor in most case

* **SystemMemoryStatus**: Auto
  * Sets whether memory is soldered or not in SMBIOS info, purely cosmetic and so we recommend `Auto`

* **UpdateDataHub**: YES
  * Update Data Hub fields

* **UpdateNVRAM**: YES
  * Update NVRAM fields

* **UpdateSMBIOS**: YES
  * Updates SMBIOS fields

* **UpdateSMBIOSMode**: Create
  * Replace the tables with newly allocated EfiReservedMemoryType, use `Custom` on Dell laptops requiring `CustomSMBIOSGuid` quirk
  * Setting to `Custom` with `CustomSMBIOSGuid` quirk enabled can also disable SMBIOS injection into "non-Apple" OSes however we do not endorse this method as it breaks Bootcamp compatibility. Use at your own risk

:::

## UEFI

![UEFI](../images/config/config-universal/aptio-v-uefi.png)

**ConnectDrivers**: YES

* Forces .efi drivers, change to NO will automatically connect added UEFI drivers. This can make booting slightly faster, but not all drivers connect themselves. E.g. certain file system drivers may not load.

### Drivers

Add your .efi drivers here.

Only drivers present here should be:

* HfsPlus.efi
* OpenRuntime.efi

### APFS

By default, OpenCore only loads APFS drivers from macOS Big Sur and newer. If you are booting macOS Catalina or earlier, you may need to set a new minimum version/date.
Not setting this can result in OpenCore not finding your macOS partition!

macOS Sierra and earlier use HFS instead of APFS. You can skip this section if booting older versions of macOS.

::: tip APFS Versions

Both MinVersion and MinDate need to be set if changing the minimum version.

| macOS Version           | Min Version        | Min Date   |
| :---------------------- | :----------------- | :--------- |
| High Sierra (`10.13.6`) | `748077008000000`  | `20180621` |
| Mojave (`10.14.6`)      | `945275007000000`  | `20190820` |
| Catalina (`10.15.4`)    | `1412101001000000` | `20200306` |
| No restriction          | `-1`               | `-1`       |

:::

### Audio

Related to AudioDxe settings, for us we'll be ignoring(leave as default). This is unrelated to audio support in macOS.

* For further use of AudioDxe and the Audio section, please see the Post Install page: [Add GUI and Boot-chime](https://dortania.github.io/OpenCore-Post-Install/)

### Input

Related to boot.efi keyboard passthrough used for FileVault and Hotkey support, leave everything here as default as we have no use for these quirks. See here for more details: [Security and FileVault](https://dortania.github.io/OpenCore-Post-Install/)

### Output

Relating to OpenCore's visual output, leave everything here as default as we have no use for these quirks.

### ProtocolOverrides

Mainly relevant for Virtual machines, legacy Macs and FileVault users. See here for more details: [Security and FileVault](https://dortania.github.io/OpenCore-Post-Install/)

### Quirks

::: tip Info
Relating to quirks with the UEFI environment, for us we'll be changing the following:

| Quirk            | Enabled | Comment                          |
| :--------------- | :------ | :------------------------------- |
| UnblockFsConnect | NO      | Needed mainly by HP motherboards |

:::

::: details More in-depth Info

* **DisableSecurityPolicy**: NO
  * Disables platform security policy in firmware, recommended for buggy firmwares where disabling Secure Boot does not allow 3rd party firmware drivers to load.
  * If running a Microsoft Surface device, recommended to enable this option

* **RequestBootVarRouting**: YES
  * Redirects AptioMemoryFix from `EFI_GLOBAL_VARIABLE_GUID` to `OC_VENDOR_VARIABLE_GUID`. Needed for when firmware tries to delete boot entries and is recommended to be enabled on all systems for correct update installation, Startup Disk control panel functioning, etc.

* **UnblockFsConnect**: NO
  * Some firmware block partition handles by opening them in By Driver mode, which results in File System protocols being unable to install. Mainly relevant for HP systems when no drives are listed

:::
