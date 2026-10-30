# Windows 10 to Windows 11 In-Place Upgrade with MDT

This guide describes how to upgrade existing Windows 10 computers to Windows 11 using the Microsoft Deployment Toolkit (MDT). The upgrade is an in-place upgrade: applications, files and settings are kept.

> **Note:** MDT is no longer under active development by Microsoft, but it remains functional and supported for Windows 11 deployments when used with the current Windows ADK.

## Contents

1. [Requirements](#1-requirements)
2. [Install the tools](#2-install-the-tools)
3. [Create the deployment share](#3-create-the-deployment-share)
4. [Import Windows 11](#4-import-windows-11)
5. [Create the upgrade task sequence](#5-create-the-upgrade-task-sequence)
6. [Configure the rules](#6-configure-the-rules)
7. [Update the deployment share](#7-update-the-deployment-share)
8. [Run the upgrade on a client](#8-run-the-upgrade-on-a-client)
9. [Verify the upgrade](#9-verify-the-upgrade)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Requirements

### Server (deployment host)

- Windows Server or Windows 10/11 with at least 50 GB of free disk space
- Local administrator rights
- A network share reachable by all client computers

### Client computers

Each client must meet the Windows 11 hardware requirements:

| Component | Requirement |
|-----------|-------------|
| Processor | 1 GHz or faster, 2 or more cores, 64-bit, on the supported CPU list |
| Memory | 4 GB RAM or more |
| Storage | 64 GB or more |
| Firmware | UEFI with Secure Boot capability |
| TPM | Version 2.0 |
| Graphics | DirectX 12 compatible, WDDM 2.0 driver |
| Operating system | Windows 10, version 22H2 (recommended) |

Check TPM and Secure Boot on a client with PowerShell (run as administrator):

```powershell
Get-Tpm | Select-Object TpmPresent, TpmReady
Confirm-SecureBootUEFI
(Get-CimInstance Win32_TPM -Namespace root\cimv2\Security\MicrosoftTpm).SpecVersion
```

Clients that fail these checks should not be upgraded with this method.

### Downloads

- Windows 11 ISO (Enterprise or Education, volume licence media)
- [Windows Assessment and Deployment Kit (ADK) for Windows 11](https://learn.microsoft.com/windows-hardware/get-started/adk-install)
- Windows PE add-on for the ADK (same version as the ADK)
- [Microsoft Deployment Toolkit](https://www.microsoft.com/download/details.aspx?id=54259), build 8456
- MDT patch KB4564442 (fixes UEFI boot and BDD.log issues with recent ADK versions)

---

## 2. Install the tools

Install in this order on the deployment host:

1. Windows ADK, selecting **Deployment Tools** (and **User State Migration Tool** if required)
2. Windows PE add-on for the ADK
3. MDT (`MicrosoftDeploymentToolkit_x64.msi`)
4. MDT patch KB4564442: replace the files as described in the patch notes, normally in `C:\Program Files\Microsoft Deployment Toolkit\Templates\Distribution\Tools`

---

## 3. Create the deployment share

1. Open **Deployment Workbench**.
2. Right-click **Deployment Shares** and select **New Deployment Share**.
3. Set the path, for example `D:\DeploymentShare`.
4. Set the share name, for example `DeploymentShare$`.
5. Accept the default options and finish the wizard.

Grant read access on the share to the computer accounts or a security group that contains the clients being upgraded. A dedicated service account with read access is also suitable.

---

## 4. Import Windows 11

1. Mount the Windows 11 ISO (for example as drive `E:`).
2. In Deployment Workbench, expand the deployment share, right-click **Operating Systems** and select **Import Operating System**.
3. Select **Full set of source files**.
4. Set the source directory to the mounted ISO drive.
5. Set a destination folder name, for example `Windows 11 Enterprise 23H2`.
6. Finish the wizard.

Full source files are required. A custom WIM image cannot be used for an in-place upgrade.

---

## 5. Create the upgrade task sequence

1. Right-click **Task Sequences** and select **New Task Sequence**.
2. Set the task sequence ID (for example `UPG-W11`) and the name (for example `Upgrade to Windows 11`).
3. On the template page, select **Standard Client Upgrade Task Sequence**.
4. Select the Windows 11 operating system imported in step 4.
5. Complete the remaining pages and finish the wizard.

Applications that must be installed or updated after the upgrade can be added to the task sequence under **State Restore** or after the upgrade step.

---

## 6. Configure the rules

Right-click the deployment share, select **Properties**, and open the **Rules** tab. A basic `CustomSettings.ini` for an unattended upgrade is:

```ini
[Settings]
Priority=Default
Properties=MyCustomProperty

[Default]
OSInstall=Y
SkipCapture=YES
SkipAdminPassword=YES
SkipProductKey=YES
SkipComputerBackup=YES
SkipBitLocker=YES
SkipLocaleSelection=YES
SkipTimeZone=YES
SkipDomainMembership=YES
SkipUserData=YES
SkipSummary=YES
SkipFinalSummary=YES
SkipTaskSequence=YES
TaskSequenceID=UPG-W11
FinishAction=REBOOT
```

Adjust the values to the environment. Remove `SkipTaskSequence` and `TaskSequenceID` to select the task sequence manually on each client.

---

## 7. Update the deployment share

1. Right-click the deployment share and select **Update Deployment Share**.
2. Select **Optimize the boot image updating process**.
3. Wait for the process to complete.

Boot images are not used for an in-place upgrade, but updating the share writes the rules and scripts that the clients use.

---

## 8. Run the upgrade on a client

The upgrade task sequence must be started from inside the running Windows 10 installation. It cannot be started from PXE or boot media.

1. Sign in to the client as a local administrator.
2. Open an elevated command prompt.
3. Connect to the share:

   ```cmd
   net use Z: \\SERVER\DeploymentShare$ /user:DOMAIN\serviceaccount
   ```

4. Start the Lite Touch script:

   ```cmd
   cscript.exe Z:\Scripts\LiteTouch.vbs
   ```

5. If the task sequence was not pre-selected in the rules, select **Upgrade to Windows 11** in the wizard.
6. Wait for the upgrade to finish. The computer restarts several times.

The same command can be distributed by Group Policy, a startup script, or an endpoint management tool for larger groups of computers.

---

## 9. Verify the upgrade

After the final restart:

1. Run `winver` and confirm Windows 11 and the expected version.
2. Check the MDT logs in `C:\Windows\Temp\DeploymentLogs` (main file: `BDD.log`).
3. Check the Windows Setup logs in `C:\$WINDOWS.~BT\Sources\Panther` if the upgrade failed or was rolled back.

---

## 10. Troubleshooting

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| "This PC can't run Windows 11" | TPM, Secure Boot or CPU check failed | Run the checks in section 1 and correct firmware settings, or exclude the device |
| Cannot connect to the share | Permissions or firewall | Check share and NTFS permissions and that SMB is reachable |
| Task sequence fails at the upgrade step | Compatibility block or low disk space | Review `setupact.log` and `setuperr.log` in the Panther folder |
| Upgrade rolls back | Incompatible driver or application | Inspect `setupact.log` for the blocking item and remove or update it |
| Task sequence is not offered in the wizard | Task sequence disabled or wrong template | Confirm that it is enabled and uses the **Standard Client Upgrade** template |
| Script fails with "Access denied" | Not elevated | Run the command prompt as administrator |

---

## References

- [Microsoft Deployment Toolkit documentation](https://learn.microsoft.com/mem/configmgr/mdt/)
- [Windows 11 system requirements](https://learn.microsoft.com/windows/whats-new/windows-11-requirements)
- [Windows ADK installation](https://learn.microsoft.com/windows-hardware/get-started/adk-install)
