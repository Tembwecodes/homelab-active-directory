# Home Lab: Active Directory Domain Environment

A self-built, two-tier Windows Server/Active Directory lab created to develop hands-on systems administration skills for an IT Support career, using VirtualBox.

## Overview

This lab simulates a small business network with a domain controller and a domain-joined client, configured and managed entirely from scratch. The goal was to practise the core day-to-day tasks of a 1st Line/IT Support role: building servers, managing Active Directory, joining clients to a domain, applying Group Policy, and troubleshooting real connectivity and configuration issues as they came up.

## Architecture

| Component | Role | OS |
|---|---|---|
| **DC01** | Domain Controller (Server Core) | Windows Server 2022 |
| **Win11-Client** | Domain-joined workstation | Windows 11 Pro |

Both VMs run in Oracle VirtualBox on a single host, connected via:
- **Internal Network (`intnet`)** — isolated network used for domain communication between DC01 and the client
- **NAT adapter** (client only, added temporarily) — used solely to download Windows features requiring internet access, without exposing the domain network

**Domain:** `homelab.local`

## What Was Built

### Domain Controller (DC01)
- Windows Server 2022, installed and managed via **Server Core** (no GUI) — managed through Sconfig and PowerShell
- Promoted to Domain Controller for `homelab.local`
- Static IP configuration with DC01 acting as its own DNS server
- Organisational Units created: **IT, HR, Finance**
- AD user accounts created across the OU structure
- Security group: **IT-Support**
- Group Policy Object: **IT-Password-Policy**, linked to the IT OU, enforcing a minimum password length

### Client Workstation (Win11-Client)
- Windows 11 Pro installed from an official Microsoft ISO
- Configured with a local account during setup (bypassing the forced Microsoft account/internet requirement), later used to validate domain login
- VirtualBox Guest Additions installed for proper display scaling and input integration
- Static IP configured, with DNS pointed at DC01
- Joined to the `homelab.local` domain
- **RSAT: Active Directory Domain Services and Lightweight Directory Services Tools** installed, enabling full GUI-based AD management (Active Directory Users and Computers) directly from the client

## Skills Demonstrated

- Windows Server installation and Server Core administration
- Active Directory Domain Services: OU design, user/group management, domain promotion
- Group Policy creation and application
- DNS and static IP network configuration
- Domain join process and AD-based authentication
- RSAT/remote administration tooling
- VirtualBox virtualisation: networking modes (Internal Network vs NAT), VM resource management
- Systematic troubleshooting under real (not staged) failure conditions

## Troubleshooting Highlights

Real issues encountered during the build, and how they were diagnosed and resolved:

**Windows 11 ISO wouldn't boot**
The VM's virtual optical drive had no ISO mounted. Re-downloaded a verified ISO directly from Microsoft's official site and re-attached it via VirtualBox storage settings.

**OOBE forced a Microsoft account with no internet access**
The client sits on an isolated internal network by design, so the standard "I don't have internet" bypass wasn't initially appearing. Used the `OOBE\BYPASSNRO` command (via Shift+F10) to enable local account creation — and when keyboard shortcuts weren't reaching the VM, used VirtualBox's on-screen soft keyboard to send the key combination reliably.

**RSAT installation failed silently, then failed with a COMException**
Diagnosed using `Get-WindowsCapability` to confirm the feature wasn't installed, then `Add-WindowsCapability` directly via PowerShell to surface the actual error. The first attempt failed with a generic "operation was canceled" error — confirmed the NAT adapter still had working internet via `Test-NetConnection`, then retried; the second attempt succeeded.

**Intermittent "domain controller not operational" errors**
AD Users and Computers would periodically lose contact with DC01, despite DC01 showing as "Running." Verified all four core AD services (DNS, NTDS, Netlogon, DFSR) were healthy and DC01 had ample free memory — ruling out a DC-side fault. Traced the real cause to **host machine memory pressure** (85–95% usage) while running two VMs simultaneously, which was intermittently stalling VirtualBox's internal networking. Resolved by trimming both VMs' allocated RAM (Win11-Client: 4096 MB → 3072 MB; DC01 confirmed at 2048 MB), easing pressure on the 16 GB host.

**Duplicate/misnamed OU ("Compters")**
A typo OU sat alongside the correct "Computers" container from the original DC01 build. Attempted rename failed (name collision with the existing container), so disabled "Protect object from accidental deletion" via Advanced Features view and deleted the duplicate cleanly.

## Next Steps

- [ ] Expand GPO testing (software restriction policies, login scripts)
- [ ] Add a file server role with NTFS/share permissions tied to security groups
- [ ] Document full screenshots/walkthrough for each build stage
- [ ] Explore certificate services or a second domain controller for redundancy practice

---

*Built as part of ongoing IT Support / Cybersecurity skills development. Feedback and suggestions welcome.*
