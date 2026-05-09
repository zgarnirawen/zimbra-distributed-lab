# Zimbra 10.1.3 Distributed Mail Server Deployment

**Authors:** Rawen ZGARNI & Warda GUESMI  
**Institution:** ENICarthage, Tunisia  
**Academic Year:** 2025/2026  
**Date:** May 2026

A production-grade distributed Zimbra 10.1.3 (Daffodil) mail server infrastructure deployed across 3 VMware virtual machines running Rocky Linux 9.7. This project demonstrates enterprise mail architecture with separated LDAP, MTA/Proxy, and Mailbox roles.

---

## 🏗️ Architecture Overview

### Three-Node Distributed Configuration

| VM | Hostname | Role | IP Address | RAM | Services |
|---|---|---|---|---|---|
| **VM1-LDAP** | ldap.mail.lab | Directory Services | 192.168.232.10 | 2 GB | ldap, stats, zmconfigd |
| **VM2-MTA** | mta.mail.lab | MTA + Proxy Gateway | 192.168.232.11 | 3 GB | amavis, antispam, antivirus, logger, memcached, mta, opendkim, proxy, snmp, stats, zmconfigd |
| **VM3-Mailbox** | store.mail.lab | Mailbox + Admin | 192.168.232.12 | 8 GB | mailbox, memcached, service webapp, spell, stats, zimbra webapp, zimbraAdmin webapp, zimlet webapp, zmconfigd |

### Traffic Flow

```
User Browser → VM2 (Proxy:443) → VM3 (Mailbox)
Incoming Email → VM2 (Postfix:25) → VM3 (LMTP:7025)
Authentication → All Nodes → VM1 (LDAP:389)
Admin Console → Host/VPN → VM3 (HTTPS:7071/8443)
```

---

## 💻 Technical Stack

| Component | Version / Detail |
|---|---|
| **Guest OS** | Rocky Linux 9.7 (RHEL-compatible) |
| **Mail Platform** | Zimbra 10.1.3 (Daffodil) |
| **Installer** | `zcs-10.1.3_GA_4200000.RHEL9_64.20241119113627.tgz` |
| **MTA** | Postfix (bundled with Zimbra) |
| **Directory** | OpenLDAP (zimbra-ldap) |
| **Proxy** | Nginx (zimbra-proxy) |
| **Anti-spam/virus** | Amavis, SpamAssassin, ClamAV |
| **DKIM** | OpenDKIM |
| **Hypervisor** | VMware Workstation |
| **Public Access** | Tailscale Funnel |

---

## 🌐 Network Configuration

### Static IP Assignment (all 3 VMs)

Network interface: **br0** (VMware bridge)  
Network: **192.168.232.0/24** (NAT - VMnet8)

### /etc/hosts Configuration

Add the following to `/etc/hosts` on **all three VMs**:

```
127.0.0.1       localhost
127.0.0.1       mail.entreprise.tn
192.168.232.10  ldap.mail.lab  ldap
192.168.232.11  mta.mail.lab   mta
192.168.232.12  store.mail.lab store
```

---

## 🚀 Deployment Process

### Installation Order (Critical)

⚠️ **Must be followed sequentially** — each node registers with LDAP during installation.

1. **VM1-LDAP** → Install `zimbra-ldap`
2. **VM2-MTA** → Install `zimbra-mta`, `zimbra-proxy`, `zimbra-logger`, `zimbra-snmp`
3. **VM3-Mailbox** → Install `zimbra-store`, `zimbra-spell`, `zimbra-memcached`

### System Preparation (All 3 VMs)

Before running the Zimbra installer:

```bash
# Disable SELinux
sudo setenforce 0
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# Disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable firewalld

# Disable IPv6
echo "net.ipv6.conf.all.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Install dependencies
sudo dnf install -y perl perl-core wget screen libaio nmap-ncat libstdc++ \
    net-tools bind-utils sysstat

# Full system update
sudo dnf update -y
sudo reboot
```

---

## 🌍 Public Access via Tailscale Funnel

To enable remote evaluation and demonstration without router configuration or public IPs, Tailscale Funnel was deployed on VM3.

### Setup (VM3-Mailbox only)

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled

# Connect to Tailscale network
sudo tailscale up --accept-dns=false

# Enable Funnel for HTTPS (port 443)
sudo tailscale funnel 443
```

### Access URLs

| Access Type | URL | Availability |
|---|---|---|
| **Webmail (public)** | `https://store.tail38d366.ts.net/` | Anyone on the internet |
| **Admin Console (private)** | `https://store.tail38d366.ts.net:8443/zimbraAdmin/` | Tailscale VPN only |
| **Webmail (local)** | `https://192.168.232.12/` | Host machine only |
| **Admin Console (local)** | `https://192.168.232.12:7071` | Host machine only |

**Security Note:** The webmail interface is publicly accessible for demonstration purposes. The Admin Console remains accessible only through the private Tailscale VPN network.

---

## 🐛 Known Issue — Jetty XML Duplicate ID Bug

### Problem

After installation on VM3, the `mailboxd` service fails to start with this error in `/opt/zimbra/log/mailbox.log`:

