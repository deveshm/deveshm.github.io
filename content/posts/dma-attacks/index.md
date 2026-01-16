---
title: "Overview of DMA attacks"
date: 2023-01-28
description: "Learn about DMA attacks and how to protect yourself against them"
summary: "Learn about DMA attacks and how to protect yourself against them"
tags: ["dma", "defense", "pci"]
---
This post is inspired by @marcing, author of [pentests.pl](https://pentests.pl). Thanks for your presentation Marcin!

DMA or **Direct Memory Access** attacks involve using hardware components to directly access a computer's RAM (Random Access Memory) without needing to go through the CPU. This functionality is provided for performance reasons as bypassing the CPU saves time when fetching data from and writing data to RAM.

If we are able to utilise DMA to directly access RAM, we may be able to completely compromise a machine without needing to authenticate to it first. The researcher most well known for research on these attacks is Ulf Frisk, who has some amazing videos on his Youtube channel [here](https://www.youtube.com/channel/UC2aAi-gjqvKiC7s7Opzv9rg).

A DMA attack compromises of hardware, software, targets, and the steps involved in the process. Let's start with the **hardware**:

**PCI** (Peripheral Component Interconnect) is a local computer bus for attaching hardware devices in a computer. This basically just means it is a standard for allowing you to connect hardware devices to the inner workings of a computer e.g. graphics cards, hard drives, SSDs, Wi-Fi etc. **PCIe** is a serial computer expansion bus standard which extends on PCI and is much faster for data transfer. There are a bunch of other standards like mini PCIe, mini SATA and M2 (which replaced mini SATA).

To connect to a PCIe bus, there are multiple hardware based options, including FPGA based options and USB3380 boards. A list of supported FPGA devices can be found [here](https://github.com/ufrisk/pcileech-fpga#supported-devices). The required drivers for using a Windows attack machine with a PCIe connection to the Victim can be found [here](https://github.com/ufrisk/pcileech/wiki/PCILeech-on-Windows). You may also need some adapters to connect your hardware devices to the Victim, and a list of them can be found [here](https://github.com/ufrisk/pcileech#hardware-based-memory-aqusition-methods).

The most common **software** used for PCI based attacks is called pcileech by the same researcher, and full source code for it can be found [here](https://github.com/ufrisk/pcileech). There is a similar attack for FireWire / Thunderbolt interfaces (which is also based on PCI), and the most common software used for this is called Inception, found [here](https://github.com/carmaa/inception). A demonstration of Inception can be found [here](https://www.youtube.com/watch?v=oW2ABJ0SN88).

So, what can you do with DMA attacks?

The **target** may be a linux, mac, or Windows based machine. With DMA attacks, most commonly you can read / write to and from memory. This means you basically own the Victim machine. You can mount the live RAM as a file on your attacker machine and see all the contents, you can execute kernel code on the target system, you can spawn a system shell and other executables, you can pull and push files, and you can patch / unlock machines to remove password requirements.

The **Steps** involved in a DMA attack include:

 1. Opening the Victim machine and removing the battery
 1. Finding an empty PCIe slot or emptying a currently used slot
 1. Booting up the Victim and connecting it to a hardware memory acquisition device e.g. the USB3380-EVB
 1. From Windows, you can run commands such as `pcileech.exe testmemread`, `pcileech.exe dump`, `pcileech.exe kmload -kmd win10_x64` and spawn a system shell using `pcileech.exe wx64_pscmd -kmd 0xXXXXX000`.
 1.  1. Note that the kernel module load function injects a module into RAM which utilises `HAL.dll`. This DLL is a file used by Windows for communication with hardware components (stands for Hardware Abstraction Layer). It is usually loaded into memory with a static virtual and physical memory address in windows kernel memory, and so it can be overridden by pcileech to perform our own actions first and then continue its execution.
 1. Profit
 1.  1. Kill AV drivers, leave a backdooor, run DOOM

To **protect** yourself, here are common mitigations:

 - Physical security of hardware interfaces that support DMA
 - Recent versions of Microsoft Windows require drivers to be digitally signed by Microsoft, which aims to prevent any non-signed drivers from being installed
 - Recent Linux kernel versions allow you to disable FireWire
 - Using an [IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) unit to only allow certain devices to access memory. This is also [utilised in Windows 10](https://docs.microsoft.com/en-us/windows/security/information-protection/kernel-dma-protection-for-thunderbolt#how-windows-protects-against-dma-drive-by-attacks) but unfortunately doesn't work for all DMA attacks.
 - Never store sensitive data unencrypted RAM (glhf)

Cya next time!

Live and Learn!

Uri Frisk's blog: http://blog.frizk.net/