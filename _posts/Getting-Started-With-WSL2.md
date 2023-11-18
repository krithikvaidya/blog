---
layout: post
title: "Getting Started With The Windows Subsystem For Linux"
author: "Krithik Vaidya"
tags: [WSL 2, linux, windows]
image: /WSL/linux.png
---

## Introduction

The Windows Subsystem for Linux allows Windows 10 users to run a Linux environment directly on Windows, without the extra overhead of needing a dual-boot or VM-based setup.  

## WSL 1 vs WSL 2

WSL 1 (initially released in August 2016), is the first version of the Windows Subsystem for Linux which provides a layer on top of the Windows kernel, enabling users to run Linux executables (ELF) on Windows. There is no real Linux kernel involved here -- WSL 1 just provides a Linux-compatible kernel interface for Windows 10. It mainly performs the translation of Linux system calls to Windows system calls. For the Linux applications to run, we need to first install a Linux distribution (such as Ubuntu) from the Windows store. This then provides the required low-level dependencies for the application to run.

However, there were a few problems with this approach. Since there is no "true" Linux kernel involved, WSL 1 is not capable of running all Linux software, such as 32-bit binaries, or those that require specific Linux kernel services not implemented in WSL (in particular, kernel modules such as device drivers). 

WSL 2 (initially released in June 2019 and generally released in April 2020) consists of an actual Linux kernel running inside Windows 10. It uses modern advancements in the Hyper-V technology (which is Microsoft’s virtualization platform, which enables VMs to run on the Host OS). It runs a full Linux kernel in a lightweight Virtual Machine. This adds full system call capability and overcomes the shortcomings of WSL 1, in terms of being unable to run all Linux software. According to the Microsoft docs, "WSL 2 provides the benefits of WSL 1, including seamless integration between Windows and Linux, fast boot times, a small resource footprint, and requires no VM configuration or management. While WSL 2 does use a VM, it is managed and run behind the scenes, leaving you with the same user experience as WSL 1."

## Why I installed WSL 2

My personal reasons for installing WSL 2 :-

- Audio and Mic issues in Dual-booted Ubuntu due to lack of availability of appropriate drivers for my laptop.
- I would generally use Ubuntu for work-related stuff, and switch to Windows for entertainment related stuff. It got annoying to constantly switch.
- While installing Ubuntu as dual-boot, I created the partition for it in my SSD. I ran out of space and now it's kinda hard to resize/allocate extra space since the free space is non-contiguous.
- The new Windows Terminal looked kinda cool :P

