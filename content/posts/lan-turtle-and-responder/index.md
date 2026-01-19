---
title: "LAN Turtle + Responder, how to defend"
date: 2019-03-16
description: "Learn about the LAN Turtle and Responder, and how to defend against these technologies"
summary: "Learn about the LAN Turtle and Responder, and how to defend against these technologies"
tags: ["lanturtle", "responder", "defense"]
---

The LAN turtle is a physical device sold by Hak5 that provides the user remote access to the network it is connected to. It comes with a female Ethernet port and a male USB port, allowing it to be connected to a network and a computer.

The LAN turtle has modules you can utilise in programs that you write for the device. These modules include functionality such as:

 - providing stealthy remote communications to allow for remote control e.g. using ptunnel to proxy TCP traffic over ICMP
 - network intelligence gathering / analytics
 - detailed sniffing of the network e.g. URLSnarf for HTTP sniffing

Thus the device can be used as a stealthy way to have a **reverse connection from a victim network to your attacker machine**.

Responder is a tool for listening on the network in a windows domain environment, and responding to LLMNR and NBNS requests seen on the network. It has additional support for listening and responding to HTTP, DHCP and DNS requests.

Both these tools together allow an attacker to easily sniff for Windows Domain credentials on a victim network. You simply install and configure responder on the LAN Turtle, and plug it in to the victim network via ethernet and USB. This works great for getting credentials out of a locked computer or laptop, as the machine usually still sends out network based requests even when it is locked. Responder will respond to such requests (e.g. WPAD requests, browser traffic) and capture any credentials that the laptop sends (e.g. HTTP Basic Auth, NetNTLM and NTLM auth).

This is a difficult attack to protect from, especially because ethernet adapters are allowed to be plugged in and installed on a locked machine, even on newer operating systems. Additionally, computers trust their local network and send out all types of traffic to these attached ethernet adapters.

The recommendation against the poisoning that Responder does is straight forward actually. To protect Windows credentials, we can **disable LLMNR and NBNS** so that Windows defaults to stronger authentication schemes. SSL should also be used for all client connections to FTP, HTTP and SMB servers. Detecting that you have a LAN Turtle attached to your network somewhere is difficult. It's always good to monitor for any unknown network interfaces being added to servers and workstations, and you may consider using **honey tokens** (fake credentials being sent over the network) to try and detect attackers that use them. Additionally, you can allowlist Domain Controllers on client machines so that a host instrusion detection system (HIDS) is able to tell when a device other than these DCs responds to network requests.

---

Live and Learn!

Responder: https://github.com/SpiderLabs/Responder

LAN Turtle: https://shop.hak5.org/products/lan-turtle