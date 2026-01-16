---
draft: true
title: "Windows Penetration Testing and Exploitation 101"
date: 2026-01-16
description: "Windows Penetration Testing and Exploitation 101"
summary: "Windows Penetration Testing and Exploitation 101"
tags: ["windows", "hacking"]
---
Windows is a complicated beast, and the security landscape for Windows is notorious for being hard to understand. I thought I'd summarise the main points red teamers and penetration testers should know about the Windows security environment and give an overview of what I know.

As always, hope you enjoy!

## Definitions

Security Model
* Users, Groups, Domains, Trusts, Forests

NTLM and Kerberos
* Protocols used for providing authentication, integrity and confidentiality in a Windows domain environment
* NTLM hashes and PTH attacks
* Kerberos tickets and golden ticket / silver ticket attacks

ACLs and Group Policy

PowerShell

Windows Defender

AMSI (Anti-malware scan interface)

ASLR (Address Space Layout Randomisation)
* When executables are loaded into memory, the address where they are loaded changes so as to make the location of functions and libraries difficult to find without knowledge of the base address

DEP (Data Execution Prevention)
* Memory areas can be marked as non-executable e.g. data structures

Credential storage locations

 * NTDS.DAT
 * SAM / SYSTEM files
 * LSASS process in memory


AMSI and PowerShell constrained mode bypass e.g. powershell -version 2
Kerberoasting
Living off the land e.g. rundll32.exe, installutil.exe, regsvcs.exe,
regasm.exe
ASLR bypass
DEP bypass
AMSI and PowerShell constrained mode bypass e.g. powershell -version 2
Kerberoasting
Living off the land e.g. rundll32.exe, installutil.exe, regsvcs.exe, regasm.exe
KASLR bypass (blog by offensive security)


## Favourite tools

Impacket's wmiexec.py, psexec.py, mssqlclient.py, secretsdump.py
PowerShell versions of older tools e.g. Invoke-Mimikatz
PowerUp and PowerView
 * Looks for privesc using %PATH% hijacking, DLL load order attacks, writable services, GPO files and process rights


BloodHound / SharpHound
 * Can help find kerberoastable users, rights like DCSync


Group policy auditor and gpp-decrypt
Other useful PowerShell scripts e.g. Invoke-Mimikatz, Invoke-Obfuscation, Invoke-ReflectivePEInjection, Invoke-DllInjection


---
Live and Learn!

Windows KASLR Bypass blog by Offensive Security: https://www.offensive-security.com/vulndev/development-of-a-new-windows-10-kaslr-bypass-in-one-windbg-command/