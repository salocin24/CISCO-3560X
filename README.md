# Cisco 3560-X IOS Upgrade & Permanent IP Services License — Lab Report

Nov 17, 2026 · @salocin24

## Objective

Convert a time-limited IP Services evaluation license on a Catalyst 3560-X into a permanent license so that Policy-Based Routing (PBR) survives future reboots.

The switch (WS-C3560X-24T-S, hostname `C3560X`) was running an IP Services **evaluation** license activated in late September 2026 with an 8-week-4-day term, due to expire on or about **24 November 2026**. The evaluation was the only thing enabling PBR, which the lab uses to steer inter-VLAN media traffic. On the first reboot after expiry, the switch would fall back to its permanent IP Base license and PBR would stop working. The goal was a permanent fix rather than a temporary one, without purchasing a new license or new hardware.

## Background: Cisco licensing models

The fix hinged on one distinction that is easy to get wrong: on the 3560-X, Right-To-Use licensing is an **IOS-version feature, not a hardware capability**.

The Catalyst 3560-X / 3750-X family uses a single `universalk9` image that contains all feature sets (LAN Base, IP Base, IP Services); the active level is chosen by license, not by a different image. Three licensing mechanisms are relevant:

- **PAK (Product Activation Key):** a purchased `.lic` file registered to the switch serial. The legitimate paid path; these are no longer sold for this end-of-sale platform.
- **Evaluation:** a built-in time-limited license (8 weeks 4 days on this platform) that unlocks a higher feature set, then expires and reverts on the next reload. This is what the switch was running.
- **Right-To-Use (RTU):** an honor-based model activated from the CLI (`license right-to-use activate ipservices acceptEULA`) that produces a **permanent** license (`Period left: Life time`). RTU is not a paid entitlement — it certifies that the operator has the right to use the feature — but it is the mechanism Cisco built into the IOS, and it survives reboots.

The critical detail: RTU was **added to the 3560-X in the IOS 15.x trains** (introduced around 15.0(2)SE1). The switch was running **12.2(55)SE5**, which predates RTU, which is why the `license right-to-use` command was rejected as invalid. The common misreading — and one made early in this project — is to treat RTU as unavailable on the *platform*; in fact it was unavailable only on that *IOS version*. An upgrade to a 15.x train was therefore the actual fix, not a hardware swap or a network redesign.

## Diagnosis

The switch was running IP Services on a counting-down evaluation license, one week from reverting to IP Base.

Starting state:

| Item | Value |
| --- | --- |
| Model | WS-C3560X-24T-S |
| IOS | 12.2(55)SE5 (`C3560E-UNIVERSALK9-M`) |
| Bootloader | 12.2(58r)SE1 |
| Base license | IP Base, permanent |
| Active license | IP Services, **evaluation**, \~1 week remaining |
| Feature at risk | PBR (route-map `VIA-ASUS` on interface Vlan10) |

PBR on the 3560-X requires the IP Services feature set; IP Base cannot apply `ip policy route-map` to an interface. Because the evaluation clock counts **run-time**, not wall-clock, the \~24 November estimate assumed continuous uptime — `show license` reported the true remaining period. Confirming the platform used the older model, the `license right-to-use` command did not exist on 12.2(55)SE5, and `show license` showed the `ipservices` entry as `License Type: Evaluation` with a decreasing `Period left`.

## Procedure

Upgrade to IOS 15.2(4)E10, then activate the permanent RTU IP Services license. All switch commands were run over console/SSH; the file transfer was pushed from the workstation.

### 1. Enable SCP on the switch

```
conf t
 ip scp server enable
end
```

SCP rides on the existing SSH service — no separate server needed.

### 2. Transfer the image (workstation → switch)

The target image was `c3560e-universalk9-mz.152-4.E10.bin` (26,771,456 bytes). Because the switch runs legacy SSH on 12.2 IOS, the `scp` client needs `-O` (force the legacy SCP protocol, since old IOS has no SFTP subsystem) plus the legacy KEX/cipher/MAC/host-key exceptions, and an explicit destination **filename**:

```
scp -O -o KexAlgorithms=+diffie-hellman-group1-sha1 -o HostKeyAlgorithms=+ssh-rsa -o Ciphers=+aes128-cbc -o MACs=+hmac-sha1 c3560e-universalk9-mz.152-4.E10.bin col@192.168.50.3:flash:/c3560e-universalk9-mz.152-4.E10.bin
```

### 3. Verify integrity

