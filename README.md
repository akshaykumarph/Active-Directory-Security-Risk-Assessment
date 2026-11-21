# Active Directory Security Risk Assessment

AI-assisted security risk assessment of a Windows Server 2022 Active Directory domain controller, mapped to MITRE ATT&CK with remediation recommendations.

I'm making a deliberate pivot from running my family's business into cybersecurity, specifically GRC and security-risk work. This project (Course-End Project 02 of my Advanced Executive Program in Cybersecurity, Simplilearn/IIIT-Bangalore) was my first time standing up and attacking a full Active Directory environment from scratch, so there was a fair amount of trial and error along the way. I've kept that honest in the write-up rather than cleaning it into a seamless story.

## What I built and broke

I stood up a Windows Server 2022 Domain Controller (`mydomain.local`, hostname `DC01`), planted three deliberate misconfigurations, then switched hats and attacked it from a Kali Linux VM using nothing but standard open-source tools. Three of those findings chained into actual cracked plaintext credentials:

| # | Finding | MITRE ATT&CK |
|---|---|---|
| 1 | Anonymous SMB session allowed, domain SID disclosed (deeper RID-cycling was blocked by hardening) | T1590 |
| 2 | An SMB share gave standard domain users read access to internal files it shouldn't have | T1135 / T1039 |
| 3 | A user's LDAP `description` field had a plaintext password sitting in it | T1552 |
| 4 | One account had Kerberos pre-auth disabled, so I could pull an AS-REP hash with zero credentials | T1558.004 |
| 5 | A service account with an SPN was Kerberoastable | T1558.003 |
| 6 | Both of those cracked hashes shared the same password, so cracking one effectively cracked both | Credential reuse |

I also confirmed SMB1 was off and SMB signing was enforced, verified four separate ways, so not everything in this lab was broken.

## How it went

1. Built the DC: new forest, static IP, renamed host, planted a leaked password, a Kerberoastable SPN, and an AS-REP-roastable account
2. Set up and hardened SMB sharing, with one share deliberately over-permissioned for later discovery
3. From Kali: full Nmap scan, then SMB/RPC enumeration with enum4linux-ng, smbclient, and netexec (hit a tool-naming mixup partway through, more on that in the full report)
4. Pulled Kerberos hashes via AS-REP Roasting and Kerberoasting with Impacket
5. Ran both hashes against rockyou.txt first, which went nowhere. A custom wordlist built around the lab's planted password cracked both, which is how I confirmed the reuse
6. Enumerated LDAP both anonymously and authenticated. The authenticated pass is where the plaintext password turned up
7. Mapped everything to MITRE ATT&CK and wrote up remediation for each finding

Full write-up: [`Active-Directory-Security-Risk-Assessment.pdf`](./Active-Directory-Security-Risk-Assessment.pdf)
Every command I ran, task by task: [`Commands-Reference.md`](./Commands-Reference.md)

## Environment

Windows Server 2022 DC and a Kali Linux 2026.2 attacker VM, both on an isolated VMware Workstation Pro network.

## Tools

Nmap, enum4linux-ng, smbclient, netexec, Impacket (GetNPUsers, GetUserSPNs), hashcat, and ldapsearch, plus the native AD DS tooling on the Windows side.

## A note on the AI

I used Claude in place of the ChatGPT the course brief names, mainly for clarifying commands, working through a couple of genuine dead ends (the DFSR warning in `dcdiag` looked like a real problem until I checked the actual event log and found it was a known false positive on a single, newly-promoted DC with no replication partner), and helping draft the report. Every command, every click, and every read of the results was mine; Claude never touched the lab itself.

---

Part of my AEP Cybersecurity Capstone (Simplilearn / IIIT-Bangalore). Companion assessment on the same lab, *Inside the Vault: Cracking the Code of Corporate Domain Secrets*, link to come once it's published.