I installed WSL 2 and the Ubuntu 20.04 distro following the instructions [here](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

## Windows Terminal and Configuring it (Startup, etc.)

Windows Terminal is a terminal application that supports multiple shells (command prompt, powershell, bash, etc.). It supports having multiple tabs open and is extensively customizable. It even supports GPU-based rendering, which is supposed to improve performance. Watch the Microsoft Developer video below to learn more:-

<iframe width="560" height="315" src="https://www.youtube.com/embed/S4Tp_CN3m0o" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

WSL sometimes has an issue where it won't be able to connect to the network (talked about later). To fix this, a PowerShell script needed to be run. So, I was able to configure Windows Terminal, at startup, to run powershell (which is the default), run a powershell script to fix the network issue before starting WSL, start WSL and change directory to the WSL install root.

![Host Capture Filter](/blog/assets/img/WSL/network.png)

## Running GUI apps on Linux

*Disclaimer*: I'm new to this and I'm pretty sure what I've done is not the best way to do it. You could maybe refer to [this](https://www.reddit.com/r/bashonubuntuonwindows/comments/hlebff/tutorial_wsl2_gui_using_xvnc_and_systemdgenie/) to set up a GUI that claims to "have an almost native Ubuntu experience".

Currently, WSL does not support running GUI apps natively (although, according to Microsoft, support for this should be out in the next few months). For now, I've set up X11 forwarding to support an X window system, using the X2go client and server. I pretty much followed the procedure in the "GUI Apps" section of [this](https://dev.to/derkoe/development-environment-in-wsl2-137l) article.

However, the DISPLAY environment variable isn't automatically set correctly, and I have to manually set it everytime. I've mentioned this in more detail [here](https://stackoverflow.com/questions/64204441/setting-a-consistent-display-number-for-x2go-client)

For my purposes, this is quite fine for now as I have very limited use cases for running graphical applications within WSL. Starting VS Code (which is the major GUI app that I use) from Ubuntu WSL starts it using the normal Windows Desktop environment, which is good.

## Network Issues

I'd have issues where sometime WSL wouldn't be able to connect to the network for whatever reason. Restarting the Host Network service and Hyper-V network adapters everytime I opened my terminal seem to fix the issue. So I added the following powershell script to run at startup of my Windows Terminal:

```

echo "Restarting Host Network Service"
Stop-Service -name "hns"
Start-Service -name "hns"

echo "Restarting Hyper-V adapters"

Get-NetAdapter -IncludeHidden | Where-Object `
    {$_.InterfaceDescription.StartsWith('Hyper-V Virtual Switch Extension Adapter')} `
    | Disable-NetAdapter -Confirm:$False

Get-NetAdapter -IncludeHidden | Where-Object `
    {$_.InterfaceDescription.StartsWith('Hyper-V Virtual Switch Extension Adapter')} `
    | Enable-NetAdapter -Confirm:$False

```


## Moving Ubuntu To A Separate Drive

When you install Ubuntu on Windows, it installs it by default to the same drive as where your Windows OS is installed. This was not good for me as I was running out of space there, and I had to move it to another drive. Fortunately, WSL provides functionality to import and export your distro, and I was able to move Ubuntu using the following command:

```
wsl.exe --export Ubuntu-20.04 - | wsl.exe --import Ubuntu_20.04 D:\WSL2\Ubuntu -
```

To remove the old distro, I ran
```
wsl.exe --unregister Ubuntu-20.04
```

## Access Linux filesystems in Windows and WSL 2

If you have some files in non-Windows filesystems (such as an ext4 filesystem if you have a Linux distro installed as dual boot), Microsoft recently (Sept 2020) released a new feature allowing you to mount and access them. If you wish to know more about this, read the article [here](https://devblogs.microsoft.com/commandline/access-linux-filesystems-in-windows-and-wsl-2/).

## Current Issues

- CUDA support only available on the Windows 10 Development Channel, which is not the most stable and there are more frequent windows updates
- CUDA support is still unstable - [1](https://github.com/microsoft/WSL/issues/6014#issuecomment-707233874), [2](https://github.com/microsoft/WSL/issues/6098) 
- Windows has a case insensitive file system, where files/folders with the same names but having characters with varying capitalization are not treated differently. For example, *test.txt* and *Test.txt* are not different and only one of them can exist in a given folder. Whereas Linux is case-sensitive. This can sometimes cause issues when you need to move files/folders from folders created by Ubuntu on WSL into folders created by Windows. There is, though, a [way](https://www.auslogics.com/en/articles/enable-case-sensitive-file-and-folder-names/) to make Windows folders case-sensitive on a per-folder basis.

- Doesn't natively support running GUI apps (yet).

## Conclusion

I've been using WSL 2 only for a short time now (about 2 weeks), but I'm pretty happy with it's performance and stability. Windows Terminal is the icing on top, bringing in a more modern terminal experience. Overall, I'm glad that I don't have to face any more driver-related and other dual-boot-related issues anymore.

## References

[https://wiki.ubuntu.com/WSL](https://wiki.ubuntu.com/WSL)  
[https://docs.microsoft.com/en-us/windows/wsl/](https://docs.microsoft.com/en-us/windows/wsl/)  
[https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux)  
[https://devblogs.microsoft.com/commandline/access-linux-filesystems-in-windows-and-wsl-2/](https://devblogs.microsoft.com/commandline/access-linux-filesystems-in-windows-and-wsl-2/)  
[https://stackoverflow.com/questions/62533122/how-to-move-a-windows-10-wsl-2-linux-distribution-to-another-location](https://stackoverflow.com/questions/62533122/how-to-move-a-windows-10-wsl-2-linux-distribution-to-another-location)  