---
layout: post
title:  "Run Docker with Hyper-V isolation in an Azure Virtual Machine"
---

Setting up the Layout Tests for the WebKit Windows Port, I ran into issues running the tests 
in Docker with process isolation. Specifically creating off-screen windows and drawing to them didn't 
seem to work the same as in hyper-v isolation.

I'm running these tests in Docker CE (Moby) on Windows Server 2025 running on Standard D8ads v6 Azure instances. 
If you attempt to turn on Hyper-V via "Turn Windows features on and off" it doesn't work; and 
there's a number of blogs stating you need to recreate the VM with security type of "Standard" 
instead of "Trusted Launch". However I found that turning on Hyper-V via Powershell worked:

```
Enable-WindowsOptionalFeature -Online -FeatureName containers –All
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V –All
```

Enabling Microsoft-Hyper-V will require the VM to restart.
