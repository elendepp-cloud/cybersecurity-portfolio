# Nexus — Hack The Box Write-up

**Platform:** Hack The Box  
**Machine:** Nexus  
**Operating System:** Linux
**Difficulty:** meidl 
**Status:** In Progress  
**Current Progress:** User access obtained — Root flag remaining

---

## 1. Overview

This write-up documents my hands-on experience solving the Nexus machine on Hack The Box.

The objective is to analyze the target, enumerate exposed services, identify vulnerabilities, obtain initial access, and investigate privilege escalation opportunities.

The machine is currently in progress. Initial access has been achieved, while root-level access remains to be completed.

## 2. Reconnaissance

### Target Information

- Target IP: `REDACTED`
- Platform: Hack The Box
- Operating System: To be documented

### Initial Network Scan

**Tool:** Nmap

```bash
#nmap -sC -sV -Pn TARGET-IP
```

### Findings

| Port | Service |
|---|---|---|---|
| 80/tcp | http |
| 22/tcp | ssh |
## 3. Enumeration

Document the discovered services and potential attack surface.

### Service Enumeration

- Open ports:80,22
- Running services:http,ssh 
- Service versions:-<img width="1536" height="912" alt="image" src="https://github.com/user-attachments/assets/01e0205a-4e95-4c33-a830-d3e9135330f7" />

- Interesting findings:-

### Additional Enumeration

Document relevant commands, outputs, and observations.

## 4. Initial Access

**Status:** Completed

Document the actual steps that led to initial access.

### Methodology

- Identified entry point:
- Vulnerability or weakness:
- Exploitation steps:
- Access obtained:

### Evidence

Add relevant terminal output and screenshots, with sensitive information redacted.

## 5. User-Level Access

**Status:** Completed

Document how user-level access was obtained.

- User account:
- Access method:
- User flag: Obtained

## 6. Privilege Escalation

**Status:** In Progress

The objective of this stage is to investigate possible privilege escalation paths and obtain root-level access.

### Enumeration

Document:

- User permissions
- Sudo privileges
- Running processes
- Scheduled tasks
- SUID binaries
- Interesting files and configurations

### Investigation Notes

```bash
# Add commands and actual findings here
```

### Root Access

**Status:** Not yet obtained.

The privilege escalation process is still ongoing.

## 7. Flags & Results

| Objective | Status |
|---|---|
| Initial Access | Completed |
| User Flag | Obtained |
| Root Flag | Pending |
| Machine Completion | In Progress |

## 8. Tools Used

Add only tools actually used during the investigation.

- Nmap
- To be documented

## 9. Key Learnings

To be completed after finishing the machine.

## 10. Conclusion

The Nexus machine provided practical experience in reconnaissance, enumeration, initial access, and privilege escalation investigation.

The machine remains in progress until root-level access is obtained.

---

**Disclaimer:** This write-up documents work performed in an authorized Hack The Box environment.
