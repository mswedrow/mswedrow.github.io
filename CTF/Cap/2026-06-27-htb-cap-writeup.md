---
layout: post
title: "HTB Cap - Writeup"
date: 2026-06-27
categories: [htb, linux]
tags: [idor, ftp, wireshark, linux-capabilities, setuid, privilege-escalation]
---

Cap is an easy-difficulty Linux machine hosting an HTTP server with network traffic capture functionality. An IDOR vulnerability exposes other users' captures, one of which contains plaintext FTP credentials. Those credentials are reused for SSH, granting initial access. Privilege escalation is achieved by abusing the `cap_setuid` Linux capability on the Python binary to spawn a root shell.

## Skills Required

- Web enumeration
- Linux fundamentals
- Wireshark
- Python

## Skills Learned

- Gobuster
- Ffuf
- getcap

---

## Enumeration

### Nmap

We start with an nmap port scan to identify open services:

![nmap scan](/assets/images/cap/Nmap.png)

Three ports are open: `21` (FTP), `22` (SSH), and `80` (HTTP). Task 2 directs us to the website on port 80, which is accessible at the IP address of the box on port 80.

![Dashboard](/assets/images/cap/Dashboard.png)

### IDOR — Accessing Another User's Capture

Clicking on the three-lined menu at the tope left of the site reveals a "Security Snapshot" feature.

![Sidebar](/assets/images/cap/Sidebar.png)

Running a scan produces a URL ending in `/data/1`, indicating scans are stored sequentially under `/data/`.

Since most applications index from 0, manually navigating to `/data/0` returns a different scan — one with significantly more packets captured, belonging to another user. This is a textbook **IDOR (Insecure Direct Object Reference)**: the application exposes sequential numeric identifiers in the URL without validating whether the requesting user has permission to access that record.

![Data](/assets/images/cap/Data_0.png)

---

### Wireshark — Credential Extraction

Download the `.pcap` from `/data/0` and open it in Wireshark. Filtering for FTP traffic surfaces a session with plaintext credentials:

| Field    | Value             |
| -------- | ----------------- |
| Username | `nathan`          |
| Password | `Buck3tH4TF0RM3!` |

![Pcap](/assets/images/cap/PcapFile.png)

FTP transmits credentials in plaintext — this is by design and a well-known weakness of the protocol.

---

## Foothold — SSH

The FTP credentials are reused for SSH:

```bash
ssh nathan@10.129.38.186
# password: Buck3tH4TF0RM3!
```

![SSH](/assets/images/cap/SSH.png)

From here, `user.txt` is accessible in Nathan's home directory.

![Flag](/assets/images/cap/Ls.png)

---

## Privilege Escalation

### Getcap — Identifying Misconfigured Capabilities

The `getcap` command can be used to identify which Linux capabilities are assigned to specific files or executables. Linux capabilities are a fine-grained alternative to blanket root privileges, designed to give processes only the permissions they need.

```bash
getcap -r / 2>/dev/null
```

The `-r /` flag is used to search recursively from the root directory, and the `2>/dev/null` removes all error messages to clean up the output.

![Getcap](/assets/images/cap/GetCap.png)

The `cap_setuid` capability allows a process to arbitrarily change its user ID — including to `0` (root).

### Python — Root Shell

Confirm Nathan can execute the binary:

```bash
ls -al /usr/bin/python2.8
```

![Permissions](/assets/images/cap/Ls-al.png)

Then abuse the capability:

```bash
/usr/bin/python2.8 -c "import os, pty; os.setuid(0); pty.spawn('/bin/bash')"
```

- `os.setuid(0)` — sets the process UID to root
- `pty.spawn('/bin/bash')` — spawns an interactive bash shell

![Python](/assets/images/cap/Python.png)

The prompt changes to `root@cap`, confirming full privilege escalation. `root.txt` is accessible under `/root/`.

![RootFlag](/assets/images/cap/Root.png)

---

## Analysis

**FTP/HTTP — no encryption.** Both protocols transmit data in plaintext. FTP credentials were captured via an exposed pcap. Even without this specific pcap, it would be trivial to passively capture and read FTP data on the network using Wireshark. SFTP and HTTPS exist precisely to address this.

**IDOR — broken access control.** The sequential `/data/<id>` URL scheme exposes an internal resource identifier (the specific ID of a security snapshot) directly to the client with no authorization check. IDOR is listed in the OWASP Top 10 under A01: Broken Access Control. The fix is server-side: validate that the requesting user is entitled to the resource before serving it.

**`cap_setuid` on Python — capability misconfiguration.** Linux capabilities are a subset of superuser privileges that are given to processes. They are primarily used to split the permissions needed from a process to only what is required. Ironically, this is done for security purposes, as traditional all-or-nothing `setuid` root binaries would require full superuser control if even one elevated permission was needed. The enhanced security design fails when capabilities are misconfigured, especially with a binary such as Python. The `cap_setuid` capability allows any user that runs the binary to change their uid and spawn a shell, completely bypassing security restrictions. For this box specifically, the `nathan` user had executable privileges for Python, which allowed the user to become root.

## Mitigations

- **Replace FTP/HTTP with SFTP/HTTPS.** Non-negotiable in any real environment, as information in transit must be encrypted.
- **Password Enforcement.** The password for the `nathan` account should not be the same for both FTP and SSH.
- **Fix the IDOR:** validate authorization server-side on every request; avoid exposing sequential numeric identifiers in URLs (Indirect Object Reference); use opaque identifiers (e.g. UUIDs) if references must be client-visible.
- **Remove `cap_setuid` from Python.** If the script legitimately needs elevated privileges, run it explicitly with `sudo` or as root, scoped to the specific script — not the interpreter globally. An alternative is `Cap_net_bind_service`, which allows a specific Python script to bind to a high-level service instead of possessing the ability to gain root privileges.

### Out-of-Scope Finding

The web server also exposes directory listings for `netstat` and `ip` endpoints, which output the results of the corresponding Linux commands — leaking IP addressing and socket connection data publicly. Directory indexing should be disabled and these endpoints removed or access-controlled.

![Gobuster](/assets/images/cap/Gobuster.png)
