

# Cisco ASA Firewall — TFTP-Based OS Restoration & Flash Reset
<p align="center">
  <img src="assets/brandons-logo.png" alt="Brandon's logo" width="140">
</p>

**Physical Cisco infrastructure · ROMMON recovery · TFTP image transfer · Post-recovery verification**

## Overview

<p align="center">
  <img src="assets/flowchart.png" alt="Cisco ASA firewall recovery workflow flow chart" width="100%">
</p>
<p align="center"><em>High-level visual summary of the firewall recovery, sanitization, and TFTP-based operating system restoration process documented in this repository.</em></p>

This project documents the reset and operating system restoration of a physical Cisco ASA firewall. The work involved recovering CLI access, removing the saved configuration, transferring the ASA software image to a TFTP server, erasing and rebuilding internal flash, restoring the image, and reviewing the appliance after boot.

The screenshots were captured during the work through a serial-console session. Hardware serial numbers and MAC addresses have been obscured in the published evidence.

[↑ Back to Table of Contents](#table-of-contents)

---

## Table of Contents

- [Environment](#environment)
  - [Recovery network topology](#recovery-network-topology)
- [01. Console access and ROMMON inspection](#01-console-access-and-rommon-inspection)
- [02. Temporary recovery boot](#02-temporary-recovery-boot)
- [03. Configuration reset and management setup](#03-configuration-reset-and-management-setup)
- [04. Image inventory and TFTP backup](#04-image-inventory-and-tftp-backup)
- [05. Internal flash erasure](#05-internal-flash-erasure)
- [06. TFTP operating system restoration](#06-tftp-operating-system-restoration)
- [07. ROMMON cleanup and normal boot](#07-rommon-cleanup-and-normal-boot)
- [08. Configuration review](#08-configuration-review)
- [09. Software and hardware verification](#09-software-and-hardware-verification)
- [10. Boot image configuration](#10-boot-image-configuration)
- [11. Interface and failover checks](#11-interface-and-failover-checks)
- [Results and scope](#results-and-scope)
- [About the Engineer](#about-the-engineer)

---

## Environment

| Item | Recorded configuration |
| :--- | :--- |
| Platform | Cisco ASA 5585-X, ASA SSP-60; separate FirePOWER SSP module installed |
| Console | Direct serial connection with Tera Term |
| Recovery environment | ROMMON and ASA CLI |
| ASA software image | `asa984-8-smp-k8.bin` |
| Image transfer | TFTP, over a direct recovery network |
| TFTP server | `192.168.1.10` |
| ASA address for transfer | `192.168.1.1/24` |
| Internal storage | `disk0:` |

### Recovery Network Topology

<p align="center">
  <img src="assets/topology.png" alt="Cisco ASA physical recovery network topology showing serial-console management, the TFTP server, temporary IP addressing, and the firmware restoration path" width="100%">
</p>

<p align="center"><em>Physical console access and temporary Ethernet/TFTP image-transfer path used during the ASA recovery.</em></p>

The operation documented here is the restoration of the **ASA software environment**. The installed FirePOWER module was identified during inspection, but its independent recovery is not part of this record.

[↑ Back to Table of Contents](#table-of-contents)

---

## 01. Console access and ROMMON inspection

Accessed the firewall through its physical console and interrupted startup to reach the ROM monitor before changing the boot behavior.

- `BREAK` / `ESC` — interrupted the normal boot sequence to access ROMMON.
- `?` — displayed available ROMMON commands and confirmed the recovery options supported by the device.

![ROMMON session showing platform and available commands](assets/step1.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 02. Temporary recovery boot

Temporarily bypassed the saved startup configuration so the appliance could boot into ASA software for maintenance.

- `confreg` — inspected the configuration register.
- `confreg 0x41` — set the recovery register to bypass the startup configuration on the next boot.
- `boot` — loaded the existing ASA image from internal flash.

![Configuration-register change and recovery boot](assets/step2.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 03. Configuration reset and management setup

After the ASA CLI loaded, removed the saved configuration and prepared a temporary interface configuration for the TFTP connection.

- `enable` / `en` — entered privileged EXEC mode.
- `write erase` — cleared the saved startup configuration.
- `configure terminal` / `conf t` — entered global configuration mode.

![Privileged CLI access and startup configuration reset](assets/step3.png)

- `configure factory-default 192.168.1.1 255.255.255.0` — applied a factory-default configuration with an IP address on the recovery network.
- `end` — returned to privileged EXEC mode.

The initial command syntax was rejected; the subsequent valid factory-default command was accepted.

![Factory-default configuration and recovery network setup](assets/step4.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 04. Image inventory and TFTP backup

Identified the installed ASA image and copied it to the TFTP server before clearing the local filesystem.

- `mode single` — checked the security-context mode; the output reported that the appliance was already in single-context mode.
- `dir all` — inspected flash contents and located the ASA software image.

![ASA image identified in the flash directory](assets/step5.png)

- `ping 192.168.1.10` — verified reachability to the TFTP host.
- `copy disk0: tftp:` — backed up the selected ASA image from local flash to the TFTP server.

An initial source filename was not found. The transfer proceeded after selecting the existing `asa984-8-smp-k8.bin` file.

![TFTP connectivity test and image backup](assets/step6.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 05. Internal flash erasure

Erased and reinitialized internal flash after confirming the image had been transferred off the appliance.

- `erase disk0:` — erased the existing filesystem contents and recreated the flash filesystem.

![Flash erase and filesystem initialization](assets/step7.png)

- `dir disk0:` — verified that the filesystem was accessible and that no files remained after the operation.

![Directory check showing an empty disk0 filesystem](assets/step8.png)

This step confirms a filesystem-level erase. It is not presented as a certified or forensic media-sanitization result.

[↑ Back to Table of Contents](#table-of-contents)

---

## 06. TFTP operating system restoration

Transferred the backed-up ASA operating system image from the TFTP host to the newly initialized flash filesystem.

- `ping 192.168.1.10` — rechecked connectivity to the TFTP host.
- `copy tftp: disk0:` — retrieved `asa984-8-smp-k8.bin` and wrote it to internal flash.

The transfer reported **111,532,032 bytes copied**.

![ASA image restored to disk0 from the TFTP server](assets/step9.png)

- `dir disk0:` — confirmed that the restored image existed on flash before rebooting.

![Restored operating system image present in flash](assets/step10.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 07. ROMMON cleanup and normal boot

Returned to ROMMON after the image transfer to remove temporary network-boot variables and restore normal boot behavior.

- `reload` — restarted the ASA to enter ROMMON again.

![ASA restart and re-entry into ROMMON](assets/step11.png)

- `set` — displayed ROMMON variables, including the network settings and image selection.

![Review of active ROMMON boot variables](assets/step12.png)

- `unset ADDRESS` — cleared the ROMMON IP address.
- `unset GATEWAY` — cleared the ROMMON gateway.
- `unset SERVER` — cleared the ROMMON TFTP server address.
- `unset IMAGE` — removed the network-boot image selection.
- `set` — reviewed the variables after the changes.

An attempt to unset `NETMASK` returned an error on this ROMMON implementation; the remaining variable state was checked using `set`.

![ROMMON network-boot variables cleared and reviewed](assets/step13.png)

- `sync` — saved changes to ROMMON variables.
- `confreg 0x1` — restored the normal configuration-register setting.
- `reset` — rebooted the firewall with the updated boot settings.

![ROMMON synchronization, normal register setting, and reset](assets/step14.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 08. Configuration review

Checked the saved and active configuration after the firewall completed its normal boot.

- `enable` / `en` — returned to privileged EXEC mode.
- `show startup-config` / `show start` — inspected the saved startup configuration for residual customer settings.

![Post-recovery startup configuration review](assets/step15.png)

- `show running-config all` / `show run all` — displayed the active configuration, including defaults, to review the reset state.

![Running configuration inspection](assets/step16.png)

The configuration review shows the appliance's baseline state; it does not by itself prove that all previously stored data is unrecoverable.

[↑ Back to Table of Contents](#table-of-contents)

---

## 09. Software and hardware verification

Confirmed the running ASA software version, the booted image, installed hardware, and basic system-resource status.

- `show version` — checked the ASA release, system image, platform, and configuration register.

The output shows **Cisco ASA Software 9.8(4)8** booted from `disk0:/asa984-8-smp-k8.bin`.

![ASA version and loaded operating system image](assets/step17.png)

- `show inventory` / `show inv` — identified the chassis and installed hardware modules.
- `show cpu usage` — checked CPU utilization after recovery.
- `show memory` — reviewed system memory information.

The hardware output identifies an ASA security services processor and a separate FirePOWER SSP module.

![Chassis inventory, CPU, and memory inspection](assets/step18.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 10. Boot image configuration

Configured the ASA to reference the restored image on internal flash rather than a temporary network boot image.

- `dir all` — confirmed the image was still present on flash.
- `boot system disk0:asa984-8-smp-k8.bin` — selected the local ASA image as the boot target.
- `end` — exited configuration mode.
- `show running-config boot system` — checked the boot system directive in the active configuration.
- `show bootvar` — confirmed the BOOT variable pointed to the restored image.
- `show module` — reviewed the modules detected by the ASA chassis.

![Flash verification, local boot image selection, and module status](assets/step20.png)

The ROMMON recovery settings were explicitly saved with `sync`, and the firewall was restarted using `reset` before successful ASA software and hardware verification (see Steps 07–09). The restored image was subsequently selected as the local boot target, and `show running-config boot system` and `show bootvar` were used to verify the active boot configuration. The final configuration was also saved and the appliance restarted, as confirmed by the engineer performing the recovery. The embedded screenshots document the ROMMON save/restart and post-boot checks; the final ASA configuration-save and subsequent restart are reported as completed work rather than presented as independently visible in the cited screenshot.

[↑ Back to Table of Contents](#table-of-contents)

---

## 11. Interface and failover checks

Reviewed the appliance's final interface and high-availability state without assigning it a production network role.

- `show failover` — checked whether failover was enabled or configured.
- `show interface ip brief` — reviewed interface addresses and administrative/operational status.

The recorded output shows failover off/not configured and external GigabitEthernet ports administratively down.

![Final failover and interface status checks](assets/step21.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## Results and scope

The console record documents completion of the ASA-side recovery workflow:

- CLI access recovered through ROMMON and a temporary configuration-register setting.
- Stored startup configuration removed and resulting configuration reviewed.
- Existing ASA software image copied to a TFTP server before the flash erase.
- Internal flash recreated, verified empty, and repopulated with the ASA image over TFTP.
- ASA software booted and checked against the recorded version and hardware inventory.
- ROMMON variables saved and the appliance restarted successfully; the restored ASA image and hardware state were verified after boot.
- Local boot-image selection and BOOT variable verified; the engineer additionally confirms saving the final ASA configuration and restarting the appliance.
- Interface status and failover state reviewed.

**Scope:** This project covers the ASA operating system on physical Cisco hardware. It does not establish independent FirePOWER module recovery, certified media destruction, or a production firewall cutover.

**Evidence:** All 20 available screenshots are embedded in the relevant steps. Original image filenames are preserved; no `step19.png` was included in the source evidence.

[↑ Back to Table of Contents](#table-of-contents)

---

## About the Engineer

<p align="center">
  <img src="assets/brandons-logo.png" alt="Brandon Stevenson — personal logo" width="120">
</p>

<h3 align="center">Brandon Stevenson</h3>

<p align="center">
  <strong>IT Infrastructure Technician</strong><br>
  Enterprise Infrastructure &nbsp;·&nbsp; Systems Administration &nbsp;·&nbsp; Software Development
</p>

<p align="center">
  <strong>Bachelor of Science — Cloud and Network Engineering (In Progress)</strong><br>
  Western Governors University<br><br>
  <strong>Associate of Applied Science — Software Programming</strong><br>
  College of Southern Nevada
</p>

My background spans hands-on enterprise hardware, IT support, endpoint deployment, and software development. I've worked with device imaging and configuration, Active Directory, Microsoft Intune, Microsoft 365, and infrastructure troubleshooting. I focus on practical problem-solving, careful validation, and clear documentation of completed work.

<p align="center">
  <a href="https://brandons-resume.com/">Portfolio</a>
  &nbsp;&nbsp;·&nbsp;&nbsp;
  <a href="https://www.linkedin.com/in/brandon-in-tech/">LinkedIn</a>
</p>

<p align="center"><sub>Technical documentation and project evidence by Brandon Stevenson.</sub></p>
