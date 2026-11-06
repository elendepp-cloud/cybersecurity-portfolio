# HTB Touch — Windows Privilege Escalation via MySQL UDF

## Machine Information

| Property | Details |
|---|---|
| Platform | Hack The Box |
| Machine | Touch |
| Operating System | Windows |
| Target Domain | `touch.htb` |
| Difficulty | Easy |
| Main Focus | Windows Privilege Escalation |
| Primary Technique | MySQL User-Defined Function (UDF) |
| Tools | Nmap, Burp Suite, RDP, PowerShell, MySQL, Kali Linux |

---

## 1. Overview

This write-up documents my research and practical investigation of the HTB Touch machine.

The main objective was to identify the attack surface, investigate the exposed web application, obtain an initial foothold, and understand how a misconfigured MySQL service could potentially lead to Windows privilege escalation.

The most important security concept explored in this machine was **Service-Based Windows Privilege Escalation through MySQL UDF**.

The investigation covered:

- Network enumeration and service identification.
- Web application and API investigation.
- Information disclosure.
- Initial access to the Windows environment.
- Local enumeration.
- MySQL privilege analysis.
- Investigation of the MySQL plugin directory.
- Understanding DLL-based User-Defined Functions.
- Verification of command execution under `NT AUTHORITY\SYSTEM`.

**Important status note:** Execution under `NT AUTHORITY\SYSTEM` was demonstrated. However, the final contents of `root.txt` were not successfully retrieved and validated. Therefore, this report does not claim a fully verified root flag.

---

# 2. Initial Enumeration

The first stage was identifying exposed services and understanding the target's attack surface.

### Nmap scan

The following command was used for service discovery:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

The scan identified the following open ports:

| Port | Service | Description |
|---|---|---|
| 135 | MSRPC | Microsoft RPC |
| 3389 | RDP | Remote Desktop Protocol |
| 5985 | HTTP | Windows Remote Management |
| 8443 | HTTP | Nexion DeviceHub web application |

### Initial observations

Several details were immediately interesting:

- The target was running Windows.
- RDP was exposed, suggesting a possible remote interactive access path.
- Port 5985 indicated that Windows Remote Management was available.
- Port 8443 hosted a web application called **Nexion DeviceHub**.

Rather than immediately attempting exploitation, I focused on understanding the application and its exposed functionality.

---

# 3. Web Application Enumeration — Port 8443


<img width="1920" height="1140" alt="Screenshot 2026-10-27 185321" src="https://github.com/user-attachments/assets/202ab7d5-3c12-4601-b657-43f922c2c980" />

Opening the service on port 8443 revealed a login interface for:

**Nexion DeviceHub**

The application appeared to be related to device management.

The initial investigation focused on:

- The login page.
- Available HTTP methods.
- Application endpoints.
- API responses.
- Information exposed without authentication.

### API investigation

During the investigation, the `/api/status` endpoint was examined.

The endpoint exposed device-related information, including a device serial number:

```text
NX-DH-2024-B7042
```

This was an important discovery because it provided information that could potentially be useful when investigating the application's authentication mechanism.

The investigation also identified the `/evox/about` endpoint, which returned XML-related information.

### Security observations

<img width="1920" height="1140" alt="Screenshot 2026-10-27 194525" src="https://github.com/user-attachments/assets/4625b46f-76ca-45dc-a251-b1e9e046c768" />


The application demonstrated why API enumeration is important during penetration testing.

An application may protect its main interface with authentication while exposing additional information through separate API endpoints.

Information disclosure does not always provide direct access, but it can reveal:

- Internal device information.
- Application architecture.
- Identifiers.
- Potentially sensitive configuration details.

In this case, the exposed serial number became relevant to the subsequent authentication investigation.

---

# 4. Initial Access Investigation

The next stage was examining how the application handled authentication.

During the investigation, the device serial number was found to be relevant to the authentication process.

The serial number was tested as part of the authentication investigation, and the following credentials were identified in the lab:

