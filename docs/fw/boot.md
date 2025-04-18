---
title: Apple Silicon Boot Flow
summary:
  Boot flow used by Apple Silicon devices, from SoC-integrated ROM to user code
---

Apple Silicon devices seem to follow a boot flow very similar to modern iOS devices.

# Stage 0 (SecureROM)

This stage is located in the boot [ROM](../project/glossary.md#r). Among others, it verifies, loads and executes normal stage 1 from [NOR](../project/glossary.md#n). If this fails, it falls back to [DFU](../project/glossary.md#d) and wait for an [iBSS](../project/glossary.md#i) loader to be sent, before continuing with the [DFU](../project/glossary.md#d) flow at stage 1.

# Normal flow

## Stage 1 (LLB/iBoot1)

This stage is the primary early loader, located in the on-board [NOR](../project/glossary.md#n). This boot stage very roughly goes as follows:

* Read the `boot-volume` variable from [NVRAM](../project/glossary.md#n): its format is `<gpt-partition-type-uuid>:<gpt-partition-uuid>:<volume-group-uuid>`. Other related variables seem to be `update-volume` and `upgrade-boot-volume`, possibly selected by metadata inside the `boot-info-payload` variable;
* Get the local policy hash:
  - First try the local proposed hash ([SEP](../project/glossary.md#s) command 11);
  - If that is not available, get the local blessed hash ([SEP](../project/glossary.md#s) command 14)
* Read the local boot policy, located on the iSCPreboot partition at `/<volume-group-uuid>/LocalPolicy/<policy-hash>.img4`. This boot policy has the following specific metadata keys:
  - `vuid`: UUID: Volume group UUID - same as above
  - `kuid`: UUID: KEK group UUID
  - `lpnh`: SHA384: Local policy nonce hash
  - `rpnh`: SHA384: Remote policy nonce hash
  - `nsih`: SHA384: Next-stage IMG4 hash
  - `coih`: SHA384: fuOS (custom kernelcache) IMG4 hash
  - `auxp`: SHA384: Auxiliary user-authorized kernel extensions hash
  - `auxi`: SHA384: Auxiliary kernel cache IMG4 hash
  - `auxr`: SHA384: Auxiliary kernel extension recept hash
  - `prot`: SHA384: Paired Recovery manifest hash
  - `lobo`: bool: Local boot policy
  - `smb0`: bool: Reduced security enabled
  - `smb1`: bool: Permissive security enabled
  - `smb2`: bool: Third-party kernel extensions enabled
  - `smb3`: bool: Manual mobile device management (MDM) enrollment
  - `smb4`: bool?: MDM device enrollment program disabled
  - `sip0`: u16: SIP customized
  - `sip1`: bool: Signed system volume (`csrutil authenticated-boot`) disabled
  - `sip2`: bool: CTRR ([configurable text region read-only](https://keith.github.io/xcode-man-pages/bputil.1.html)) disabled
  - `sip3`: bool: `boot-args` filtering disabled

  And optionally the following linked manifests, each located at `/<volume-group-uuid>/LocalPolicy/<policy-hash>.<id>.im4m`
  - `auxk`: AuxKC (third party kext) manifest
  - `fuos`: fuOS (custom kernelcache) manifest

* If loading the next stage:

  - The boot directory is located at the target partition Preboot subvolume, at path `/<volume-uuid>/boot/<local-policy.metadata.nsih>`;
  - Decrypt, verify and execute `<boot-dir>/usr/standalone/firmware/iBoot.img4` with the device tree and other firmware files in the same directory. No evidence towards other metadata descriptors yet.

* If loading a custom stage ([fuOS](../project/glossary.md#f)):

  - ...

If it fails at any point during this, it will either error out or fall back to [DFU](../project/glossary.md#d), waiting for an iBEC loader to be sent, before continuing with the [DFU](../project/glossary.md#d) flow at stage 2.

## Stage 2 (iBoot2)

This stage is the OS-level loader, located inside the OS partition and shipped as part of macOS. It loads the rest of the system.

# [DFU](../project/glossary.md#d) flow

## Stage 1 (iBSS)

This stage is sent to the device by the "reviving" host. It bootstraps, verifies and runs the second stage, iBEC.

## Stage 2 (iBEC)

# Modes

Once booted, the [AP](../project/glossary.md#a) can be in one of several boot modes, as confirmed by the [SEP](../project/glossary.md#s):

|  ID | Name                                      |
|----:|-------------------------------------------|
|   0 | macOS                                     |
|   1 | 1TR ("one true" recoveryOS)               |
|   2 | recoveryOS ("ordinary" recoveryOS)        |
|   3 | kcOS                                      |
|   4 | restoreOS                                 |
| 255 | unknown                                   |

The [SEP](../project/glossary.md#s) only allows execution of certain commands (such as editing the boot policy) in [1TR](../project/glossary.md#1), or it will fail with error 11, "AP boot mode".
