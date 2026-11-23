---
title: "Active Directory enumeration from a standard domain user"
slug: "ad-enumeration-corp-lab"
authors: ["Shravan Kakkar (TheF0-ol)"]
date: "2026-11-23"
tags: ["Active Directory", "Enumeration", "Windows", "Home Lab"]
summary: "Built a Windows Server 2022 domain controller and a domain-joined Windows 11 client in my own lab, then enumerated it as a standard, non-admin domain user with two Python tools I wrote. Found a password policy with no account lockout and confirmed every command needed zero admin rights."
---

## 1. Overview

- Objective: Build a small Active Directory lab from scratch, then enumerate it the way an attacker with ordinary domain credentials would, using tools I wrote myself (`ad_enum.py` and `ad_enum_v2.py`, both in my [AD-Enumeration repo](https://github.com/TheF0-ol/AD-Enumeration)).
- What the lab is: A single-domain forest (`corp.lucid`) with one domain controller and one domain-joined client, running on a fully isolated host-only virtual network.
- What I did: Logged in as a standard domain user (not Administrator) and ran both tools against the real domain, then reviewed what they returned for anything worth flagging.

## 2. Lab details

- Domain controller: Windows Server 2022 Standard (Evaluation), promoted to a new forest root, `corp.lucid` / NetBIOS `CORP`
- Client: Windows 11 Enterprise (Evaluation), joined to `corp.lucid`
- Network: VMware host-only network, `192.168.56.0/24`, no internet access, DHCP disabled, static IPs (DC: `192.168.56.10`)
- Domain objects created for the lab: 4 user accounts (`alice`, `bob`, `carol`, `svc_sql`) and one security group (`IT-Support`)
- Enumeration was run logged in as `alice`, a plain domain user with no admin rights on either machine

## 3. My findings

- No account lockout policy is configured (`Lockout threshold: Never`). A real attacker with no admin rights could attempt unlimited password guesses against any domain account with no automatic lockout to slow them down.
- A clear naming convention (`svc_sql`) identifies a likely service account, something an attacker would specifically target for a Kerberoasting attempt, since service accounts are commonly over-privileged and rarely have their passwords rotated. (Covered as a separate attack write-up.)
- `gpresult /scope computer /v` correctly refused to run (`ERROR: Access Denied`) for a non-admin user. This confirms Group Policy contents are not exposed to standard users by default, which is the expected, correct behavior, not a weakness.
- Every enumeration command I ran (`net user /domain`, `net group /domain`, `net group "Domain Computers" /domain`, `net accounts /domain`, `nltest /dsgetdc`, `nltest /domain_trusts`) succeeded with **no administrator rights at all**. This is worth stating plainly: a domain's basic structure (every username, every group membership, every computer account, the entire password policy, and the domain's trust relationships) is readable by any authenticated user by default. None of this requires exploiting anything, it's the platform's default behavior.

## 4. Tools I used

- `ad_enum.py` — users and group membership, written by me, [source](https://github.com/TheF0-ol/AD-Enumeration/blob/main/ad_enum.py)
- `ad_enum_v2.py` — adds DC discovery, computer accounts, password policy, GPO summary, and domain trusts, written by me, [source](https://github.com/TheF0-ol/AD-Enumeration/blob/main/ad_enum_v2.py)
- Both are plain Python using only the standard library and built-in Windows commands (`net`, `nltest`, `nslookup`, `gpresult`), no third-party AD libraries or admin tooling

## 5. How I did it

### 5.1 Build

1. Built the DC and promoted it to a new forest (`Install-ADDSForest -DomainName corp.lucid -DomainNetbiosName CORP -InstallDns`).
2. Created the 4 test users and the `IT-Support` group with `New-ADUser` / `New-ADGroup` / `Add-ADGroupMember`.
3. Built the Windows 11 client, set its DNS to the DC's IP, and joined it to `corp.lucid`.

### 5.2 Enumeration

4. Logged into the client as `alice` (a plain domain user, confirmed with `whoami` → `corp\alice`).
5. Ran `python ad_enum.py --domain corp.lucid`.
6. Ran `python ad_enum_v2.py --domain corp.lucid --output-dir results`.
7. Reviewed the CSVs both tools produced (real, unedited samples are in the [repo's `samples/` folder](https://github.com/TheF0-ol/AD-Enumeration/tree/main/samples)).

### 5.3 Fixing what broke

8. The first run of `ad_enum_v2.py` surfaced three real bugs that no amount of testing against simulated output would have caught:
   - `nslookup -type=SRV` against the `_msdcs` subdomain consistently timed out against the real DC.
   - `net computer /domain` is not a valid way to list computer accounts, real output showed it returned `The option /DOMAIN is unknown`, since that command is actually for adding or removing a single computer.
   - `nltest /trusteddomains` isn't a real option, real output showed it just printed the command's own usage text.
9. Fixed computer enumeration to use `net group "Domain Computers" /domain` instead (computer accounts are members of that built-in group, and are identifiable by a trailing `$`, e.g. `DC-01$`), and fixed the trust command to `nltest /domain_trusts`. Dropped the SRV lookup entirely since `nltest /dsgetdc` already returns the same discovery data reliably. Reran and confirmed clean output.

## 6. Proof of concept

### 6.1 Command and real output (users, truncated)

```
PS C:\Users\alice\Desktop\ad-tools> python ad_enum.py --domain corp.lucid
Running as: alice (corp\alice)  |  Host: CLIENT01
Domain: corp.lucid

Enumerating domain users (net user /domain)...
  Found 7 user account(s).

Domain Admins (net group /domain)...
  1 member(s).
...
Saved 15 row(s) to ad_enum_20261123_104940.csv
```

### 6.2 Real password policy output (`net accounts /domain`, via `ad_enum_v2.py`)

```
Force user logoff how long after time expires?:       Never
Minimum password age (days):                          1
Maximum password age (days):                          42
Minimum password length:                              7
Length of password history maintained:                24
Lockout threshold:                                    Never
Lockout duration (minutes):                           30
Lockout observation window (minutes):                 30
Computer role:                                        PRIMARY
```

- `Lockout threshold: Never` means no lockout policy at all, an account can be guessed against indefinitely with no automatic block.
- All of the above came from an unprivileged domain user account, no admin rights, no exploit, just the default `net accounts` command.

## 7. Impact

- Reconnaissance: a compromised low-privilege account (phished credentials, a found password, anything) is enough to map every user, every group membership, every computer, and the entire password policy of the domain, with zero additional exploitation.
- The missing lockout policy means a password-spraying attack against any of these accounts would not trigger an automatic lockout, making brute-force guessing meaningfully easier for an attacker who's already reached this stage.
- The presence of a service account following an identifiable naming pattern (`svc_sql`) is itself useful reconnaissance; it narrows an attacker's next step to a specific, likely-overprivileged target.

## 8. Fix

1. Set an account lockout policy (`net accounts /lockoutthreshold:5` or the equivalent Group Policy setting), rather than leaving it at `Never`.
2. Where feasible, restrict read access to sensitive group memberships (e.g. `Domain Admins`) using a more restrictive ACL than the AD default, though this has real operational trade-offs and isn't a blanket recommendation for every environment.
3. Rotate service account passwords regularly and monitor for unusual Kerberos ticket requests, covered in more depth in the attack-focused write-up.
4. Treat "any domain user can enumerate the whole domain" as the expected baseline, not a bug, and build detection (logging, SIEM alerting on bulk enumeration patterns) around that assumption rather than trying to block it outright.

## 9. What I learned

- Simulated test data is not a substitute for testing against a real domain. All three of `ad_enum_v2.py`'s bugs were syntax/behavior assumptions that looked reasonable but were simply wrong, and none of them would have surfaced without running the tool against an actual DC.
- A command failing isn't always a bug. `gpresult` returning `Access Denied` for a non-admin user is the system working correctly, the fix was documenting that behavior, not forcing the command to succeed.
- AD enumeration doesn't require any special tooling or privilege to be meaningful. Built-in Windows commands, run by a completely ordinary user, already expose most of a domain's structure.
- Writing my own tooling instead of using an existing one meant I actually had to understand what each command returns and why, rather than treating a script as a black box.

## 10. References

- Microsoft Learn — [Install-ADDSForest](https://learn.microsoft.com/en-us/powershell/module/addsdeployment/install-addsforest)
- Microsoft Learn — [net accounts](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/net-accounts)
- Microsoft Learn — [nltest](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/nltest)
- My tools and their real test output: [AD-Enumeration repository](https://github.com/TheF0-ol/AD-Enumeration)
