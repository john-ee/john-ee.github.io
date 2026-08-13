---
title: "OSDCloud v1 Quickstart"
date: 2025-08-05 16:00:00 +0000
categories: [Windows, Imaging]
tags: [OSDCloud, reset]
draft: false
---
## OSDCloud (OSD Module) — Setup on a New Device

According to osdeploy.com :
> OSDCloud is a Community Tool for deploying Windows 11 (amd64 and arm64) over the internet without using local infrastructure. OSDCloud runs in WinPE using the OSDCloud or the OSD PowerShell Modules.

This guide sets up a fresh Windows 11 machine as an **OSDCloud build host** — the machine you use to create the bootable ISO. This is a one-time setup per build machine.
The bootable ISO will build your Windows Image on the device by :
- Downloading the Windows image directly from Microsoft
- Pulling the drivers from the OEM's website
This means no need to rebuild your ISO regularly but you need a connection to the network during the OSDCloud phase.

> **Module note:** This guide uses the `OSD` module (sometimes called "OSDCloud v1"), which contains the mature `New-OSDCloudTemplate` / `Edit-OSDCloudWinPE` / `New-OSDCloudISO` workflow. Do **not** install the standalone `OSDCloud` or `OSDeploy` modules from the PowerShell Gallery for this workflow — those are a separate, newer preview toolchain with different commands and a hard expiration date. [OSD on PowerShellGallery](https://www.powershellgallery.com/packages/OSD/26.8.1.1)

---

### Prerequisites

- Windows 10/11 or Windows Server, admin rights, internet access
- ~10 GB free disk space for the workspace and ADK
- Powershell terminal opened with elevated access

### 1. Install the Windows ADK + WinPE add-on

Download both from: `https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install`

Run as Administrator, in order:

```powershell
adksetup.exe        # In the feature picker, select ONLY "Deployment Tools"
adkwinpesetup.exe   # WinPE add-on — separate installer, install after the ADK
```

> Match the ADK version to the Windows build you'll be deploying where possible. Mismatches between ADK version and target OS build are the most common cause of template/ISO build failures.

### 2. Install the OSD PowerShell module

```powershell
# Run PowerShell as Administrator
Install-Module -Name OSD -Force -SkipPublisherCheck
Import-Module -Name OSD -Force
```

Verify the right commands are present:

```powershell
Get-Command -Module OSD | Where-Object { $_.Name -match 'OSDCloud' }
```

You should see: `New-OSDCloudTemplate`, `New-OSDCloudWorkspace`, `Edit-OSDCloudWinPE`, `New-OSDCloudISO`, `Start-OSDCloud`, `Start-OSDCloudGUI`.

### 3. Build the base template

Use `-WinRE` instead of plain WinPE — WinRE has minimal but working WiFi support, which plain WinPE does not.

```powershell
New-OSDCloudTemplate -WinRE
```

### 4. Create the workspace

The workspace is where the ISO gets assembled. Re-run this any time you want to start clean (it does not touch the template).

```powershell
New-OSDCloudWorkspace -WorkspacePath C:\OSDCloud
```

### 5. (Recommended) Patch WinPE against Secure Boot certificate revocation - Untested

*Rufus will warn you that the image is not fit for Secure Boot. Claude recommended the following steps but the built ISO worked without issues.*

Modern UEFI firmware may reject older WinPE bootloaders as "revoked" even with a valid signature (KB5025885 / CVE-2023-24932). To avoid needing to disable Secure Boot on every target device:

1. Download the latest **Cumulative Update (.msu)** matching your ADK's Windows version from `https://www.catalog.update.microsoft.com`
2. Rebuild the template with it included:

```powershell
New-OSDCloudTemplate -WinRE -CumulativeUpdate "C:\Path\To\windows11.0-kbXXXXXXX-x64_....msu"
New-OSDCloudWorkspace -WorkspacePath C:\OSDCloud
```

### 6. Important operational notes

- **`Edit-OSDCloudWinPE` is additive only and resets Startup config on every call.** There is no "clear drivers" switch. To start clean, delete and recreate the workspace (`Remove-Item C:\OSDCloud\* -Recurse -Force` then `New-OSDCloudWorkspace` again), then run `Edit-OSDCloudWinPE` **once** with every parameter you need.
- **`New-OSDCloudISO` produces two files**: `OSDCloud.iso` (press-any-key prompt at boot) and `OSDCloud_NoPrompt.iso` (boots straight to WinPE, no keypress). Use the `_NoPrompt` version for a smoother unattended experience.
- **Flashing to USB**: use [Rufus](https://rufus.ie) — select the ISO, GPT partition scheme for UEFI, click Start. Rufus may warn about Secure Boot revocation; either disable Secure Boot on the target device's BIOS temporarily, or apply the cumulative-update patch in step 5 to avoid it.

You're now ready to customize WinPE and build an ISO — see the two follow-up guides:

- **Automated (Zero-Touch) ISO** — silent, no prompts, fixed configuration
- **GUI ISO** — technician picks driver pack, language, edition, etc. at boot

## OSDCloud — Automated (Zero-Touch) ISO

Builds an ISO that requires **no interaction** at boot: it auto-connects, wipes the disk, and installs a fixed OS/edition/language/build with no prompts.

---

### 1. Start from a clean workspace

```powershell
Remove-Item -Path "C:\OSDCloud\*" -Recurse -Force -ErrorAction SilentlyContinue
New-OSDCloudWorkspace -WorkspacePath C:\OSDCloud
```

###23. Customize WinPE — drivers, silent config, branding

Edit the `-StartOSDCloud` parameters to match what you want deployed. This example: Windows 11, 25H2, Enterprise, French, volume-activated, silent wipe, auto-restart.

```powershell
Edit-OSDCloudWinPE `
  -CloudDriver WiFi `
  -StartOSDCloud "-OSVersion 'Windows 11' -OSBuild 25H2 -OSEdition Enterprise -OSLanguage fr-fr -OSActivation Volume -ZTI -Restart" `
  -Brand "YourCompany"

  # If using the manual WiFi staging from step 2, use this instead and add:
  # -StartPSCommand "netsh wlan add profile filename='X:\OSDCloud\WifiProfile.xml'; netsh wlan connect name='YourSSID'"
```

**Parameter notes:**
- `-ZTI` suppresses **all** prompts, including the disk-wipe confirmation. Test on a scratch device first — it erases the drive with no "are you sure."
- `-Restart` reboots into OOBE automatically once imaging completes.
- `-OSActivation Volume` assumes KMS/MAK licensing (typical for Enterprise). Use `Retail` if devices activate via an embedded OEM key.
- Set `-OSLanguage` explicitly — there's no prompt to pick it at runtime.

### 3. (Optional) Include some packages or installers

You can run your own scripts and installers by setting them in the right folder. Run this command : 
```powershell
New-OSDCloudWorkSpaceSetupCompleteTemplate
```

This will create the SetupComplete scripts templates in the folder `C:\OSDCloud\Config\Scripts\SetupComplete`. SetupComplete.cmd calls SetupComplete.ps1

In this scenario, let's pre-install VLC. Download the installer, copy it in the folder `C:\OSDCloud\Config\Scripts\SetupComplete` and add this command to the script SetupComplete.ps1 : 
```powershell
Start-Process "$PSScriptRoot\vlc-3.0.23-win64.exe" -ArgumentList "/S"
```

This process is applicable to the GUI ISO.

### 4. Build the ISO

```powershell
New-OSDCloudISO
```

Use `OSDCloud_NoPrompt.iso` for the smoothest fully-hands-off boot.

**Note:** If you included files in step 5, you can verify there are present with the following commands :
```powershell
# Mount the ISO (returns a drive letter, like a virtual DVD)
$Mount = Mount-DiskImage -ImagePath "C:\OSDCloud\OSDCloud_NoPrompt.iso" -PassThru
$DriveLetter = ($Mount | Get-Volume).DriveLetter
Write-Host "Mounted at ${DriveLetter}:"

# Browse to confirm your SetupComplete folder is actually in there
Get-ChildItem "${DriveLetter}:\OSDCloud\Config\Scripts\SetupComplete" -Recurse -ErrorAction SilentlyContinue

# Once done checking, unmount it
Dismount-DiskImage -ImagePath "C:\OSDCloud-Interactive\OSDCloud_NoPrompt.iso"
```

### 5. Flash to USB

[Rufus](https://rufus.ie) → select USB drive → select `OSDCloud_NoPrompt.iso` → GPT partition scheme (UEFI) → **START**.

### 6. Test before rollout

Boot a scratch/VM device first and confirm: connects to network, wipes disk, installs the correct OS/edition/language, restarts into OOBE cleanly.

![OSDCloud Downloading Windows 11 25H2 directly from Microsoft](/assets/img/2025-08-05-OSDcloud-v1-quickstart/001-automated-build.png)

## OSDCloud — GUI ISO (Technician Picks Options)

Builds an ISO that boots into the `OSDCloudGUI`, letting the technician choose Windows version, build, edition, language, and activation type at deployment time — instead of everything being fixed in advance.

---

### 1. Start from a clean workspace

```powershell
Remove-Item -Path "C:\OSDCloud\*" -Recurse -Force -ErrorAction SilentlyContinue
New-OSDCloudWorkspace -WorkspacePath C:\OSDCloud
```

### 2. Customize WinPE — drivers + GUI launch, no fixed config

```powershell
Edit-OSDCloudWinPE `
  -CloudDriver WiFi `
  -StartOSDCloudGUI `
  -Brand "YourCompany"
```

This gives the technician the **full, unrestricted GUI** at boot: every Windows version/build/edition/language/activation option is selectable, and the correct vendor driver pack is auto-detected and applied based on the device's actual make/model — no vendor flag needed.

### 3. (Optional) Lock some fields while leaving others free

If you want to standardize on certain choices (e.g., always Windows 11 25H2 Enterprise, Volume-activated) but still let the technician freely pick the **language**, drop a config file into the workspace **after** step 2 and **before** building the ISO. Omit any key you want left open in the GUI.

```powershell
New-Item -Path "C:\OSDCloud\Media\OSDCloud\Automate" -ItemType Directory -Force

@'
{
  "OSVersion": ["Windows 11"],
  "OSBuild": ["25H2"],
  "OSEdition": ["Enterprise"],
  "OSActivation": ["Volume"],
  "ZTI": false,
  "Restart": true
}
'@ | Out-File "C:\OSDCloud\Media\OSDCloud\Automate\Start-OSDCloudGUI.json" -Encoding UTF8 -Force
```

> This locks Windows version, build, edition, and activation type, while leaving **language** (and anything else not listed) free for the technician to pick. Remove keys to open up more fields, or add `"OSLanguage": ["fr-fr", "en-us"]` to constrain language to a specific shortlist instead of the full list.
>
> ⚠️ Not independently verified against every OSD module version — after building, boot the ISO once (a VM is fine) and confirm the GUI actually reflects the locked/open fields as expected before relying on it for real deployments.

### 4. Build the ISO

```powershell
New-OSDCloudISO
```

Use `OSDCloud_NoPrompt.iso` to skip the initial "press any key" boot screen — the technician still gets the full `OSDCloudGUI` right after.

### 5. Flash to USB

[Rufus](https://rufus.ie) → select USB drive → select `OSDCloud_NoPrompt.iso` → GPT partition scheme (UEFI) → **START**.

### 6. At deployment time

The technician:
1. Boots the target device from USB
2. Connects to WiFi if no Ethernet is available.
3. Picks language (and any other unlocked fields) in the `OSDCloudGUI`
4. Confirms the disk wipe when prompted (not silent in this GUI mode)
5. Walks away — OS, matching vendor drivers, and language pack install automatically

### Notes

- Since this mode isn't `-ZTI`, the technician **will** see a disk-wipe confirmation — this is expected and a useful safety check for a GUI-driven, human-attended workflow.
- Vendor drivers are matched automatically per-device via SMBIOS detection — a single ISO works across mixed Dell/HP/Lenovo fleets without any per-model configuration.

![OSDCloudGUI](/assets/img/2025-08-05-OSDcloud-v1-quickstart/002-osdgui.png)
