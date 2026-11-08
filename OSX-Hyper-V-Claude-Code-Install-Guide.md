# macOS on Hyper-V --- Claude Code Installation Runbook

**Repository:**
[Coastrnaut/OSX-Hyper-V](https://github.com/Coastrnaut/OSX-Hyper-V)\
**Host target:** Windows 11 with Hyper-V\
**Automation assistant:** Claude Code\
**Document purpose:** A staged, safety-conscious procedure for reviewing
the repository, building its OpenCore EFI, creating a Hyper-V VM, and
completing macOS installation.

> **Important legal and platform note**
>
> Apple's macOS software license generally restricts virtualization to
> Apple-branded hardware and imposes additional conditions. A Windows PC
> that is not Apple-branded may not be an authorized host. Review the
> current macOS license and applicable law before proceeding. This
> runbook does not override those restrictions.
>
> This repository is a Hackintosh/virtualization project, not an
> Apple-supported Hyper-V product. Hardware compatibility, boot success,
> graphics, audio, and updates are not guaranteed.

------------------------------------------------------------------------

## 1. How to use this runbook

The safest workflow is to let Claude Code inspect and explain each stage
before it executes anything that changes the host or creates a VM.

Use the following gates:

1.  **Read-only assessment** --- Claude inspects the checked-out
    repository and host prerequisites.
2.  **EFI preparation** --- Claude proposes and then builds the EFI
    after approval.
3.  **Hyper-V preparation** --- Claude reviews the VM creation script
    and presents exact changes.
4.  **VM creation** --- You approve the exact VM configuration and
    commands.
5.  **macOS installation** --- You operate the installer in the Hyper-V
    console, with Claude assisting.
6.  **Post-install validation** --- Claude helps verify the VM and
    explains optional project scripts.

Do not give Claude blanket permission to run arbitrary scripts as
Administrator. Do not disable Windows security controls globally just to
make a script run. Inspect scripts before executing them, and approve
privileged operations individually.

## 2. Repository and project overview

The repository describes itself as an OpenCore configuration for running
macOS on Windows Hyper-V. It builds on:

-   OpenCore --- bootloader and configuration.
-   MacHyperVSupport --- Hyper-V-specific macOS support.
-   OCE-Build --- EFI build/versioning workflow.
-   PowerShell scripts --- EFI build, AMD patch generation, and VM
    creation.

Repository:

``` text
https://github.com/Coastrnaut/OSX-Hyper-V
```

The upstream project README currently documents Sequoia, Sonoma,
Ventura, and earlier macOS versions as supported; Tahoe is listed as in
progress. Confirm the current README and release notes before choosing a
version because support can change.

### Known limitations from the project documentation

-   No hardware graphics acceleration in the default configuration.
-   The synthetic framebuffer provides basic display support; the README
    describes an 8 MB video-memory configuration and a default 1024×768
    resolution.
-   Hyper-V does not expose an audio device to macOS by default.
-   GPU passthrough via Discrete Device Assignment (DDA) is limited to
    supported Windows Server / Hyper-V Server configurations; Windows 11
    Pro and Enterprise do not support DDA.
-   AMD and newer Intel CPUs may require CPU-specific OpenCore
    configuration and patches.
-   Installer boot can require 6--8 GB of VM RAM for Big Sur and newer.
-   macOS updates may break compatibility; take a VM backup/checkpoint
    before attempting updates.

These constraints make this setup more suitable for experimentation and
basic compatibility testing than graphics-intensive workloads.

------------------------------------------------------------------------

## 3. Prerequisites

### Windows host

-   Windows 11 edition that supports the Hyper-V role. Verify your
    edition and current Microsoft documentation before attempting
    installation. Windows 11 Home is not an officially supported Hyper-V
    host edition.
-   CPU with hardware virtualization support enabled in UEFI/BIOS.
-   Sufficient RAM. The project README recommends 8 GB for Big Sur and
    newer; 12--16 GB assigned to the VM is a more comfortable starting
    point when the host has enough memory.
-   Adequate free storage for the EFI disk, macOS virtual disk,
    installer/recovery files, and backups.
-   Hyper-V Manager and Hyper-V PowerShell tools.
-   Git for Windows.
-   PowerShell 7 or Windows PowerShell, as required by the repository
    scripts.
-   Python 3 if required by repository build tooling or recovery-image
    tooling.
-   Claude Code installed and authenticated.

### Check host information

Run these commands in a normal PowerShell window:

``` powershell
Get-ComputerInfo |
    Select-Object WindowsProductName, WindowsVersion, OsBuildNumber,
                  CsManufacturer, CsModel, CsTotalPhysicalMemory

Get-CimInstance Win32_Processor |
    Select-Object Name, Manufacturer, NumberOfCores,
                  NumberOfLogicalProcessors, VirtualizationFirmwareEnabled,
                  VMMonitorModeExtensions

Get-Volume |
    Sort-Object DriveLetter |
    Select-Object DriveLetter, FileSystemLabel, FileSystem,
                  SizeRemaining, Size
```

Record:

-   Exact CPU model and manufacturer.
-   Physical core count.
-   Windows edition and build.
-   Total RAM.
-   Available storage.
-   Whether virtualization is enabled in firmware.

If `VirtualizationFirmwareEnabled` is false, enable Intel VT-x / AMD-V
(SVM) in firmware before proceeding. Menu names vary by motherboard.

### Check Hyper-V

Open PowerShell as Administrator:

``` powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V*
Get-Service vmms
Get-Command Get-VM, New-VM, New-VHD -ErrorAction SilentlyContinue
```

If Hyper-V is available but not enabled, the repository's README
documents this command for supported Windows editions:

``` powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

Restart Windows if prompted. Then verify:

``` powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
Get-Service vmms
```

Do not run the enable command on an unsupported Windows edition or if
your organization manages Hyper-V settings.

------------------------------------------------------------------------

## 4. Clone the repository

Open PowerShell:

``` powershell
New-Item -ItemType Directory -Force C:\VMs | Out-Null
Set-Location C:\VMs

git clone https://github.com/Coastrnaut/OSX-Hyper-V.git
Set-Location C:\VMs\OSX-Hyper-V

git status
git remote -v
```

Keep the repository in a working directory rather than under a protected
Windows system folder.

Before proceeding, inspect the current branch and repository status. Do
not overwrite local changes if the directory already existed.

------------------------------------------------------------------------

## 5. Install and start Claude Code

Install Claude Code using Anthropic's current official instructions:

[Claude Code installation
documentation](https://docs.anthropic.com/en/docs/claude-code/overview)

On Windows, use the currently documented installer for your environment.
Do not copy an installer command from an untrusted third-party site.

From the repository directory:

``` powershell
Set-Location C:\VMs\OSX-Hyper-V
claude
```

Authenticate when prompted. Confirm Claude Code starts successfully:

``` powershell
claude --version
```

### Claude Code safety rules

Keep these rules in effect throughout the workflow:

-   Start in read-only mode for repository and host inspection.
-   Do not allow automatic execution of commands requiring elevation.
-   Never expose passwords, Apple Account credentials, API keys, or
    recovery keys to Claude.
-   Do not disable Defender, SmartScreen, Secure Boot on the Windows
    host, or other Windows protections.
-   Disabling Secure Boot for the **guest VM** is a project-specific VM
    setting; it is not a reason to disable Secure Boot for the Windows
    host.
-   Do not execute downloaded scripts without reviewing their source and
    origin.
-   Do not overwrite `.env`, `.serialdata`, OpenCore configuration, or
    existing VM files without a backup and explicit approval.
-   Do not delete, format, or initialize a physical host disk. Disk
    operations must target an explicitly verified VHDX file.
-   Ask Claude to show exact commands before administrative actions.

------------------------------------------------------------------------

## 6. Stage 1 --- Read-only repository and host assessment

Paste this prompt into Claude Code:

``` text
You are assisting with the Coastrnaut/OSX-Hyper-V repository.

Objective:
Prepare a safe, reproducible plan for building this project's OpenCore EFI
and installing macOS in a Hyper-V virtual machine.

This is a READ-ONLY assessment. Do not execute repository scripts, install
dependencies, download macOS recovery files, edit files, create VMs, or change
Windows/Hyper-V configuration.

Tasks:
1. Read README.md and all relevant documentation under docs/.
2. Inspect scripts/, src/, .env-example, .gitignore, and build configuration.
3. Identify the exact build entry point and dependencies from the checked-out
   repository. Do not assume commands from older instructions still apply.
4. Inspect the VM creation script and determine every parameter and side effect.
5. Identify the expected EFI output and generated support scripts.
6. Determine the OpenCore configuration requirements for my exact CPU.
7. Identify Intel- or AMD-specific patches required by this repository.
8. Explain the .serialdata file and whether it is required for this install.
   Do not generate or invent SMBIOS values.
9. Check the host's Windows edition, Hyper-V availability, CPU, RAM, and disk
   capacity using read-only commands only.
10. Identify any commands that require Administrator privileges.
11. Identify scripts that download, unpack, or execute third-party content.
12. Identify risks of destructive disk operations and how to prevent them.

Output:
- Host compatibility assessment.
- Repository architecture and build workflow.
- Exact dependency list.
- Exact commands proposed for each stage.
- Proposed macOS version, with reasons and compatibility caveats.
- Proposed VM CPU, RAM, disk, and network settings.
- Risks, assumptions, and unresolved questions.

Do not proceed to implementation. Wait for my explicit approval.
```

Review the report carefully. Resolve any unknown CPU model, Windows
edition, storage, or version compatibility before moving on.

------------------------------------------------------------------------

## 7. Stage 2 --- Configure and build the EFI

The repository README documents the following build command for macOS
10.8 and newer:

``` powershell
.\scripts\build.ps1
```

Legacy options documented by the project are:

``` powershell
# macOS 10.7 and older
.\scripts\build.ps1 --legacy

# macOS 10.4–10.5 and 10.6 in 32-bit mode
.\scripts\build.ps1 --legacy --32-bit
```

Do not use legacy options for a modern macOS release.

### Intel CPU considerations

The repository README states that Intel Tiger Lake (11th generation) and
newer systems should follow the Comet Lake configuration guidance and
use the documented CPUID emulation patch. The actual correct
configuration must be confirmed against the current repository and CPU
model.

Do not paste CPU patches blindly. Have Claude check the exact generated
configuration and validate the plist after changes.

### AMD CPU considerations

The project documents an AMD patch-generation script:

``` powershell
.\scripts\amd.ps1 --cpu <core_count>
```

Replace `<core_count>` with the number of virtual CPU cores assigned to
the VM. The README also calls out `DummyPowerManagement` and AMD kernel
patches. The CPU-core count used for patch generation must agree with
the VM configuration and the relevant `cpuid_cores_per_package` patch.

Do not assume that the physical core count should automatically be
assigned to the VM. Choose a conservative VM core count based on host
capacity and the repository's patch requirements.

### Claude prompt --- build preparation

``` text
The read-only assessment is complete. Prepare the EFI build, but do not run
the build yet.

1. Summarize the exact files that need to change for my CPU.
2. Show proposed diffs for all OpenCore and project configuration changes.
3. Explain the AMD or Intel-specific patches that apply.
4. Confirm the VM CPU count that will be used by the configuration.
5. Explain whether .serialdata is needed. Do not invent serial numbers,
   MLB values, ROM values, or UUIDs.
6. List every dependency and where it will be obtained.
7. Identify any download or script execution that needs separate approval.
8. Define how the generated config.plist will be validated.

Wait for my approval of the proposed changes.
```

After approving the changes, use this prompt:

``` text
I approve the specific EFI configuration changes shown in your previous
report.

Apply only those changes. Preserve unrelated files and existing local work.
Before running the build:
- Show the exact build command.
- Show the scripts and external tools it will invoke.
- Confirm the working directory.
- Confirm that the output path will not overwrite an unrelated file.

After approval, run the repository's documented build command.

Then verify:
- The build completed successfully.
- dist/ exists.
- dist/EFI.vhdx exists, if that is the output specified by this repository
  version.
- dist/Scripts/ contains the expected VM and post-install scripts.
- The EFI configuration passes the appropriate OpenCore validation.
- The EFI disk can be inspected and contains the expected EFI directory.

Report all warnings and errors. Do not create or start a VM yet.
```

### Expected build output

The repository README says the build creates a `dist/` directory
containing an EFI VHDX and a `dist/Scripts/` directory with VM setup and
configuration scripts. Confirm exact names in the current checkout
rather than assuming the output has not changed.

------------------------------------------------------------------------

## 8. Stage 3 --- Review Hyper-V VM creation

The project README documents a VM creation script at:

``` text
dist\Scripts\create-virtual-machine.ps1
```

It gives examples in this general form:

``` powershell
# Project README example: defaults
.\dist\Scripts\create-virtual-machine.ps1 -name "My New Virtual Machine"

# Project README example: explicit version and resources
.\dist\Scripts\create-virtual-machine.ps1 `
    -name "macOS" `
    -version 10.15 `
    -cpu 4 `
    -ram 16 `
    -size 128
```

The version example above is Catalina (10.15). Do not use it unless
Catalina is the intended version. Check the current script's parameter
definitions and accepted version values before running it.

### Suggested starting configuration

Use these only as planning values; adjust them to the host and the
repository script:

  -----------------------------------------------------------------------
  Setting                                                  Starting point
  ------------------------------ ----------------------------------------
  VM name                                                         `macOS`

  Generation                                                            2

  Virtual CPUs                                                          4

  Startup memory                                                    12 GB

  Primary virtual disk                                             128 GB

  Network                                    Default Switch, if available

  EFI boot disk                                Project-generated EFI VHDX

  Secure Boot                      Disabled for the guest, as required by
                                                     project instructions
  -----------------------------------------------------------------------

For Big Sur and newer, the repository recommends at least 8 GB of memory
for the installer. A host with 16 GB total RAM may need a smaller VM
allocation to leave sufficient memory for Windows. Do not overcommit
host RAM.

### Claude prompt --- inspect VM creation script

``` text
Inspect dist/Scripts/create-virtual-machine.ps1 and every script or function
it calls.

Do not execute it.

Report:
1. All parameters and their defaults.
2. VM generation and firmware settings.
3. How the EFI VHDX is attached.
4. How the primary macOS disk is created and attached.
5. How the installer/recovery media is obtained or attached.
6. Whether the script creates a virtual switch or uses an existing one.
7. Whether it changes Secure Boot, boot order, checkpoints, or other settings.
8. Whether it overwrites or deletes existing VM or disk files.
9. All Administrator requirements.
10. Exact commands to create the VM using the configuration I approve.

Add safeguards so that the script refuses to overwrite an existing VM or
VHDX unless I explicitly approve that specific operation.

Do not modify the script unless I approve a proposed diff.
```

### Before VM creation

In Hyper-V Manager, confirm that:

-   The VM is Generation 2.
-   The EFI VHDX is attached to the appropriate SCSI location.
-   The primary macOS disk is a separate VHDX.
-   The EFI disk is first in the boot order (the project README
    specifies SCSI location 0 for EFI and a different location for the
    primary disk).
-   Guest Secure Boot is disabled as required by the project.
-   The assigned CPU count agrees with the AMD patch configuration, if
    applicable.
-   The VM has enough memory and storage.
-   The selected virtual switch exists and is appropriate.

Do not attach a physical disk or select a host disk for formatting.

------------------------------------------------------------------------

## 9. Stage 4 --- Obtain and attach macOS recovery media

Follow the repository's current instructions and Apple's permitted
software acquisition methods. The README describes using OpenCore's
`macrecovery.py` tooling to obtain a recovery image and its associated
chunklist from Apple's recovery servers.

Reference material:

-   [OpenCore Install Guide --- macOS
    Recovery](https://dortania.github.io/OpenCore-Install-Guide/installer-guide/)
-   [Apple macOS support and downloads](https://support.apple.com/macos)

The project README describes placing the downloaded `.dmg` and
`.chunklist` files in a directory named:

``` text
com.apple.recovery.boot
```

The README's example layout places that directory alongside `EFI/` at
the root of the EFI volume. Follow the current project instructions for
the exact layout and whether the recovery files should be on the EFI
disk or a separate installer disk.

Do not download macOS images from random file-hosting sites. Do not
provide Apple Account credentials to scripts or Claude.

### Claude prompt --- recovery media verification

``` text
Review the current repository and its documented recovery-media procedure.

Do not download anything yet.

Tell me:
- Which recovery image format this repository expects.
- Whether the recovery image belongs on the EFI VHDX or a separate VHDX.
- The exact expected directory and filenames.
- How to verify the image and chunklist are present and non-empty.
- How to verify the source is an official Apple recovery endpoint.
- How to attach the resulting media to the VM without changing the host's
  physical disks.

Show the commands and wait for approval before downloading or attaching media.
```

------------------------------------------------------------------------

## 10. Stage 5 --- Start the VM and install macOS

Once the EFI, recovery media, and VM configuration have all been
verified:

1.  Open Hyper-V Manager.
2.  Select the newly created VM.
3.  Open **Settings** and re-check the VM configuration.
4.  Start the VM and connect to its console.
5.  Select the OpenCore EFI/recovery boot entry.
6.  If a separate installer disk is used, select the corresponding macOS
    Base System or installer entry.
7.  Wait for macOS Recovery to load.

### Format the virtual macOS disk

In macOS Recovery:

1.  Open **Disk Utility** from the Utilities menu.
2.  Select **View → Show All Devices** if necessary.
3.  Identify the primary virtual disk by its virtual-disk identity and
    expected capacity.
4.  Confirm it is the VM's installation VHDX, not the EFI or recovery
    media.
5.  Erase the primary virtual disk:
    -   Scheme: GUID Partition Map
    -   Format: APFS for macOS 10.13 and newer
    -   Name: `Macintosh HD` (or another chosen name)
6.  Quit Disk Utility.

**Destructive-operation warning:** Erasing a disk destroys its contents.
If the disk identity or capacity is unclear, stop. Never erase a disk
merely because it appears first in a list.

### Run the macOS installer

1.  Select **Reinstall macOS**.
2.  Choose the APFS target volume created above.
3.  Follow the installer prompts.
4.  Allow the VM to restart when requested.
5.  At the OpenCore menu, select the installer or newly installed macOS
    entry as appropriate.
6.  Repeat the boot selection through intermediate installer restarts
    until setup completes.
7.  Complete macOS Setup Assistant.

The project README notes that installation may take 30 minutes to 2
hours and can require multiple reboots.

### Claude prompt --- installation troubleshooting

``` text
I am installing macOS using the Coastrnaut/OSX-Hyper-V project.

Help me troubleshoot based on the exact screen or error message I provide.
Do not recommend destructive disk actions without first identifying the
target virtual disk and explaining the consequences.

For each issue:
1. Explain the likely cause.
2. Ask for the relevant OpenCore log or Hyper-V setting if needed.
3. Give the smallest reversible diagnostic step first.
4. Distinguish project-specific workarounds from general macOS advice.
5. Do not recommend random EFI downloads or unverified patches.
```

------------------------------------------------------------------------

## 11. Stage 6 --- Post-install Hyper-V support

The repository says its EFI disk includes a `Scripts/` directory with
post-install support. It documents a script to install the
MacHyperVFramebuffer driver and configure additional Hyper-V support.

After booting into macOS, mount/open the EFI volume and inspect the
script before running it.

The README documents:

``` bash
cd /Volumes/EFI
bash ./Scripts/post-install.sh
```

The project also documents an optional optimization script:

``` bash
cd /Volumes/EFI
bash ./Scripts/optimize-vm.sh
```

The optimization script is described as disabling Spotlight indexing,
reducing disk I/O, and turning off system animations. Review its
contents and understand the effects before running it.

### Claude prompt --- post-install script review

``` text
Review the post-install.sh and optimize-vm.sh scripts from the EFI volume.

Do not execute them.

For each script:
- Explain every significant operation.
- Identify files, services, daemons, drivers, permissions, and system settings
  it changes.
- Identify whether it is safe to run more than once.
- Identify how to reverse its changes, if possible.
- Explain whether the current macOS version is supported by the script.

Then provide a cautious execution and verification plan.
Wait for approval before running any command inside the macOS guest.
```

------------------------------------------------------------------------

## 12. Stage 7 --- Validate the installation

After the first successful macOS desktop login, verify:

### Hyper-V host checks

``` powershell
Get-VM -Name "macOS" |
    Format-List Name, State, Generation, ProcessorCount, MemoryAssigned,
                Uptime, Status

Get-VMHardDiskDrive -VMName "macOS" |
    Format-Table ControllerType, ControllerNumber, ControllerLocation, Path
```

Adjust the VM name if you chose a different one.

### macOS guest checks

Open Terminal in macOS:

``` bash
sw_vers
uname -a
sysctl -n hw.ncpu
df -h
```

Verify:

-   The installed macOS version is the intended version.
-   The guest boots from its installed system volume.
-   The EFI boot entry remains available.
-   The VM can shut down and restart cleanly.
-   Network access works if configured.
-   Display resolution is usable.
-   The post-install support behaves as expected.

Do not assume audio or GPU acceleration works; the project's README
documents these as limitations.

### Backup

After confirming the VM boots, shut it down cleanly and back up:

-   The VM configuration.
-   The primary macOS VHDX.
-   The EFI VHDX.
-   Any project configuration needed to reproduce the EFI build.

Keep backups outside the VM's active storage directory. Do not copy a
running VM's VHDX as if it were a consistent backup.

### Optional: iOS device wireless debugging

Hyper-V cannot pass a USB-connected iPhone or iPad through to the macOS
guest. Instead, use Xcode's network pairing (Xcode 15+ with iOS 17+) to
install, run, and debug apps over Wi-Fi.

**1. Move the VM to an external (bridged) switch.** The VM creation
script connects the VM to the NAT'd `Default Switch`, which does not
pass the Bonjour/mDNS traffic Xcode needs to discover devices. On the
Windows host, in an elevated PowerShell session, find the name of the
connected physical adapter:

``` powershell
Get-NetAdapter -Physical | Where-Object Status -eq 'Up' | Select-Object Name, InterfaceDescription, Status
```

Shut the VM down, then create the external switch with the value from
the **Name** column and attach the VM to it:

``` powershell
New-VMSwitch -Name "External" -NetAdapterName "Wi-Fi" -AllowManagementOS $true
Connect-VMNetworkAdapter -VMName "macOS" -SwitchName "External"
```

Ethernet bridges more reliably than Wi-Fi; some Wi-Fi drivers do not
bridge cleanly. After booting, the VM should have an IP address on the
same subnet as the device.

**2. Prepare the device.**

-   Enable Settings → Privacy & Security → **Developer Mode** (the
    device restarts).
-   Join the same Wi-Fi network/subnet as the VM and keep the device
    unlocked.

**3. Pair in Xcode.**

-   Open Window → **Devices and Simulators** in the macOS guest.
-   Select the discovered device, click **Pair**, and enter the code
    shown on the device.
-   Choose the device as the run destination and build. Sign in under
    Xcode → Settings → Accounts for code signing.

**Notes:**

-   The first connection is slow while Xcode copies the device's debug
    symbols.
-   If the device does not appear, mDNS is usually not reaching the VM:
    the VM is still on `Default Switch`, the Wi-Fi network uses
    client/AP isolation, or the device is on a different VLAN.
-   Work that strictly requires a cable (firmware restore, some
    accessory development) still needs a physical Mac.

------------------------------------------------------------------------

## 13. Troubleshooting reference

Use the current repository README and issues as the authoritative
project-specific reference. The following cases are called out by the
README.

  --------------------------------------------------------------------------------
  Symptom                                      Project-documented direction
  -------------------------------------------- -----------------------------------
  Early reboot after selecting installer,      Check installer memory. The README
  `#[EB.MM.AKM|!] Err(0xE) <- EB.MM.MKP`       recommends 6--8 GB for Big Sur and
                                               newer.

  Stuck at `vm_shared_region_start_address()`  Try Reset NVRAM from OpenCore. If
  or                                           the GUI still fails, the README
  `failed lookup: com.apple.dock.fullscreen`   suggests installing Catalina first
                                               and upgrading afterward.

  Reboots after selecting installer            May be normal during installation;
                                               select the installer entry again
                                               when appropriate.

  Display fixed at 1024×768                    Review the project's
                                               `SupportedResolutions` framebuffer
                                               configuration.

  No audio device                              Documented default Hyper-V
                                               limitation.

  Slow graphics                                Expected without GPU acceleration;
                                               default framebuffer is not hardware
                                               accelerated.
  --------------------------------------------------------------------------------

When reporting an issue to the project, collect:

-   macOS version.
-   Windows version/build and edition.
-   CPU model.
-   VM CPU count and RAM.
-   EFI build/release version.
-   OpenCore error text or logs.
-   Relevant Hyper-V VM settings.

Never post serial numbers, SMBIOS identifiers, Apple Account details, or
other private identifiers in a public issue.

------------------------------------------------------------------------

## 14. Final Claude Code prompt --- audit the completed VM

``` text
Perform a final non-destructive audit of my macOS Hyper-V installation.

Do not modify the VM, EFI, macOS installation, or host configuration.

Verify and report:
1. The VM's generation, CPU count, memory, and attached VHDX paths.
2. EFI disk attachment and boot order.
3. The guest macOS version.
4. Whether the VM boots from its installed system volume.
5. Whether the project post-install support appears installed.
6. Known limitations affecting graphics, audio, and resolution.
7. Any mismatch between the VM settings and the OpenCore CPU configuration.
8. Backup recommendations and any remaining risks.

Produce a concise installation report with PASS, WARN, or FAIL for each check.
Do not claim a check passed unless you have evidence for it.
```

------------------------------------------------------------------------

## 15. Useful project references

-   [Coastrnaut/OSX-Hyper-V
    repository](https://github.com/Coastrnaut/OSX-Hyper-V)
-   [Project README](https://github.com/Coastrnaut/OSX-Hyper-V#readme)
-   [Project documentation
    directory](https://github.com/Coastrnaut/OSX-Hyper-V/tree/main/docs)
-   [Project scripts
    directory](https://github.com/Coastrnaut/OSX-Hyper-V/tree/main/scripts)
-   [OpenCore Install
    Guide](https://dortania.github.io/OpenCore-Install-Guide/)
-   [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)
-   [Microsoft Hyper-V
    documentation](https://learn.microsoft.com/windows-server/virtualization/hyper-v/)
-   [Claude Code
    documentation](https://docs.anthropic.com/en/docs/claude-code/overview)
-   [Apple macOS support](https://support.apple.com/macos)

## 16. Completion checklist

-   [ ] Confirm applicable macOS license and host eligibility.
-   [ ] Confirm Windows edition and Hyper-V availability.
-   [ ] Record exact CPU, RAM, and storage.
-   [ ] Clone the Coastrnaut repository.
-   [ ] Complete Claude's read-only repository assessment.
-   [ ] Review and approve CPU-specific OpenCore changes.
-   [ ] Build and validate the EFI.
-   [ ] Review the VM creation script and parameters.
-   [ ] Obtain and verify recovery media using the documented procedure.
-   [ ] Create the Generation 2 VM and verify disk attachment order.
-   [ ] Install macOS onto the intended virtual disk.
-   [ ] Complete post-install script review and approved configuration.
-   [ ] Verify guest boot, networking, display, and Hyper-V integration.
-   [ ] Shut down and back up the completed VM.

------------------------------------------------------------------------

## 17. Installation record (2026-11-07)

This section records the parameters and settings used for an actual
installation following this runbook. Paths use `$env:USERPROFILE` in
place of the user profile directory.

### Host

  Item                        Value
  --------------------------- ---------------------------------------------
  Windows                     Windows 11 Pro, build 26300
  CPU                         Intel Core i9-14900K (24 cores / 32 threads)
  RAM                         64 GB
  Storage                     C: NTFS, ~2.9 TB free
  PowerShell                  Windows PowerShell 5.1.26100
  Git / Python                2.55.0 / 3.10.11
  Hyper-V                     Enabled with
                              `Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All`,
                              then rebooted

`Win32_Processor.VirtualizationFirmwareEnabled` reported `False` before
Hyper-V was enabled. This was because Windows virtualization-based
security (HVCI) was already running a hypervisor. It did not mean
virtualization was disabled in firmware.

### OpenCore configuration change

The i9-14900K is newer than Tiger Lake, so the Comet Lake CPUID spoof
from the project README was added to `src/config.yml`, in the
"Boot fixes" `Kernel:` block:

``` yaml
Kernel:
  # Spoof CPU as Comet Lake for Intel Tiger Lake (11th Gen) and newer
  # (host: i9-14900K / Raptor Lake Refresh).
  Emulate:
    Cpuid1Data:             Data    | <55 06 0A 00 00 00 00 00 00 00 00 00 00 00 00 00>
    Cpuid1Mask:             Data    | <FF FF FF FF 00 00 00 00 00 00 00 00 00 00 00 00>
  Quirks:
    ProvideCurrentCpuInfo:  Boolean | true
```

No AMD patches (`amd.ps1`) were needed, and no `.serialdata` file was
used.

### EFI build

``` powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass `
    -File C:\GIT\OSX-Hyper-V\scripts\build.ps1 `
    -pwd C:\GIT\OSX-Hyper-V
```

-   **Always pass `-pwd` explicitly under Windows PowerShell 5.1.**
    `build.ps1` uses `[CmdletBinding()]`, and in PowerShell 5.1 that
    leaves `$PSScriptRoot` empty when the default value of `-pwd` is
    calculated. The default then becomes `C:\`, so the OCE-Build CLI
    runs as `ocebuild.exe build -c C:\ -p config.yml` and scans the
    entire drive. It looks hung, using one full CPU core and writing
    nothing. The VM scripts under `dist/Scripts/` don't use
    `CmdletBinding`, so their defaults resolve correctly.
-   **Remote code:** the build downloads and runs
    `Qonfused/OCE-Build/main/ci/bootstrap.ps1`. That script downloads
    and runs the unpinned **`nightly`** `ocebuild.exe` release.
-   **Result:** OpenCore 1.0.6, with `dist/` containing `EFI/`,
    `Scripts/`, `Tools/`, `Utilities/` and `Docs/`. This version of
    the build does **not** create `dist/EFI.vhdx`; the VM creation
    script creates the EFI disk.
-   **Validation:** `dist\Utilities\ocvalidate\ocvalidate.exe
    dist\EFI\OC\config.plist` reported "No issues found".
    `Cpuid1Data` in the final plist is `VQYKAAAAAAAAAAAAAAAAAA==`.

### VM creation

The script was run elevated through a wrapper so that a transcript
could be captured:

``` powershell
Start-Transcript -Path .\create-vm.log -Force
& 'C:\GIT\OSX-Hyper-V\dist\Scripts\create-virtual-machine.ps1' `
    -pwd 'C:\GIT\OSX-Hyper-V\dist' `
    -name 'macOS' `
    -version 15 `
    -cpu 8 `
    -ram 16 `
    -size 256
Stop-Transcript
```

  Parameter    Value                              Notes
  ------------ ---------------------------------- ---------------------------------------
  `-pwd`       `C:\GIT\OSX-Hyper-V\dist`          Passed explicitly as a precaution
  `-name`      `macOS`                            VM name and folder name
  `-version`   `15`                               macOS Sequoia (board ID `Mac-7BA5B2D9E42DDD94`)
  `-cpu`       `8`                                Virtual processors
  `-ram`       `16`                               Startup memory in GB
  `-size`      `256`                              macOS disk in GB (room for Xcode)
  `-outdir`    *(default)*                        `$env:USERPROFILE\Documents\Hyper-V`

The recovery download triggers an `Invoke-WebRequest` script-parsing
security prompt in Windows PowerShell 5.1. Answer it in the elevated
console. The download used OpenCore's `macrecovery.py` against
Apple's recovery servers and produced `BaseSystem.dmg` (884 MB) and
`BaseSystem.chunklist`. Both were copied to
`com.apple.recovery.boot/` on the EFI disk. VM creation took about
2 minutes.

### Resulting VM settings (verified with Hyper-V cmdlets)

  Setting                 Value
  ----------------------- ----------------------------------------------------------
  Name                    `macOS`
  Location                `$env:USERPROFILE\Documents\Hyper-V\macOS\`
  Generation              2
  Virtual processors      8
  Startup memory          16 GB
  Dynamic memory          **Enabled** (script default; kept on deliberately)
  Automatic checkpoints   Disabled
  Secure Boot (guest)     Off
  SCSI 0:0                `EFI.vhdx` (FAT32, `EFI/`, `Scripts/`, `Tools/`, `com.apple.recovery.boot/`)
  SCSI 0:1                `macOS.vhdx`, dynamic, 256 GB
  Boot order              SCSI 0:0 (EFI) → Network → SCSI 0:1
  Network adapter         `Default Switch`

If you see memory-related freezes or errors in the guest, the first
thing to try is disabling dynamic memory (VM **Settings → Memory**,
with the VM off).

------------------------------------------------------------------------

**Maintenance note:** This guide is based on the repository's published
README and documentation available at the time it was prepared. The
repository can change. Before executing commands, compare this guide
with the current checked-out README, scripts, release notes, and
documentation. The checked-out repository---not this guide---determines
the actual script parameters and behavior.
