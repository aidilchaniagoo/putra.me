---
author: Aidil Putra
date: 2025-12-20
summary:  |
  SSH tunnel port forwarding allows you to securely route network traffic through an SSH connection
title: "SSH Tunnel Port Forwarding"
tags:
  - "networking"
  - "linux"
---

## Overview

SSH tunnel port forwarding allows you to securely route network traffic through an SSH connection.
This is extremely useful when:

- A target network has **no gateway**
- Devices are **only reachable from another Linux host**

SSH supports **three major tunneling modes**:

- Local port forwarding (`-L`)
- Remote port forwarding (`-R`)
- Dynamic port forwarding / SOCKS proxy (`-D`)

This guide focuses on **Dynamic Port Forwarding (SOCKS proxy)** with practical examples.

---

## Run SSH Proxy

From your local machine, run in background:


```bash
ssh -f -N -D 1080 user@linux-host
```

Explanation:

- `-D 1080` → Open SOCKS5 proxy on localhost:1080
- `-f` → Run in background
- `-N` → No remote command execution

Once connected, your local machine has a SOCKS proxy at:

```bash
127.0.0.1:1080
```

## Using SSH Proxy

### Access internal devices using a browser

- Configure the browser using a SOCKS proxy (Browser → Settings):

```bash
- Proxy Type: SOCKS
- SOCKS Version: SOCKS5
- Host: 127.0.0.1
- Port: 1080
- Enable: Proxy DNS through SOCKS
```

- Access Device
```
http://<device-ip>
https://<device-ip>
```

---

### Access internal devices using SSH SOCKS proxy

SSH into another host behind Linux:

```bash
ssh -o "ProxyCommand=nc -x 127.0.0.1:1080 %h %p" user@internal-host
```

This allows chaining access without exposing routes.

---

### Perform APT using SSH SOCKS proxy

- Temporary configuration using single command

```bash
sudo apt -o Acquire::http::Proxy="socks5h://127.0.0.1:1080" update
```

- Persistent configuration by adding a new apt.conf.d file

Create file:

```bash
sudo vim /etc/apt/apt.conf.d/99socks
```

Content:

```bash
Acquire::http::Proxy "socks5h://127.0.0.1:1080";
Acquire::https::Proxy "socks5h://127.0.0.1:1080";
```
---

### Perform DNF using SSH SOCKS proxy

- Modify file `/etc/dnf/dnf.conf`

```bash
vim /etc/dnf/dnf.conf
```

- Add this configuration at the bottom of dnf.conf

```bash
proxy=socks5h://localhost:1080
```

- Test dnf command
```bash
sudo -E dnf install curl
```
---

## Conclusion

SSH tunnel port forwarding is a powerful, secure, and reversible solution for accessing isolated networks.
It avoids permanent routing changes while giving full Layer-7 access through a trusted Linux host.
