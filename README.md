# TryHackMe: MD2PDF Detailed Walkthrough

<p align="center"><img width="200" height="200" alt="image" src="https://github.com/user-attachments/assets/660460ee-027d-468f-825b-6b9ebaf92ecd" /></p>

## Overview
Welcome to the detailed walkthrough for the **MD2PDF** room on TryHackMe. This machine demonstrates a critical vulnerability often found in document generation libraries. By carefully enumerating the application and analyzing the backend technologies, we will identify and exploit a **Server-Side Request Forgery (SSRF)** vulnerability to access restricted internal endpoints and retrieve the hidden flag.

**Difficulty:** Easy / Medium  
**Key Concepts:** Web Enumeration, Metadata Analysis, SSRF (Server-Side Request Forgery), Bypassing Access Controls.

---

## The Attack Path: A Visual Summary

For users new to this concept, here is a simplified breakdown of the vulnerability we will exploit:

| 1. The Goal | 2. The Block | 3. The Exploit (SSRF) |
| :--- | :--- | :--- |
| We want to see the `/admin` page. | The server blocks us because we are an **external** user. | We trick the server's PDF generator into requesting the page as an **internal** user. |
| `Hacker -> /admin` | `Status: 403 Forbidden` | `Hacker -> PDF Tool -> /admin -> PDF Document` |

---

## Phase 1: Reconnaissance and Enumeration

### 1. Port Scanning with Nmap
We begin our assessment by scanning the target machine to identify open ports and the services running on them. A thorough scan is the foundation of any penetration test.

```bash
nmap -sV -p- <TARGET_IP>
````
<img width="1919" height="1032" alt="Screenshot 2026-03-30 161702" src="https://github.com/user-attachments/assets/06f6c93e-6ce9-4b48-b812-6657a8e0b45e" />

*Command Breakdown:*

  * `-sV`: Probes open ports to determine service and version info.
  * `-p-`: Scans all 65,535 ports to ensure nothing is missed.

**Scan Analysis:**

  * **Port 22/tcp (SSH):** OpenSSH 8.2p1 Ubuntu. Standard secure shell access. We likely need credentials to use this.
  * **Port 80/tcp (HTTP):** A standard web server hosting our primary target application.
  * **Port 5000/tcp (HTTP):** A secondary web service. This is highly suspicious and warrants further investigation.

### 2\. Web Application Analysis (Port 80)

Navigating to `http://<TARGET_IP>` on port 80 reveals a clean, simple web interface titled "MD2PDF".

<img width="1919" height="1079" alt="Screenshot 2026-03-30 161745" src="https://github.com/user-attachments/assets/e8a901ca-d657-4777-adb2-2f2c9d92ed28" />

The application provides a text area where users can input Markdown-formatted text. Clicking the "Convert to PDF" button processes the input and generates a viewable and downloadable PDF document.

When we input basic Markdown (like `# This is MD2PDF` and `### This is a subheading`) and convert it, the application successfully renders a PDF in the browser.

<img width="1919" height="1079" alt="Screenshot 2026-03-30 161752" src="https://github.com/user-attachments/assets/2d440e88-97b7-4be6-9257-87bace4e9f56" />

### 3\. Directory Enumeration

While the main functionality is obvious, web applications often hide administrative or unlinked directories. We use `gobuster` to brute-force the web directory structure on port 80.

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```
<img width="1919" height="1031" alt="Screenshot 2026-03-30 162829" src="https://github.com/user-attachments/assets/17b2bedd-d543-44cc-8e57-391b281d18a1" />

The scan discovers a highly interesting endpoint:

  * `/admin` (Status: 403 Forbidden) [Size: 166]

Attempting to access `http://<TARGET_IP>/admin` directly through our browser gives us a critical piece of information:

> **Forbidden**
> This page can only be seen internally (localhost:5000)

<img width="1919" height="1079" alt="Screenshot 2026-03-30 162839" src="https://github.com/user-attachments/assets/b9225b83-d031-407f-8b2b-53afae3334af" />

**Key Takeaway:** The `/admin` dashboard exists, but the web server's access controls explicitly block external IP addresses. It will only serve the page if the HTTP request originates from the server itself (`localhost` or `127.0.0.1`) targeting port `5000`.

-----

## Phase 2: Vulnerability Discovery

### 1\. Analyzing the PDF Generator Metadata

