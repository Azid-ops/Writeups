# TryHackMe – Intermediate Nmap Room Write‑Up

## Overview

This room focuses on using **Nmap** effectively and following up with basic enumeration to gain access to a target host.
Target IP used in this write‑up: `10.10.11.63`

---

## 1. Initial Nmap Scan

I started with an aggressive Nmap scan to detect open ports, services, versions, and OS information:

```bash
nmap -A 10.10.11.63
```

### Scan Results (Summary)

* **Host**: `10.10.11.63`
* **Status**: Host is up

**Open Ports:**

* `22/tcp` – SSH – `OpenSSH 8.2p1 Ubuntu 4ubuntu0.4`
* `2222/tcp` – SSH – `OpenSSH 8.2p1 Ubuntu 4ubuntu0.4`
* `31337/tcp` – Unknown service (labeled `Elite?`)

Nmap also attempted service fingerprinting on the unknown service running on port **31337**. The response from that port was very interesting:

```text
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

This is clearly a **credential leak** left on the custom service.

---

## 2. HTTP Enumeration Attempt

Before using the credentials, I tried to see if there was a web service running on port 80.

I ran **Gobuster** for directory enumeration:

```bash
gobuster dir -u http://10.10.11.63 -w /usr/share/wordlists/dirb/common.txt
```

However, Gobuster returned:

```text
error on running gobuster on http://10.10.11.63/: connection refused
```

This indicates that there is **no HTTP service** listening on port 80, so web enumeration is not useful in this case.

---

## 3. Enumerating Port 31337

The custom service on port `31337/tcp` responded with:

```text
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

This looks like the developer left credentials here "in case I forget" – a common misconfiguration in CTF-style machines.

From here, we can assume:

* **Username**: `ubuntu`
* **Password**: `Dafdas!!/str0ng`

These credentials are likely valid for SSH.

You can also manually confirm this by visiting:

```text
http://10.10.11.63:31337/
```

which shows the same user:pass information.

---

## 4. Gaining Access via SSH

Using the credentials found on port `31337`, I connected to the target over SSH:

```bash
ssh ubuntu@10.10.11.63
```

When prompted for a password, I entered:

```text
Dafdas!!/str0ng
```

The login was successful:

```text
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.13.0-1014-aws x86_64)
...
```

This gave me a shell as the user **ubuntu**.

---

## 5. Basic Post-Exploitation & Flag Retrieval

After gaining access, I listed the contents of the current directory:

```bash
ls
```

No interesting files appeared there, so I checked the home directories:

```bash
ls -la /home
```

Output:

```text
drwxr-xr-x 1 root   root   4096 Mar  2  2022 .
drwxr-xr-x 1 root   root   4096 Mar  2  2022 ..
drwxr-xr-x 1 ubuntu ubuntu 4096 Nov 16 10:25 ubuntu
drwxr-xr-x 2 root   root   4096 Mar  2  2022 user
```

There is a second home directory `/home/user`, which looks promising. I moved into it:

```bash
cd /home/user
ls
```

Output:

```text
flag.txt
```

Finally, I read the flag:

```bash
cat flag.txt
```

This displayed the room’s flag.

---

## 6. Lessons Learned

1. **Nmap’s service detection can leak credentials**
   The service on port `31337` revealed `ubuntu:Dafdas!!/str0ng` directly in the banner. Always carefully read Nmap output, especially for unusual / unknown services.

2. **Not every box has a web service**
   Gobuster failed because there was no HTTP service on port 80. It’s important to base further enumeration on what ports are actually open.

3. **Custom high ports are often interesting**
   Ports like `31337` are commonly used in CTFs for backdoors or custom services. Always investigate them thoroughly.

4. **Check all home directories**
   Even if you log in as `ubuntu`, flags or sensitive data might be stored in another user’s home directory (e.g. `/home/user/flag.txt`).

---