```text
Username: KioskUser
Password: K!0sk2026#
```

These credentials provided access to the DeviceHub environment.

### Why this mattered

The important security issue was not simply that a password existed.

The investigation demonstrated how information exposed through an unauthenticated API could become useful during authentication testing.

This is an example of how multiple individually small weaknesses can contribute to a larger attack chain.

---

# 5. Windows Initial Foothold

After obtaining the application credentials, the investigation moved toward accessing the underlying Windows environment.

The environment provided a restricted kiosk-style experience.

A kiosk configuration is designed to limit what a user can access. Instead of providing a normal desktop experience, it generally exposes only a limited application or interface.

However, kiosk restrictions must be implemented carefully.

In this environment, the investigation identified a way to reach a Windows command interface from the restricted environment.

The resulting access was associated with the following account:

```text
kiosk-042\kioskuser
```

This represented the initial Windows foothold.

### Why this stage was important

Obtaining access to a Windows application does not automatically mean that the underlying operating system is fully accessible.

The next step was therefore local enumeration: understanding the user's permissions, available services, installed software, and potentially sensitive configuration files.

---

# 6. Local Windows Enumeration

After obtaining the initial foothold, I investigated the local Windows environment.

The main objective was to identify services and configuration weaknesses that could provide a path to higher privileges.

The investigation focused on:

- Running services.
- Installed software.
- MySQL configuration.
- File and directory permissions.
- Stored credentials.
- Service execution context.

### MySQL discovery

A local MySQL service was identified.

The service was listening on:

```text
127.0.0.1:3306
```

This indicated that MySQL was accessible locally rather than being directly exposed as a remote service.

The investigation also identified the MySQL installation directory:

```text
C:\MySQL\bin
```

And the plugin directory:

```text
C:\MySQL\lib\plugin\
```

These locations became important during the privilege-escalation investigation.

---

# 7. MySQL Credential Discovery

During local file enumeration, a batch file named:

```text
refresh-dates.bat
```

was investigated.

The file contained MySQL credentials in plaintext.

The credentials identified during the lab investigation were:

```text
Username: root
Password: HTB@irw4ys_DB!2026
```

This provided access to the local MySQL instance.

### Security significance

Storing database credentials in plaintext inside scripts is dangerous.

If an attacker obtains access to the system and can read these files, the credentials may provide access to services with additional privileges.

This demonstrates why credential discovery is an important part of post-exploitation enumeration.

**Finding:** MySQL credentials were exposed in a local batch file.

---

# 8. MySQL Privilege Enumeration

After obtaining MySQL access, I investigated the privileges assigned to the database account.

The following query was used:

```sql
SELECT user, host, Grant_priv, File_priv
FROM mysql.user;
```

The investigation identified the MySQL `root` account with elevated database privileges, including:

```text
Grant_priv = Y
File_priv  = Y
```

### Understanding these privileges

**Grant_priv**

This privilege relates to the ability to grant privileges to other MySQL accounts.

**File_priv**

This privilege is associated with certain file operations performed through MySQL.

It is important to understand that MySQL privileges and Windows permissions are separate security layers.

For example:

- MySQL `root` is not the same as Windows `Administrator`.
- MySQL `FILE` does not automatically provide unrestricted access to the Windows filesystem.
- The actual Windows permissions available to MySQL depend on the account running the MySQL process.

This distinction became central to understanding the privilege-escalation mechanism.

---

# 9. Investigating the MySQL Plugin Directory

The next step was identifying where MySQL loads external libraries.

The following query was used:

```sql
SHOW VARIABLES LIKE 'plugin_dir';
```

The returned path was:

```text
C:\MySQL\lib\plugin\
```

### Why is this directory important?

MySQL supports external extensions, including User-Defined Functions.

On Windows, these extensions can be provided through DLL files.

The plugin directory is therefore an important location because MySQL uses it to locate supported plugin and UDF libraries.

During the local enumeration, the permissions of the MySQL installation and plugin-related directories were also investigated.