To compromise the machine, we need to understand *how* the application converts our Markdown/HTML into a PDF. Tools that render PDFs from HTML are notorious for vulnerabilities.

We download the sample PDF we generated earlier to our local Kali machine and examine its underlying metadata using `exiftool`.

```bash
exiftool md2pdf.pdf
```
<img width="1919" height="1031" alt="Screenshot 2026-03-30 162220" src="https://github.com/user-attachments/assets/56c3941f-64b6-40b2-bd16-b4b6e7348e3e" />

**The Smoking Gun:**
The `Creator` field in the metadata clearly states: **`wkhtmltopdf 0.12.5`**.

### 2\. Researching wkhtmltopdf 0.12.5

We turn to Google to research this specific software version. A quick search for `vulnerabilities wkhtmltopdf 0.12.5` yields immediate results.

<img width="1919" height="1079" alt="Screenshot 2026-03-30 162504" src="https://github.com/user-attachments/assets/aabe9da3-cc67-4c27-940b-a24fdef7b00c" />

This version is heavily outdated and suffers from severe security flaws, most notably **Server-Side Request Forgery (SSRF)** and **Local File Inclusion (LFI)**.

Because `wkhtmltopdf` processes the input as HTML to format the PDF via a web rendering engine, it interprets standard HTML tags. If we inject an `<iframe>` tag pointing to an external or internal URL, the backend server will attempt to fetch that URL and render its visual contents directly into the resulting PDF document.

-----

## Phase 3: Exploitation (SSRF)

### 1\. Initial Payload Testing

We first test if the application will fetch resources from its own local loopback address. We input the following HTML payload into the MD2PDF converter box:

```html
<iframe src="http://localhost/" width="1000" height="2000"></iframe>
```

<img width="1919" height="1079" alt="Screenshot 2026-03-30 162540" src="https://github.com/user-attachments/assets/672228ef-79e8-4eb4-ab91-cafcdc63a421" />

Submitting this payload returns an error:

The `400 Bad Request` indicates the application is either crashing or actively blocking standard requests to the root directory of `localhost`.

### 2\. Bypassing Restrictions to Read Internal Pages

We know from our Phase 1 enumeration that the `/admin` page is specifically looking for requests coming from `localhost:5000`.

We can weaponize our SSRF vulnerability to force the underlying server to make this exact request on our behalf. We update our payload in the converter to target the restricted endpoint:

```html
<iframe src="http://localhost:5000/admin" width="1000" height="2000"></iframe>
```

<img width="1919" height="1079" alt="Screenshot 2026-03-30 162951" src="https://github.com/user-attachments/assets/2fbe2463-eed4-4c6f-934f-67f93b4e2137" />

And we finally get the flag:

<img width="1919" height="1079" alt="Screenshot 2026-03-30 162956" src="https://github.com/user-attachments/assets/55f3ff26-bf23-439f-8d91-463250d0c8f6" />

**The Final Exploitation Chain:**

1.  We submit the malicious `<iframe>` payload.
2.  The web app passes our input to the vulnerable `wkhtmltopdf 0.12.5` backend.
3.  The engine parses the `<iframe>` tag and makes an HTTP GET request to `http://localhost:5000/admin`.
4.  Because this request originates directly from the server itself, the "internal only" access control restriction is completely bypassed.
5.  The server responds with the secret contents of the `/admin` page, which `wkhtmltopdf` captures and renders into the final PDF document.

Upon clicking "Convert to PDF" and viewing the resulting document, the previously hidden administration dashboard—and the room's flag—is visually displayed inside the PDF, completing the challenge.

-----

## Remediation & Mitigation

To secure this application, developers should:

1.  **Upgrade Software:** Immediately update `wkhtmltopdf` to the latest, patched version, or migrate to a more secure, modern PDF rendering library.
2.  **Input Sanitization:** Implement strict content filtering to strip dangerous HTML tags (like `<iframe>`, `<script>`, `<object>`, and `<link>`) before passing the input to the PDF renderer.
3.  **Network Isolation:** Run the document generation service in an isolated container or restricted network segment where it cannot access sensitive internal endpoints like `localhost:5000`.

### Credits 
#### Performed and written by Aadithya Vimal
#### Write-up inspired by Sakibul Ali Khan - <a href src="https://sakibulalikhan.medium.com/tryhackme-md2pdf-ctf-writeup-d3f254e2f16f">Sakibul Ali Khan - Medium Write up</a>
