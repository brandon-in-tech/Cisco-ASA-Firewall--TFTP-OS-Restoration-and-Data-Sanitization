<p align="center">
  <img src="assets/brandons-logo.png" alt="Brandon's logo" width="145" />
</p>Cisco ASA Firewall — TFTP OS Restoration & Flash Sanitization
Physical infrastructure | ROMMON recovery | TFTP image transfer | ASA boot validation

This project documents the recovery and reset of a physical Cisco ASA firewall. Work included bypassing the existing startup configuration, removing stored configuration data, backing up and restoring the ASA operating system image through a TFTP server, rebuilding the internal flash filesystem, and confirming the appliance's post-recovery state.

The screenshots are taken from the serial console during the actual work. Identifying hardware values in the published evidence have been redacted.
Contents
1. Physical console access and ROMMON inspection
2. ROMMON access and temporary boot configuration
3. Configuration reset and management setup
4. Image inventory and TFTP backup
5. Flash erasure and verification
6. TFTP operating system restoration
7. ROMMON cleanup and normal boot
8. Configuration review
9. Software and hardware validation
10. Persistent boot image configuration
11. Final interface and failover checks
12. Result and scope
Equipment and environment
Component	Recorded environment
Hardware	Cisco ASA 5585-X with ASA SSP-60; FirePOWER SSP module also present
Access	Direct serial-console session using Tera Term
Recovery interface	Cisco ROMMON and ASA CLI
Image	asa984-8-smp-k8.bin
Transfer method	TFTP over an isolated management connection
TFTP server	192.168.1.10
ASA management IP used during transfer	192.168.1.1/24
Target filesystem	disk0:


The recovery is documented as an ASA operating system restoration. The separately installed FirePOWER module is identified in hardware output but is not shown being independently reimaged.

<a id="01-console-access"></a>
01 — Physical console access and ROMMON inspection
The appliance was accessed through its physical console. Boot was interrupted to enter ROMMON and inspect the available recovery functions before changing the boot configuration.
- BREAK / ESC — interrupts automatic boot so ROMMON commands can be entered.
- ? — displays ROMMON's supported variables and commands; useful for confirming the capabilities of the installed boot monitor.

Back to contents

<a id="02-recovery-boot"></a>
02 — ROMMON access and temporary boot configuration
The configuration register was temporarily changed so the ASA could start without applying the existing startup configuration.
- confreg — checks the active configuration-register value.
- confreg 0x41 — sets the temporary recovery register, allowing the existing startup configuration to be bypassed.
- boot — starts the ASA software image already available in internal flash.

Back to contents

<a id="03-configuration-reset"></a>
03 — Configuration reset and management setup
Once the ASA CLI became available, the stored startup configuration was cleared and a temporary management configuration established for the TFTP transfer.
- enable (en) — enters privileged EXEC mode.
- write erase — removes the saved startup configuration from nonvolatile storage.
- configure terminal (conf t) — enters global configuration mode.
- configure factory-default 192.168.1.1 255.255.255.0 — applies a baseline configuration and assigns the management network used for the recovery connection.
- end — exits configuration mode.
An initial syntax attempt was rejected before the valid factory-default command was accepted. The CLI output shows the resulting management configuration being applied.

Back to contents

<a id="04-image-backup"></a>
04 — Image inventory and TFTP backup
The internal filesystem was inspected to locate the ASA image before erasure. Network reachability to the TFTP host was verified, and the image was copied off the appliance.
- mode single — checks/requests single-context mode; the output indicates the firewall was already in that mode.
- dir all — lists the files in internal flash, including ASA images, ASDM files, and older system artifacts.
- ping 192.168.1.10 — confirms IP connectivity to the TFTP server before a transfer.
- copy disk0: tftp: — copies a selected ASA image from internal flash to the TFTP server for recovery use.
The first copy request referenced an image that was not present. The source filename was corrected to asa984-8-smp-k8.bin, after which the transfer completed.

Back to contents

<a id="05-flash-erasure"></a>
05 — Flash erasure and verification
With the ASA image transferred to the TFTP host, the internal flash filesystem was erased and rebuilt. The directory was then checked before restoring the image.
- erase disk0: — erases and recreates the ASA internal flash filesystem. This removes the files stored there, including any resident ASA image.
- dir disk0: — verifies that the filesystem is accessible and contains no files after the erase operation.
The console shows the filesystem initialization finishing and dir disk0: returning no files in directory.

Back to contents

