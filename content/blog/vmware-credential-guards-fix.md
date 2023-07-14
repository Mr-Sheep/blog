---
title: "VMware Credential Guards Fix"
date: 2019-12-01T19:31:35+08:00
draft: false
tags: ["Vmware"]
categories: ["system"]
---

Just can't open my vm

<!--more-->

# Disable Hyper-V

Fireup a powershell as administrator, type in following commands

`bcdedit /set hypervisorlaunchtype off`

Then reboot

# Device guard

### **Disable Device Guard**

Editing the Registry with following key: **HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity**

Set: Name = “**Enabled”** Type =dword Data = **0**

Then reboot

Also, you can start **gpedit.msc**

Under **Computer Configuration\Administrative Templates\System\Device Guard**, change it to **disabled**.

**If you see the same settings as below, you might not have Device Guard enabled.**