The directory permissions were identified as a relevant part of the attack surface.

**Security finding:** The investigation indicated that the MySQL installation and plugin directory had overly permissive access settings.

Such permissions can create a risk if unauthorized users are able to modify libraries that a privileged service later loads.

---

# 10. Understanding MySQL UDF

The main privilege-escalation concept investigated in this machine was **MySQL User-Defined Functions (UDF)**.

UDF stands for:

**User-Defined Function**

MySQL provides built-in functions, but it also supports additional functions through external libraries.

On Windows, these libraries commonly use the `.dll` extension.

DLL stands for:

**Dynamic Link Library**

A DLL contains code that can be loaded and used by another process.

A MySQL UDF allows MySQL to call a function provided by an external library.

### Why can this become dangerous?

A UDF is not inherently malicious or vulnerable.

It becomes a security concern when several conditions exist together:

1. An attacker has sufficient MySQL privileges.
2. MySQL is allowed to load an external library.
3. The required library is accessible to MySQL.
4. The MySQL service runs under a highly privileged Windows account.

The most important detail is that the UDF function executes within the MySQL server process.

Therefore, its execution context is tied to the security context of that process.

---

# 11. Understanding the Windows Service Context

This was the most important technical concept in the privilege-escalation investigation.

Windows services run under specific accounts.

For example, a service may run under:

- Local Service.
- Network Service.
- A dedicated service account.
- Local System.

The account determines the permissions available to the service process.

If MySQL runs under:

```text
NT AUTHORITY\SYSTEM
```

then the MySQL process has SYSTEM-level privileges.

When a UDF function is executed inside the MySQL process, it can perform operations using that process's security context.

This creates the possibility of executing operating-system commands with the privileges of the MySQL service.
<img width="1920" height="1140" alt="Screenshot 2026-11-04 132555" src="https://github.com/user-attachments/assets/c97a0e3b-f31e-4308-ad2b-3d83c0f5250d" />

### The important distinction

The UDF does not magically grant SYSTEM privileges.

Instead, it provides a mechanism for executing code inside a process that may already have SYSTEM privileges.

This is why checking the Windows service account is essential during privilege-escalation analysis.

---

# 12. UDF Execution and SYSTEM Verification

During the investigation, a UDF function named `sys_eval` was used to investigate operating-system command execution through MySQL.

The function registration syntax examined was:

```sql
CREATE FUNCTION sys_eval
RETURNS STRING
SONAME 'lib_mysqludf_sys_64.dll';
```

The function was then used to check the Windows execution identity.

The identity check was based on:

```sql
SELECT sys_eval('whoami');
```

The resulting identity was:

```text
nt authority\system
```

### What does this prove?

This result demonstrates that the command was executed in the Windows SYSTEM security context.

It is strong evidence that the MySQL UDF mechanism provided operating-system command execution with elevated privileges.

### Why is `whoami` useful?

The `whoami` command identifies the Windows account associated with the current process.

It is a simple way to verify the execution context without immediately attempting more sensitive operations.

**Confirmed result:** The investigation demonstrated command execution as `NT AUTHORITY\SYSTEM`.

---

# 13. Root Flag Investigation

After confirming SYSTEM-level command execution, the next objective was investigating the Administrator desktop and the root flag.

The following path was examined:

```text
C:\Users\Administrator\Desktop\
```

The directory listing was investigated, and `root.txt` was identified as a file of approximately 34 bytes.

The file contents were then investigated through the MySQL UDF command-execution mechanism.

During this stage, output encoding and retrieval issues occurred.

A hexadecimal result was decoded and submitted as a possible flag:

```text
32fdb3e754e94ca88e7eea1105531874
```

However, Hack The Box rejected this value.

### Final status

The following distinction is important:

| Objective | Status |
|---|---|
| Identify the MySQL service | Achieved |
| Discover MySQL credentials | Achieved |
| Investigate MySQL privileges | Achieved |
| Identify the plugin directory | Achieved |
| Investigate the UDF mechanism | Achieved |
| Demonstrate SYSTEM command execution | Confirmed |
| Identify `root.txt` in the Administrator desktop | Observed during investigation |
| Retrieve and validate the correct root flag | Not completed |

