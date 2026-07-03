

A complete reference covering the OSI Model, TCP/IP, DNS, HTTP/HTTPS, and SSH — plus practical commands for curl, jq, and Netcat.

---

## Table of Contents
1. [The OSI Model](#1-the-osi-model)
2. [TCP/IP](#2-tcpip)
3. [DNS](#3-dns)
4. [HTTP/HTTPS](#4-httphttps)
5. [SSH](#5-ssh)
6. [Tools: curl, jq, Netcat](#6-tools)
7. [Quick Cheat Sheet](#7-quick-cheat-sheet)

---

## 1. The OSI Model

The **OSI (Open Systems Interconnection) Model** is a 7-layer conceptual framework describing how data travels from one device to another across a network. It's a *reference model* — real-world networking (like TCP/IP) doesn't map perfectly onto it, but it's the standard vocabulary used to talk about networking problems.

**Mnemonic (bottom to top):** *"Please Do Not Throw Sausage Pizza Away"*

| Layer # | Name | Function | Examples / Units |
|---|---|---|---|
| 7 | **Application** | Interface for user-facing network services | HTTP, FTP, DNS, SMTP — data |
| 6 | **Presentation** | Data formatting, encryption, compression | SSL/TLS, JPEG, ASCII — data |
| 5 | **Session** | Establishes, manages, and ends sessions between apps | APIs, sockets — data |
| 4 | **Transport** | Reliable/unreliable end-to-end delivery, error checking | TCP, UDP — segments |
| 3 | **Network** | Logical addressing & routing between networks | IP, routers — packets |
| 2 | **Data Link** | Physical addressing, frames between directly connected nodes | MAC address, switches — frames |
| 1 | **Physical** | Actual transmission of raw bits over a medium | Cables, radio waves, hubs — bits |

**How to think about it:** Data going *down* the stack on the sender's side gets wrapped with headers at each layer (**encapsulation**); on the receiver's side, each layer unwraps its corresponding header going *up* the stack (**decapsulation**).

**Why it matters practically:**
- Troubleshooting: "Is this a Layer 1 issue (cable unplugged), Layer 3 issue (routing/IP), or Layer 7 issue (the app itself)?"
- Tools map to layers: `ping` tests Layer 3, `curl` operates at Layer 7, a switch operates at Layer 2.

---

## 2. TCP/IP

The **TCP/IP model** is the practical 4-layer model the real internet actually runs on (OSI is theoretical; TCP/IP is what's implemented).

| TCP/IP Layer | Roughly maps to OSI | Protocols |
|---|---|---|
| Application | 5, 6, 7 | HTTP, DNS, SSH, FTP |
| Transport | 4 | TCP, UDP |
| Internet | 3 | IP, ICMP |
| Network Access (Link) | 1, 2 | Ethernet, Wi-Fi |

### 2.1 IP (Internet Protocol)
- Responsible for **addressing and routing** packets between devices.
- **IPv4:** 32-bit address, written as 4 octets, e.g. `192.168.1.10` (~4.3 billion addresses).
- **IPv6:** 128-bit address, e.g. `2001:0db8::1` — created because IPv4 addresses ran out.
- **Public IP:** reachable directly on the internet. **Private IP:** used inside a local network (e.g. `192.168.x.x`, `10.x.x.x`, `172.16.x.x–172.31.x.x`).

### 2.2 TCP vs UDP (Transport Layer)

| | **TCP** | **UDP** |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, ordered, error-checked | Best-effort, no guarantee |
| Speed | Slower (overhead of reliability) | Faster (minimal overhead) |
| Use cases | Web browsing, email, file transfer, SSH | Streaming, DNS, VoIP, gaming |

**TCP 3-Way Handshake** (how a TCP connection is established):
```
Client → SYN        → Server     (I want to connect)
Client ← SYN-ACK     ← Server     (OK, acknowledged)
Client → ACK          → Server     (Connection established)
```

### 2.3 Ports
A **port** identifies a specific service/process on a device (IP address = the building, port = the apartment number).

| Port | Service |
|---|---|
| 20/21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 25 | SMTP (email sending) |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 3389 | RDP |

- **Well-known ports:** 0–1023 (reserved for standard services)
- **Registered ports:** 1024–49151
- **Dynamic/private ports:** 49152–65535 (used for outgoing client connections)

### 2.4 Basic Networking Commands
```bash
ip a                     # show IP addresses/interfaces (modern)
ifconfig                   # show IP addresses/interfaces (older/legacy)
ping google.com              # test connectivity (Layer 3)
traceroute google.com          # show the path/hops packets take
netstat -tulnp                   # show listening ports and services
ss -tulnp                          # modern replacement for netstat
```

---

## 3. DNS

**DNS (Domain Name System)** translates human-readable domain names (`google.com`) into IP addresses (`142.250.190.14`) — it's essentially the "phonebook of the internet."

### 3.1 How DNS Resolution Works
```
You type google.com
   ↓
Browser checks local cache
   ↓
Query sent to Recursive Resolver (usually your ISP or 8.8.8.8)
   ↓
Resolver asks a Root Nameserver → "who handles .com?"
   ↓
Root refers to TLD Nameserver (.com)
   ↓
TLD refers to Authoritative Nameserver for google.com
   ↓
Authoritative server returns the IP address
   ↓
Resolver caches & returns IP to your browser
```

### 3.2 Common DNS Record Types

| Record | Purpose |
|---|---|
| **A** | Maps a domain to an IPv4 address |
| **AAAA** | Maps a domain to an IPv6 address |
| **CNAME** | Alias — points one domain to another domain name |
| **MX** | Mail exchange — where email for the domain should go |
| **NS** | Nameserver — which servers are authoritative for the domain |
| **TXT** | Arbitrary text — often used for verification, SPF/DKIM (email security) |
| **PTR** | Reverse lookup — IP address back to domain name |

### 3.3 DNS Commands
```bash
nslookup google.com           # basic DNS lookup
dig google.com                   # detailed DNS lookup (preferred by admins)
dig google.com MX                  # look up a specific record type
dig +short google.com                # concise output — just the IP
host google.com                        # simple name-to-IP lookup
cat /etc/resolv.conf                     # see which DNS servers your machine uses
cat /etc/hosts                             # local hostname-to-IP overrides (checked before DNS)
```

---

## 4. HTTP/HTTPS

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol used to transfer web content (HTML, JSON, images) between clients (browsers) and servers. **HTTPS** is HTTP secured with **TLS/SSL encryption**.

### 4.1 How It Works
1. Client sends an HTTP **request** (method + URL + headers + optional body).
2. Server processes it and sends back an HTTP **response** (status code + headers + body).

### 4.2 HTTP Methods

| Method | Purpose |
|---|---|
| `GET` | Retrieve data (no body) |
| `POST` | Submit/create new data |
| `PUT` | Update/replace a resource entirely |
| `PATCH` | Partially update a resource |
| `DELETE` | Remove a resource |
| `HEAD` | Like GET, but headers only (no body) |
| `OPTIONS` | Discover allowed methods on a resource |

### 4.3 HTTP Status Codes

| Range | Meaning | Examples |
|---|---|---|
| 1xx | Informational | 100 Continue |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 302 Found |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

### 4.4 HTTP vs HTTPS

| | HTTP | HTTPS |
|---|---|---|
| Port | 80 | 443 |
| Encryption | None (plaintext) | TLS/SSL encrypted |
| Data integrity | Vulnerable to tampering/eavesdropping | Protected |
| Certificate | Not required | Requires an SSL/TLS certificate |

**TLS Handshake (simplified):** client and server agree on encryption method → server proves identity via certificate → both sides generate a shared session key → encrypted communication begins.

### 4.5 Common Headers
```
Content-Type: application/json        # format of the body
Authorization: Bearer <token>           # auth credentials
User-Agent: ...                           # identifies the client
Cache-Control: no-cache                     # caching rules
Set-Cookie: session=abc123                    # server sets a cookie
```

---

## 5. SSH

**SSH (Secure Shell)** is an encrypted protocol used to securely log into and manage remote machines over an unsecured network. It replaced insecure protocols like Telnet, which sent data (including passwords) in plaintext.

### 5.1 How SSH Works (High-Level)
1. Client contacts server on **port 22**.
2. Server presents its **host key**; client verifies it (or trusts it on first connect — stored in `~/.ssh/known_hosts`).
3. A secure encrypted channel is negotiated.
4. Authentication happens via **password** or (preferably) **public/private key pair**.

### 5.2 Key-Based Authentication
- **Private key** — stays secret on your machine (`id_rsa` / `id_ed25519`), never shared.
- **Public key** — copied to the server (`~/.ssh/authorized_keys`), safe to share.
- The server encrypts a challenge with your public key; only your private key can decrypt it — proving your identity without ever sending a password.

### 5.3 SSH Commands
```bash
ssh user@host                       # connect to a remote server
ssh -p 2222 user@host                 # connect using a non-default port
ssh -i ~/.ssh/mykey user@host           # connect using a specific private key

ssh-keygen -t ed25519                     # generate a new SSH key pair (modern, preferred)
ssh-keygen -t rsa -b 4096                   # generate an RSA key pair (older, still common)

ssh-copy-id user@host                         # copy your public key to a remote server (enables key login)

scp file.txt user@host:/remote/path/           # securely copy a file TO a remote server
scp user@host:/remote/file.txt ./                # securely copy a file FROM a remote server
scp -r folder/ user@host:/remote/path/             # copy a directory recursively

ssh user@host "ls -la"                       # run a single remote command without logging in fully

ssh -L 8080:localhost:80 user@host             # local port forwarding (tunnel)

cat ~/.ssh/id_ed25519.pub                        # view your public key
```

### 5.4 Config Shortcut
`~/.ssh/config` lets you save connection details:
```
Host myserver
    HostName 192.168.1.50
    User john
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```
Then simply: `ssh myserver`

---

## 6. Tools

### 6.1 curl
Transfers data to/from a server via URLs — used for testing APIs and web services (HTTP layer).
```bash
curl https://example.com                     # fetch a URL
curl -o file.html https://example.com          # save output to a file
curl -O https://example.com/file.zip             # save with original filename
curl -I https://example.com                        # headers only (HEAD request)
curl -L https://example.com                          # follow redirects
curl -X POST -d "name=John" https://api.com             # send POST data
curl -H "Content-Type: application/json" -d '{"a":1}' -X POST https://api.com   # send JSON
curl -H "Authorization: Bearer TOKEN" https://api.com     # auth token
curl -u user:pass https://example.com                        # basic auth
curl -v https://example.com                                    # verbose (debug handshake/headers)
curl -s https://example.com                                      # silent (no progress bar)
curl -k https://self-signed.example.com                             # ignore SSL cert errors
```

### 6.2 jq
Parses and filters JSON — usually piped from `curl`'s API responses.
```bash
curl -s https://api.com | jq .            # pretty-print JSON response
jq '.name' file.json                        # get a specific field
jq '.[]' file.json                            # iterate array elements
jq '.users[].name' file.json                    # extract field from every item
jq 'length' file.json                             # count elements
jq 'select(.age > 18)' file.json                    # filter by condition
jq -r '.name' file.json                               # raw output (no quotes)
```

### 6.3 Netcat (nc)
Raw TCP/UDP utility — used to test if ports/services are reachable (Transport layer).
```bash
nc -zv host 22                    # check if a single port is open
nc -zv host 20-100                  # scan a range of ports
nc -l -p 1234                         # listen on a port
nc host 1234                            # connect to a listening host/port
echo "hello" | nc host 1234                # send a quick message
nc -u host 1234                              # use UDP instead of TCP
nc -w 3 host 1234                              # set connection timeout
nc host 80 <<< "GET / HTTP/1.0"                  # manual HTTP request (banner grabbing)
```

---

## 7. Quick Cheat Sheet

| Concept | Key Command / Fact |
|---|---|
| OSI Model | 7 layers: Physical → Data Link → Network → Transport → Session → Presentation → Application |
| TCP/IP Model | 4 layers: Network Access → Internet → Transport → Application |
| TCP handshake | SYN → SYN-ACK → ACK |
| DNS lookup | `dig google.com` / `nslookup google.com` |
| Check listening ports | `ss -tulnp` |
| HTTP vs HTTPS | Port 80 vs Port 443 (encrypted) |
| SSH connect | `ssh user@host` |
| SSH key auth | `ssh-keygen` then `ssh-copy-id user@host` |
| Test API endpoint | `curl -s URL \| jq .` |
| Test if port is open | `nc -zv host port` |

---

### Practice Suggestions for Day 2
1. Run `ping`, `traceroute`, and `dig` against a real domain and map each result to an OSI layer.
2. Use `curl -v` against a website and identify the TCP handshake, TLS handshake, and HTTP request/response in the verbose output.
3. Query different DNS record types (`A`, `MX`, `NS`, `TXT`) for a domain using `dig`.
4. Generate an SSH key pair, copy it to a test VM, and log in without a password.
5. Use `nc -zv` to scan a range of ports on your own VM and cross-check with `ss -tulnp`.
6. Hit a public JSON API with `curl`, pipe it through `jq` to extract a single field.
