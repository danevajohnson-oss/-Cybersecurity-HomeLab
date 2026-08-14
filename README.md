# Building My Cybersecurity Home Lab with Proxmox Code Name Project Worthington (B082)

## Overview

As part of developing my practical cybersecurity skills, I decided to build a dedicated **cybersecurity home lab** where I can safely deploy virtual machines, configure networks, test security tools, and experiment without affecting my main network or computers.

For the lab, I am using **Proxmox Virtual Environment (Proxmox VE)** as the hypervisor installed on a dedicated physical computer.

![image alt](https://github.com/danevajohnson-oss/-Cybersecurity-HomeLab/blob/aa92ad2b8dd83abca6933ee402a47646851746b6/Kali%20Installed.png)

The basic design of the lab is:

```text
                    Home Network
                  192.168.0.0/24
                         |
                         |
                 +-------+-------+
                 |               |
                 | Proxmox Host  |
                 | 192.168.0.17  |
                 |               |
                 +-------+-------+
                         |
                   vmbr1 - NAT
                   10.0.0.1/24
                         |
              Cybersecurity Lab Network
                    10.0.0.0/24
                         |
               +---------+---------+
               |                   |
          Kali Linux          Future VMs
          10.0.0.24          10.0.0.x
```

This gives me two separate networks:

* **192.168.0.0/24** — My normal home network and Proxmox management network.
* **10.0.0.0/24** — My internal cybersecurity lab network.

My Proxmox server is accessible from my normal network at:

```text
https://192.168.0.17:8006
```

The virtual machines used for cybersecurity testing will primarily reside on the **10.0.0.0/24** network.

---

# 1. Download the Required Software

Before building the lab, I downloaded the software required for the initial installation.

## Proxmox VE

Download the latest Proxmox VE ISO from the official Proxmox website.

The ISO is then written to a USB drive using an imaging utility such as:

* Rufus
* balenaEtcher
* Ventoy

The computer that will become the Proxmox server is booted from this USB drive.

---

## Kali Linux

Kali Linux will be my primary cybersecurity workstation.

Instead of downloading the Kali ISO to another computer and manually uploading it to Proxmox, I can download the ISO directly from within the Proxmox interface.

In Proxmox:

```text
Datacenter
└── Proxmox Node
    └── local
        └── ISO Images
            └── Download from URL
```

I can copy the download URL for the appropriate Kali Linux installer ISO from the official Kali Linux website and allow Proxmox to download it directly.

---

# 2. Install Proxmox VE

Boot the physical computer from the Proxmox installation USB.

Follow the installation wizard to:

1. Select the installation drive.
2. Configure the country, timezone, and keyboard layout.
3. Create the root password.
4. Configure the management network.
5. Complete the installation.

For my environment, the Proxmox management IP is:

```text
192.168.0.17
```

After installation, Proxmox can be managed from another computer on my network by opening:

```text
https://192.168.0.17:8006
```

Log in using the `root` account created during installation.

---

# 3. Create the Isolated Cybersecurity Lab Network

I don't want all of my cybersecurity virtual machines sitting directly on my normal home network.

Instead, I created a separate virtual network:

```text
10.0.0.0/24
```

The Proxmox host will act as the gateway for this network using:

```text
10.0.0.1
```

In Proxmox, navigate to:

```text
Node
└── System
    └── Network
```

Create a new **Linux Bridge**.

For example:

```text
Name: vmbr1
IPv4/CIDR: 10.0.0.1/24
Bridge Ports: <leave blank>
Autostart: Yes
```

The bridge does not need to be connected directly to a physical interface because it is being used as an internal virtual network.

The resulting design is:

```text
vmbr0
192.168.0.17
     |
Physical/Home Network
192.168.0.0/24


vmbr1
10.0.0.1
     |
Cybersecurity Lab
10.0.0.0/24
```

---

# 4. Configure NAT for the Lab Network

The VMs need to be isolated from my home network while still being able to access the Internet for tasks such as:

```bash
sudo apt update
sudo apt upgrade
```

To accomplish this, the Proxmox host can perform **Network Address Translation (NAT)** between the lab network and the external network.

IP forwarding must first be enabled on the Proxmox host.

Edit:

```bash
nano /etc/sysctl.conf
```

Ensure the following setting exists:

```text
net.ipv4.ip_forward=1
```

Apply the change:

```bash
sysctl -p
```

NAT can then be configured so traffic originating from:

```text
10.0.0.0/24
```

is translated when leaving through the Proxmox host's external interface.

A typical configuration conceptually looks like:

```bash
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o vmbr0 -j MASQUERADE
```

> **Note:** The exact firewall/NAT configuration can vary depending on the Proxmox version and whether the Proxmox firewall, `iptables`, or `nftables` is being used. The configuration should be verified before applying it permanently.

The goal is:

```text
Kali VM
10.0.0.24
    |
    v
vmbr1
10.0.0.1
    |
    | NAT
    v
Proxmox
192.168.0.17
    |
    v
Home Router
    |
    v
Internet
```

This allows the lab machines to reach the Internet without requiring a `192.168.0.x` address.

---

# 5. Create the Kali Linux Virtual Machine

With the network prepared, the next step is creating the Kali Linux VM.

From the Proxmox web interface, select:

```text
Create VM
```

Configure the VM with appropriate resources.

An example starting configuration is:

| Resource         | Configuration |
| ---------------- | ------------- |
| Operating System | Kali Linux    |
| CPU              | 4 Cores       |
| RAM              | 4 GB.         |
| Storage          | 64 GB.        |
| Network Bridge   | vmbr1         |
| Network Model    | VirtIO        |

Attach the Kali Linux ISO downloaded earlier and start the VM.

Proceed through the normal Kali Linux installation process.

---

# 6. Configure the Kali Linux Network

The Kali VM will use the following static network configuration:

```text
IP Address: 10.0.0.24
Subnet Mask: 255.255.255.0
Gateway: 10.0.0.1
```

A DNS server must also be configured.

For example:

```text
DNS: 1.1.1.1
```

The final network configuration should resemble:

```text
Kali Linux
------------------------
IP:       10.0.0.24
Network:  10.0.0.0/24
Gateway:  10.0.0.1
DNS:      1.1.1.1
```

Once configured, I can test connectivity.

First, verify that Kali can reach the Proxmox virtual gateway:

```bash
ping 10.0.0.1
```

Then test Internet connectivity:

```bash
ping 1.1.1.1
```

Finally, verify DNS resolution:

```bash
ping kali.org
```

If all three tests work, the VM has functioning network connectivity.

---

# 7. Update Kali Linux

Once Kali has Internet connectivity, I update the system before using it.

```bash
sudo apt update
```

Then:

```bash
sudo apt full-upgrade -y
```

After the updates are complete, reboot the VM if required.

```bash
sudo reboot
```

---

# 8. Create a Clean Snapshot

Before installing additional cybersecurity tools or making major configuration changes, I create a snapshot of the VM.

In Proxmox:

```text
Kali VM
└── Snapshots
    └── Take Snapshot
```

I can give the snapshot a descriptive name such as:

```text
CleanInstallKali
```

Description:

```text
This is the starting point
```

![image alt](https://github.com/danevajohnson-oss/-Cybersecurity-HomeLab/blob/8dcbfb0c9e9f7320bfd385c229a4c38c236aaace/Snapshot%20of%20Kali.png)

This gives me a known-good state that I can return to if I break the VM while experimenting.

This is especially useful in a cybersecurity lab because I expect to intentionally modify systems, test tools, change configurations, and occasionally break things.

---

# Current Lab Architecture

At this stage, the home lab looks like this:

```text
                        INTERNET
                            |
                            |
                       Home Router
                            |
                     192.168.0.0/24
                            |
                            |
                  +-------------------+
                  |   Proxmox Server  |
                  |                   |
                  |  vmbr0            |
                  |  192.168.0.17     |
                  |                   |
                  |  NAT / Routing    |
                  |        |          |
                  |  vmbr1 |          |
                  |  10.0.0.1/24      |
                  +--------+----------+
                           |
                           |
                 Cybersecurity Lab
                    10.0.0.0/24
                           |
                    +------+------+
                    |             |
                 Kali Linux    Future VMs
                 10.0.0.24     10.0.0.x
```

---

# Why I Designed the Lab This Way

The main reason for using a separate `10.0.0.0/24` network is **isolation**.

As the lab grows, I intend to deploy machines that may intentionally contain vulnerabilities or be used to simulate attacks.

Examples could eventually include:

* Kali Linux attack machines
* Windows 10/11 clients
* Windows Server
* Active Directory Domain Controllers
* Linux servers
* Vulnerable web applications
* Metasploitable
* SIEM platforms
* Security monitoring systems
* Firewalls
* IDS/IPS systems

Keeping these machines on a dedicated virtual network gives me greater control over how they communicate.

It also provides an environment where I can experiment with:

* Network scanning
* Vulnerability assessment
* Penetration testing
* Active Directory security
* Network segmentation
* Firewall configuration
* SIEM monitoring
* Incident response
* Digital forensics
* Malware analysis

without treating my normal home network as the lab.

---

# Next Steps

This is only the foundation of the home lab.

My next steps will be to expand the environment with additional virtual machines and security infrastructure.

Some planned projects include:

* [ ] Deploy a Windows 11 workstation
* [ ] Deploy Windows Server
* [ ] Build an Active Directory domain
* [ ] Create additional isolated network segments
* [ ] Deploy intentionally vulnerable machines
* [ ] Configure a firewall between lab networks
* [ ] Deploy a SIEM
* [ ] Configure centralized logging
* [ ] Generate simulated attacks from Kali Linux
* [ ] Detect those attacks using the SIEM
* [ ] Practice incident response and digital forensics

The goal is to gradually turn the Proxmox server into a small virtual enterprise environment that I can use to practice both **offensive and defensive cybersecurity techniques**.

---

## Disclaimer

This lab is intended for **educational and authorized cybersecurity testing only**.

Any offensive security techniques or tools used in this environment should only be used against systems that I own or have explicit permission to test.

---

## Repository

This repository will document the continued development of my cybersecurity home lab, including network diagrams, VM configurations, security tools, experiments, lessons learned, and troubleshooting notes.

As the lab evolves, I will continue adding documentation for each major component.
