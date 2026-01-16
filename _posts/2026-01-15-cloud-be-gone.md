---
layout: post
title: "Cloud-Be-Gone: Eliminating OneDrive for Good"
date: 2026-01-15 09:00:00 -0500
categories: [Windows]
tags: []
author: Th3_4ngl3r
toc: true
comments: false
---

## Uninstalling Microsoft One Drive

Removing OneDrive permanently from Windows 11 and ensuring it does not return with future updates is a multi‑step process that administrators should consider when the service is not required for business continuity. Beyond operational preferences, there are legitimate privacy‑driven reasons to eliminate OneDrive from environments where strict data‑handling policies apply. By default, OneDrive encourages the redirection and synchronization of user data to Microsoft’s cloud infrastructure. For organizations that manage sensitive, regulated, or confidential information, this automatic cloud exposure can introduce compliance risks and reduce control over where data resides.

Additionally, as Microsoft continues to integrate AI‑powered features across its ecosystem, cloud‑stored files may be subject to automated analysis to enable capabilities such as content recommendations, search enhancements, or productivity insights. While these features are designed to improve user experience, some organizations may view any form of cloud‑based data processing as incompatible with their privacy requirements or internal security posture. Preventing OneDrive from operating ensures that sensitive information remains fully local and outside the scope of cloud‑driven AI systems.

Because OneDrive is deeply embedded into Windows 11, a simple uninstall is not sufficient to stop it from reappearing for new user profiles or after major feature updates. A comprehensive removal strategy is necessary to maintain a controlled, cloud‑free environment and uphold strict data‑governance standards.

OneDrive has recently faced several high‑impact issues—including real data loss events, security vulnerabilities that enable excessive data access, and new features that increase the risk of accidental or intentional data exfiltration.  Recent occurences listed below.

Catastrophic user data loss due to sync behavior confusion.
In July 2025 reports of users losing all personal and work files after a misunderstanding of OneDrive sync behavior, which required a formal escalation for backend restoration.  This highlights how OneDrive’s sync model can silently remove or relocate files if users misinterpret “backup” vs. “sync.”

In June 2025 reports of users whose OneDrive account was suspended without warning.  This resulted in some cases of, locking away 30 years of personal photos and work files, with no meaningful support response.

This demonstrates the risk of OneDrive as a single point of failure.

## Step 1:  Uninstall the OneDrive App

- Right-click the Start button and select Terminal (Admin).
- Type the following command and press Enter:

```powershell
winget uninstall Microsoft.OneDrive
```

*If prompted, to agree to source terms, type Y*

## Step 2: Block OneDrive via Group Policy

Blocking OneDrive via local group policy instructs Windows that the OneDrive engine is forbidden.  This prevents OneDrive from being installed via Windows Updates or with new profiles.

- Press Win + R, type gpedit.msc, and press Enter.

**NOTE:** If the operating system is Windows 11 Home, see the Registry note below.

- Navigate to: Computer Configuration > Administrative Templates > Windows Components > OneDrive

- Double-click Prevent the usage of OneDrive for file storage. Set it to Enabled, click Apply, and then OK.

**IMPORTANT!** For Windows 11 Home Users: Since Group Policy Editor isn't available, you must use the Registry. Navigate to HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\OneDrive. Create a new DWORD (32-bit) Value named DisableFileSyncNGSC and set its value to 1.

## Step 3: Prevent Re-installation for Future Users

Windows keeps a "staged" installer that runs every time a new user logs in for the first time. You should disable this trigger.

- Press Win + R, type regedit, and press Enter.

- Navigate to: HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Active Setup\Installed Components\

- Locate the subkey with a value named OneDriveSetup.

- Right-click that subkey and Delete it.  This prevents Windows from running the "First Run" OneDrive setup for new profiles.

## Step 4: Clean up the File Explorer Sidebar

Even after uninstallation, a "ghost" OneDrive folder often remains in the File Explorer sidebar.

- In Registry Editor, navigate to: HKEY_CLASSES_ROOT\CLSID\{018D5C66-4533-4307-9B53-224DE2ED1FE6}

- Find the value System.IsPinnedToNameSpaceTree.

- Double-click it and change the value data to 0.

## Conclusion

I have created a Powershell script that can be used to automate this process!  Find it on my [Github](https://github.com/th34ngl3r/odkiller/tree/main)!

## Alternative OneDrive Removal with Appx

Using Get-AppxPackage is a common way to remove modern Windows apps.  For OneDrive, it is actually less effective than the methods used in the script provided above.

- OneDrive is not a standard Universal Windows Platform (UWP) App.  Most apps you see in the Start Menu (Calculator, Photos, etc.) are AppX/MSIX packages. OneDrive is a Win32 executable  that is "staged" in the system.

- Appx Commands: Usually return nothing or fail to remove the actual sync engine.

- The Script's Method uses winget or the native OneDriveSetup.exe /uninstall flag, which targets the actual binary files and drivers that handle file syncing.

- Using Remove-AppxPackage only removes the OneDrive app for the current user. Appx does not stop Windows from reinstalling Onedrive for the next user. To stop that, you would need to "deprovision" it, but since OneDrive is often tied to the OS image as a system component, it frequently ignores Remove-AppxProvisionedPackage.

- AppxPackage removal does not touch the registry keys that control the File Explorer sidebar. Even if you "uninstall" the package, the OneDrive - Personal folder will often remain in your sidebar as a broken link. The script's registry approach is required to clean this up.

If you are on a specific build of Windows 11 where OneDrive is showing up in the Appx list, you can add these two lines to the "Uninstall" section of the script or run manually. This covers both the current user and prevents it from being provisioned to new users if the system recognizes it as an Appx.

```powershell
# Remove for current user
Get-AppxPackage -AllUsers *OneDrive* | Remove-AppxPackage#

# Remove from the system "Seed" (Provisioned) list so new users don't get it
#Get-AppxProvisionedPackage -Online | Where-Object {$_.PackageName -like #"*OneDrive*"} | Remove-AppxProvisionedPackage -Online
```
