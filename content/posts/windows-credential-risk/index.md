---
title: "Credential Risk in a Windows Environment"
date: 2019-03-16
description: "Credential Risk in a Windows Environment"
summary: "Credential Risk in a Windows Environment"
tags: ["windows", "lsass"]
---
This article is about credential risk in a Windows Environment. The scenario is as follows: **You are an Incident Responder or Level 1 Analyst trying to determine why a server is acting strangely, or trying to triage a Security Incident**. You use your administrator credentials or equivalent to **remotely logon to the machine you want to analyse** and gather files and other evidence from the machine. The risk here is that an attacker who has compromised a computer as an administrator has the capability to steal all credentials of users that log on to that computer. Assuming that the victim machine is compromised by an attacker, does Windows provide a mechanism to support this gathering of evidence without revealing your credentials in the memory?

There is also credential risk when an administrator user wants to install a software on a remote machine for System Administration purposes. These system administrators may be revealing their credentials to attackers. Note that the attacker only requires local admin on the victim machine, NOT Domain Admin.

I did not find many articles online on Credential Risk, so thought I would summarise what I've learned.

Some possible places that attackers can steal cached credentials from include:
* The Security Accounts Manager (SAM) database
* Local Security Authority Subsystem (LSASS) process memory
* Active Directory database (domain controllers only)
* The Credential Manager store
* LSA Secrets in the registry

In at least the latest versions of Windows, it should be noted that the registry based credentials cached locally are in the form of "Verifiers" not "Authenticators". This means when an attacker gets credentials out of memory for cached logons, they don't get a hash that can be used to directly authenticate as the user, but a hash that can be cracked to reveal the users actual credentials.

I've seen recommendations that say that using network based logon is more secure than interactive logon when protecting credentials. However, it should be noted that even using `runas` in `/netonly` mode may also reveal your credentials in memory: https://blogs.technet.microsoft.com/jepayne/2016/04/04/when-the-manual-is-not-enough-runas-netonly-unexpected-credential-exposure-and-the-need-for-reality-based-holistic-threat-models/

It essentially depends upon what Logon Type is being used to authenticate to the remote host. This [ired.team article](https://www.ired.team/offensive-security/credential-access-and-credential-dumping/network-vs-interactive-logons) has a great demonstration of the different logon types, and in which scenarios credentials can be dumped from memory. **The only network logon type that doesn't cache credentials in memory is Logon Type 3.**

Microsoft has released [Windows Credential Guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/) {{< icon "shield" >}}, which "uses virtualization-based security to isolate secrets so that only privileged system software can access them". But an attacker can access these secrets if they find a bug in Credential Guard. So a combination of multiple mitigation strategies and activities should be performed.

Recommendations include:
* Using **a host agent to collect Incident Response Artefacts**. Examples of companies that offer such HIDS software include FireEye and Carbon Black, which send artefacts and logs to a central server without needing an administrator to logon to the compromised machine.
* Administrators that want to install software remotely on a machine using RDP should be utilising the **Restricted Admin Mode** flag, which makes sure that the admiministrator's credentials are not sent to the host.
* Completely disable the caching of logon credentials on hosts using Group Policy. This is usually unwanted as it means domain users can only logon to the machine if the DC is online.
* **Monitor Windows Event 4625**, which is for when a user attempts to logon to a computer but "has not been granted the requested logon type"
* Use **MFA for User Accounts** (especially accounts with higher privileges)
* Use a strong password...cheers :)

What's really fun is setting up your own little Windows AD environment at home, and utilising tools like MimiKatz / WCE to test whether you can steal credentials of a remotely logged in user.


---
Live and Learn!

 - Windows 10 credential theft mitigation document from Microsoft: https://download.microsoft.com/download/C/1/4/C14579CA-E564-4743-8B51-61C0882662AC/Windows%2010%20credential%20theft%20mitigation%20guide.docx
 - Restricted admin mode for RDP: https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/dn283323(v=ws.11)#restrictedadmin-mode-remote-desktop