```
FATAL: Duplicate id: pool
FATAL: Duplicate id: Handlers
FATAL: Duplicate id: Contexts
```

### Root Cause

The file `/opt/zimbra/jetty/etc/jetty.xml.in` (template) contains duplicate XML IDs. This template is used by `zmconfigd` to regenerate `jetty.xml` on every service restart.

**Critical:** Editing `jetty.xml` directly has **no lasting effect** — it will be overwritten. You must fix the template file.

### Solution

Extract the clean template from the original Zimbra RPM package:

```bash
cd /opt

# Extract the zimbra-store RPM from the installer tarball
sudo tar -xzf zcs-10.1.3_GA_4200000.RHEL9_64.20241119113627.tgz \
  zcs-10.1.3_GA_4200000.RHEL9_64.20241119113627/packages/zimbra-store*.rpm

# Extract jetty.xml.in from the RPM
rpm2cpio zimbra-store*.rpm | cpio -idmv ./opt/zimbra/jetty/etc/jetty.xml.in

# Replace the broken template with the clean one
sudo cp ./opt/zimbra/jetty/etc/jetty.xml.in /opt/zimbra/jetty/etc/jetty.xml.in

# Restart Zimbra services
su - zimbra -c "zmcontrol restart"
```

After restart, verify all services are running:

```bash
su - zimbra
zmcontrol status
```

All services should show **Running**.

---

## ✅ Service Verification

### Check Service Status

```bash
sudo su - zimbra
zmcontrol status
```

### Expected Output

**VM1-LDAP:**
```
ldap                    Running
stats                   Running
zmconfigd               Running
```

**VM2-MTA:**
```
amavis                  Running
antispam                Running
antivirus               Running
logger                  Running
memcached               Running
mta                     Running
opendkim                Running
proxy                   Running
snmp                    Running
stats                   Running
zmconfigd               Running
```

**VM3-Mailbox:**
```
mailbox                 Running
memcached               Running
service webapp          Running
spell                   Running
stats                   Running
zimbra webapp           Running
zimbraAdmin webapp      Running
zimlet webapp           Running
zmconfigd               Running
```

---

## 🧪 Functional Validation

### Test User Creation

```bash
su - zimbra
zmprov ca user1@mail.lab password123
zmprov ca user2@mail.lab password456
```

### Test Email Flow

1. Log in to webmail as `user1@mail.lab`
2. Send an email to `user2@mail.lab`
3. Log in as `user2@mail.lab` and verify receipt

### Connectivity Tests

```bash
# From any VM, test FQDN resolution
ping ldap.mail.lab
ping mta.mail.lab
ping store.mail.lab

# Test LDAP connectivity (from VM2 or VM3)
nc -zv ldap.mail.lab 389

# Test SMTP (from VM3)
nc -zv mta.mail.lab 25
```

---

## 📋 Quick Reference

| Item | Value |
|---|---|
| **Zimbra Version** | Zimbra 10.1.3 (Daffodil) |
| **Installer** | `zcs-10.1.3_GA_4200000.RHEL9_64.20241119113627.tgz` |
| **OS** | Rocky Linux 9.7 |
| **VM1 (LDAP)** | `192.168.232.10` — `ldap.mail.lab` |
| **VM2 (MTA/Proxy)** | `192.168.232.11` — `mta.mail.lab` |
| **VM3 (Mailbox)** | `192.168.232.12` — `store.mail.lab` |
| **LDAP Port** | 389 |
| **SMTP Ports** | 25 (inbound), 587 (submission) |
| **IMAP Ports** | 143 (plain), 993 (SSL) |
| **Proxy Ports** | 80, 443 (web), 143, 993 (IMAP) |
| **LMTP** | 7025 (MTA → Mailbox) |
| **Admin Console** | 7071 (local), 8443 (Tailscale) |

---

## 📚 Project Outcomes

- ✅ Fully operational distributed enterprise mail platform
- ✅ Validated mail flow: SMTP, IMAP, webmail, administration
- ✅ Resolved critical Jetty XML configuration bug in Zimbra 10.1.3
- ✅ Public access enabled without compromising administrative security
- ✅ Hands-on experience with production-grade mail server architecture

---

## 📄 Documentation

For complete technical details, installation procedures, troubleshooting steps, and architecture analysis, refer to the accompanying **Technical Report** (`Zimbra_Technical_Report.docx`).

---

## 🔧 Troubleshooting

### Services Won't Start

1. Check logs: `tail -f /opt/zimbra/log/mailbox.log`
2. Verify LDAP connectivity from other nodes
3. Ensure `/etc/hosts` is identical on all VMs
4. Check service status: `zmcontrol status`

### Network Issues

1. Verify static IPs: `ip addr show br0`
2. Test ping between all nodes by FQDN
3. Check firewalld is disabled: `systemctl status firewalld`
4. Verify SELinux is disabled: `getenforce`

### Jetty Won't Start

Follow the **Jetty XML Bug Fix** procedure above. Always fix the template (`jetty.xml.in`), not the generated file.

---

**Project by Rawen ZGARNI & Warda GUESMI**  
Software Engineering Students — ENICarthage, Tunisia  
Academic Year 2025/2026