<a id="06-os-restoration"></a>
06 — TFTP operating system restoration
The saved ASA image was transferred back to internal flash, followed by a directory check to confirm its presence.
- ping 192.168.1.10 — rechecks reachability after the filesystem operation.
- copy tftp: disk0: — downloads asa984-8-smp-k8.bin from the TFTP server and writes it to the ASA flash filesystem.
- dir disk0: — confirms the image file exists on disk0: after the transfer.
The console reports a completed transfer of 111,532,032 bytes, and the restored image appears in the flash directory.

Back to contents

<a id="07-rommon-cleanup"></a>
07 — ROMMON cleanup and normal boot
After the image was restored, the appliance was restarted into ROMMON to remove temporary network-boot settings and return the configuration register to its normal value.
- set — displays current ROMMON variables for review.
- unset ADDRESS — clears a previously defined ROMMON local IP setting.
- unset GATEWAY — clears a previously defined ROMMON gateway.
- unset SERVER — clears the ROMMON TFTP server setting.
- unset IMAGE — clears the temporary network-boot image selection.
- sync — saves ROMMON variable changes to nonvolatile storage.
- confreg 0x1 — restores the configuration register to the normal boot setting.
- reset — restarts the appliance with the updated ROMMON settings.
A NETMASK unset attempt was rejected by this ROMMON implementation; the final set output was used to review the resulting variables. Clearing ROMMON variables is separate from configuring the ASA's persistent boot system directive, which is completed later.

Back to contents

<a id="08-configuration-review"></a>
08 — Configuration review
After the normal boot, startup and running configuration views were checked for residual configuration and to establish the appliance's post-reset state.
- enable (en) — returns to privileged EXEC mode.
- show startup-config (show start) — displays the startup configuration currently stored by the appliance.
- show running-config all (show run all) — displays the active configuration, including default settings, for review.
The screenshots show a baseline configuration rather than a customer-specific deployment. This check supports configuration review; it does not by itself prove forensic eradication of all recoverable data.

Back to contents

<a id="09-platform-validation"></a>
09 — Software and hardware validation
The running ASA version and installed hardware were checked, followed by basic resource checks.
- show version — verifies the running ASA software, loaded image, platform, and configuration-register state.
- show inventory (show inv) — identifies installed chassis and module hardware.
- show cpu usage — checks CPU utilization after boot.
- show memory — reviews available and used system memory.
The output identifies Cisco ASA Software 9.8(4)8 running from disk0:/asa984-8-smp-k8.bin. Inventory output also identifies the ASA security-services processor and the separate FirePOWER SSP module.

Back to contents

<a id="10-boot-image"></a>
10 — Persistent boot image configuration
The correct operating system image was explicitly registered as the ASA's persistent boot target and verified through CLI output.
- dir all — confirms that the expected ASA image is present on flash before setting a boot path.
- boot system disk0:asa984-8-smp-k8.bin — sets the local ASA image as the selected boot file.
- end — exits configuration mode.
- show running-config boot system — confirms the boot system command appears in the running configuration.
- show bootvar — displays the current BOOT variable pointing to the restored image.
- show module — checks the modules reported by the ASA chassis.
The screenshot confirms the BOOT variable points to disk0:/asa984-8-smp-k8.bin. The record does not show a subsequent write memory or another reboot following this final boot-system command, so persistence across an additional power cycle is not claimed.

Back to contents

<a id="11-final-checks"></a>
11 — Final interface and failover checks
The final inspection confirmed the basic network interface and high-availability state following restoration.
- show failover — checks whether failover is enabled or configured.
- show interface ip brief — provides a condensed view of interface addressing and administrative/operational status.
The output shows failover off/not configured and the external GigabitEthernet interfaces administratively down. That is consistent with an appliance returned to a baseline state; it is not evidence of a production traffic cutover.

Back to contents

<a id="result-and-scope"></a>
Result and scope
The captured work demonstrates:
- Recovery access through ROMMON and a temporary configuration-register change.
- Startup-configuration removal and review of the resulting baseline.
- Backup and restoration of the ASA system image over TFTP.
- Flash filesystem erasure, empty-directory confirmation, and image verification after transfer.
- Successful ASA boot with software version and installed hardware checks.
- Explicit local boot-image configuration and final interface/failover inspection.
Scope boundary: This record documents the ASA software environment on the physical firewall. It does not document independent FirePOWER module recovery, a certified data-destruction process, or a full production firewall redeployment. The flash erase shown is a filesystem operation, not a claim of forensic secure erasure.

Evidence is captured from the physical console. MAC addresses and serial numbers have been obscured in the published screenshots.

Back to contents