I do not consider the root flag captured because the submitted value was rejected.

This report therefore documents the confirmed privilege-escalation research without claiming an unverified final result.

---

# 14. Classification of the Privilege Escalation

This technique can be classified as:

**Service-Based Windows Privilege Escalation via MySQL UDF**

It is not a standalone built-in Windows privilege-escalation category such as:

- AlwaysInstallElevated.
- Unquoted Service Path.
- Weak Service Permissions.

Instead, it is a privilege-escalation chain involving a third-party Windows service.

The underlying weakness is the combination of:

- Excessive MySQL privileges.
- Unsafe DLL/plugin handling.
- Overly permissive directory permissions.
- A MySQL service running with high Windows privileges.

The UDF mechanism allows code to execute inside the MySQL process, potentially inheriting its elevated execution context.

---

# 15. Defensive Recommendations

Several defensive measures can reduce the risk of this attack.

### 1. Apply the Principle of Least Privilege

MySQL should not run as `NT AUTHORITY\SYSTEM` unless there is a specific and justified requirement.

A dedicated, low-privileged service account should be used.

### 2. Restrict MySQL Privileges

Administrative SQL privileges should only be assigned to accounts that genuinely require them.

The `FILE` privilege should also be restricted.

### 3. Protect the Plugin Directory

The MySQL plugin directory should not be writable by ordinary users.

Only authorized administrators should be able to modify DLL libraries loaded by MySQL.

### 4. Monitor UDF Registration

Security monitoring should detect unexpected UDF creation and the loading of unfamiliar external libraries.

### 5. Monitor Child Processes

Unexpected processes started by MySQL may indicate suspicious activity.

For example, a database service launching a command interpreter should be investigated.

### 6. Protect Credentials

Passwords should not be stored in plaintext scripts or configuration files.

Secrets should be stored securely and rotated when exposure is suspected.

### 7. Keep Software Updated

MySQL and the Windows operating system should be regularly updated and securely configured.

---

# 16. Key Lessons Learned

This investigation demonstrated several important penetration-testing concepts.

**1. Enumeration is more important than immediately exploiting a service.**

The exposed web application and API provided useful information that influenced the next stages of the investigation.

**2. Information disclosure can contribute to initial access.**

An exposed device identifier may become relevant when investigating authentication mechanisms.

**3. Local enumeration is essential after obtaining a foothold.**

The MySQL installation, local credentials, service configuration, and directory permissions revealed a possible privilege-escalation path.

**4. Database privileges are different from operating-system privileges.**

MySQL root does not automatically mean Windows Administrator or SYSTEM.

**5. The execution context of a service matters.**

Code executed inside a privileged service process may inherit the security context of that process.

**6. A successful command-execution test is not the same as completing the entire machine.**

SYSTEM execution was demonstrated, but the final root flag was not successfully validated.

---

# 17. Conclusion

The most important finding in this investigation was the relationship between MySQL UDF execution and the Windows service security context.

The privilege-escalation mechanism can be summarized as follows:

```text
MySQL access
     |
     v
MySQL privilege investigation
     |
     v
External UDF library mechanism
     |
     v
Code execution inside MySQL process
     |
     v
MySQL service security context
     |
     v
NT AUTHORITY\SYSTEM
```

The key lesson is that a service running with excessive privileges can turn an otherwise legitimate application feature into a serious security risk.

This investigation improved my understanding of:

- Windows services and their execution contexts.
- MySQL privileges.
- DLL loading and UDF functionality.
- Windows local enumeration.
- Service-based privilege escalation.
- The difference between demonstrating SYSTEM execution and proving complete machine compromise.

**Final assessment:** SYSTEM-level command execution was demonstrated through the MySQL UDF mechanism. The final root flag remains unverified.