```
verify /md5 flash:c3560e-universalk9-mz.152-4.E10.bin
```

Computed MD5: `1e363a4f136c745fa9d069924320d85c`. The file size matched the source index exactly and the MD5 computed cleanly; the authoritative Cisco hash sits behind a service-contract login, so provenance rested on the exact size match plus the SSH-integrity-protected transfer. (See Takeaways on the limits of this.)

### 4. Set the boot variable

This IOS holds only one `boot system` entry (a second replaces the first), so a chained fallback was not possible; the old image was kept in flash as a manual ROMMON fallback instead.

```
conf t
 no boot system
 boot system flash:c3560e-universalk9-mz.152-4.E10.bin
end
copy run start
```

### 5. Reload and upgrade

```
reload
```

First boot unpacked the image and **reflashed the front-end microcode** (`Front-end Microcode IMG MGR: HW image is upgraded. MCU reset causes the switch to reload`), then auto-reloaded a second time. The bootloader 12.2(58r)SE1 was already sufficient and did not change. This step can take \~15 minutes and must not be interrupted.

### 6. Activate the permanent RTU license

```
license right-to-use activate ipservices acceptEULA
copy run start
reload
```

The command returned `Activated Permanent Right-To-Use ipservices license` / `Next Reboot level is ipservices`. The final reload booted the switch into IP Services on the permanent license.

## Verification and results

The switch booted into 15.2(4)E10 with a permanent IP Services license and PBR intact.

**IOS version** (`show version | include Version`):

```
Cisco IOS Software, C3560E Software (C3560E-UNIVERSALK9-M), Version 15.2(4)E10, RELEASE SOFTWARE (fc2)
BOOTLDR: C3560E Boot Loader (C3560X-HBOOT-M) Version 12.2(58r)SE1, RELEASE SOFTWARE (fc1)
```

**License** (`show license`):

```
Index 1 Feature: ipservices
        Period left: Life time
        License Type: PermanentRightToUse
        License State: Active, In Use
        License Priority: High
Index 2 Feature: ipbase
        Period left: Life time
        License Type: Permanent
        License State: Active, Not in Use
```

The `ipservices` line now reads `PermanentRightToUse` / `Life time` / `Active, In Use` — no evaluation countdown. This is the result that solves the original problem.

**PBR** (`show route-map`, `show run interface vlan10`):

```
route-map VIA-ASUS, permit, sequence 10
  Match clauses:
    ip address (access-lists): VLAN10-TO-LAN50
  Set clauses:
    ip next-hop 192.168.50.1
!
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 ip access-group LAN50-TO-VLAN10 out
 ip policy route-map VIA-ASUS
```

The route-map is applied to Vlan10 and functional on a license that no longer expires. The policy match counter reads zero immediately post-reboot and increments as matching traffic flows (verified by exercising the media services the policy steers).

## Takeaways

- **Version vs. platform.** The most important lesson: a rejected command (`license right-to-use`) meant the *IOS version* lacked the feature, not that the *hardware* did. Early in the project this was misread as a platform limitation, which pointed toward a redesign or a replacement switch. The real fix — an in-place IOS upgrade — was both cheaper and simpler. When a feature seems missing, check whether a newer software release on the same box adds it before changing hardware or topology.
- **Evaluation licenses count run-time, not calendar time.** The expiry estimate depended on uptime; `show license` is the authoritative source for time remaining.
- **Image integrity without the vendor hash.** Cisco's authoritative MD5/SHA512 sits behind a service-contract login. In its absence, confidence came from an exact file-size match, the SSH-integrity-protected transfer, a clean on-device MD5 computation, and a successful signed boot. Cisco images carry an embedded digital signature; where available, `verify flash:<image>` checks that signature and is stronger than matching a hash copied from an untrusted mirror. This remains the weakest link and is worth closing with the official hash via a legitimate Cisco download account.
- **Licensing compliance.** RTU is an honor-based model: it certifies a right to use, it is not a purchased entitlement. For a lab/portfolio writeup, the honest framing is "activated IP Services via Cisco's RTU honor-system licensing," not "obtained a permanent license."
- **Keep a fallback.** The prior 12.2(55)SE5 image was left in flash throughout, giving a one-command ROMMON recovery path. Console access during the reload was essential, since the microcode reflash and double-reboot are invisible over SSH.
- **Side benefit.** Moving off the 12.2(58)SE-era base to 15.2(4)E10 also brings modern SSH algorithm support, which can retire the client-side legacy-algorithm workaround used to reach the switch.
