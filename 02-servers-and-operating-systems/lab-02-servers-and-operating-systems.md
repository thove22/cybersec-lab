# Lab 02 -- Servers and Operating Systems: How Do You Build a Server That Resists Attack?

## Overview

This lab starts from a simple observation: every service on a network is delivered by a server, and every server is only as safe as the operating system and configuration underneath it. We start with the theory (why organizations depend on servers, what threatens them, which tools we choose to build them, and what hardening means), then build a Debian server inside a virtual machine, reach it remotely, install services on it, and finish by reducing its attack surface following an industry standard.

By the end you should be able to:

- Explain how a business depends on IT infrastructure and where servers and operating systems fit in.
- Describe the main types of malware and the phases of a cyberattack, and say where a defender can interrupt them.
- Justify the choice of Debian for a server using clear criteria, and explain what open source software means in practice.
- Build a minimal Linux server in a virtual machine, administer it over SSH, and run web and DNS services on it.
- Apply basic hardening to the server, guided by the CIS Benchmarks.

---

## 1. Why servers and operating systems matter in cybersecurity

Lab 01 followed a web request across the network until it reached a web server. We treated that server as a box that simply answers. This lab opens the box. A server is a computer running an operating system, with services listening on ports, administered by people, and holding data the organization cannot afford to lose. Each one of those elements is a place where something can go wrong.

From an attacker's point of view, servers are the high-value targets. One server often serves hundreds of clients, so compromising it is worth far more than compromising a single laptop. And the operating system is the layer that decides who can read a file, start a process or open a port. In practice, attackers rarely need a sophisticated exploit: a weak password, an outdated package or a service nobody remembers installing is usually enough.

The question behind this lab is: **how do you build a server that does its job and still resists attack?** To answer it we need to understand what the server is for, what threatens it, what we build it with, and how we reduce what can be attacked.

---

## 2. Concepts and technologies

### 2.1 From business to infrastructure

An organization exists to deliver value: sell a product, treat a patient, move money. Those are business processes, and today almost all of them run on information technology. IT automates them, speeds them up and keeps a record of them. The consequence is simple: when IT stops, the business stops.

What keeps it running is the information systems infrastructure: hardware, software, networks and data, plus the people and procedures that operate them. Its goals are availability, performance and security. Infrastructure is built in layers, and each layer depends on the one below it.

| Layer | What it is | Examples | When it breaks, ask... |
|-------|-----------|----------|------------------------|
| Users and procedures | The people and the rules they follow | Staff, administrators, backup routines | Who has access, and did they follow the procedure? |
| Applications and services | Software that delivers something useful | Website, database, mail service | Is the service running and responding? |
| Operating system | Software that manages the machine | Debian, Windows Server | Is the OS healthy, patched and correctly configured? |
| Hardware | The physical machine | CPU, RAM, disks, power | Is the machine powered on and are its components working? |
| Network | The links between machines | Switches, routers, cables | Can the machines reach each other? (Lab 01) |

When a layer fails, everything above it fails too. That is why troubleshooting, and security analysis, is done layer by layer, the same way the TCP/IP model worked in Lab 01.

Three terms get mixed up constantly, and each one sits at a different level of responsibility:

- **Service** -- what the user consumes (an online shop).
- **System** -- the set of components that delivers it (web server, database, payment application).
- **Platform** -- the base on which other systems are built (an operating system or a cloud environment).

An administrator needs to know which level they are working at. So does an attacker: compromising the platform gives access to everything built on it.

### 2.2 Client-server architecture

Client-server is the model behind almost everything on a network. A client sends a request, and a server processes it and returns a response. One server attends many clients at the same time.

1. **Connect** -- the client opens a connection to the server's IP address and port.
2. **Request** -- the client asks for something (a page, a file, a name resolution).
3. **Process** -- the server does the work: reads a file, queries a database, checks a permission.
4. **Respond** -- the server sends the result back and the client uses it.

"Client" and "server" describe roles, not hardware. The same machine can be a client in one conversation and a server in another. What makes a machine a server in practice is that it runs a service that waits for requests and is expected to be available all the time.

Lab 01 was exactly this model: PC0 asked, Server0 answered. Here we build the server side ourselves. One thing worth remembering: because one server serves many clients, a single compromise multiplies. That concentration of risk is the reason servers deserve more protection than ordinary workstations.

### 2.3 The operating system

The operating system (OS) sits between the hardware and the applications. Without it, the hardware is just metal and no application could run on it. Its core jobs are:

- **Processes and CPU** -- deciding which program runs, when, and for how long.
- **Memory** -- giving each program its own space and keeping them apart.
- **Storage** -- organizing data in file systems.
- **Devices and network** -- talking to disks, network cards and other hardware on behalf of applications.
- **Users and permissions** -- deciding who can do what.

The part of the OS that talks directly to the hardware is the **kernel**. Around it sits everything else the user touches: the shell, the system utilities and the services. This distinction matters in section 2.5, because "Linux" and "Debian" are not the same thing.

Desktop and server operating systems do the same basic work but optimize for different things. A desktop OS optimizes for interaction: graphical interface, ease of use, multimedia. A server OS optimizes for staying up for months without supervision, being administered remotely, and running only what it needs. The choice of OS on a server determines its stability, its security and its cost.

The security angle is direct: the OS is where access control is enforced. If an attacker takes control of the OS, particularly an administrator account, every protection built on top of it stops meaning anything.

### 2.4 Open source software

Open source software is software whose source code is publicly available, and which anyone may use, study, modify and redistribute under the terms of its license. It is developed by communities of volunteers and companies, and a large part of the internet's infrastructure runs on it.

| | Proprietary | Open source |
|---|-------------|-------------|
| Source code | Closed | Open and auditable |
| License cost | Usually paid | Usually free |
| Vendor dependence | High | Low |
| Who fixes bugs | The vendor | The community and anyone who contributes |
| Support | From the vendor | From the community or from companies that sell it |

Two misconceptions are worth clearing up. First, open source does not mean "without support": companies sell commercial support for open source products. Second, it does not mean "automatically secure". Open code can be reviewed by many people, but only if someone actually does it. The Heartbleed bug sat unnoticed in OpenSSL, one of the most widely used open source libraries, for about two years. The real advantage is not that open source is immune to flaws, but that anyone can inspect it, and fixes tend to arrive quickly once a flaw is found. Applying those fixes is still the administrator's job.

For organizations with limited budgets, avoiding license costs and vendor lock-in is a strategic advantage, which is a large part of why Linux dominates the server world.

### 2.5 Linux, Debian and choosing a distribution

**Linux** is a kernel, created by Linus Torvalds in 1991. By itself, a kernel is not something you can install and use. A **distribution** (distro) packages the Linux kernel with system utilities, a package manager, a default configuration and a set of software, producing a complete operating system.

**Debian** is one of the oldest distributions, maintained by a community project since 1993. It has a reputation for stability and for a careful release process, which is why it is widely used on servers. Debian publishes three branches:

| Branch | Purpose |
|--------|---------|
| Stable | The released version. Packages are older but thoroughly tested. This is what servers should run. |
| Testing | Packages being prepared for the next stable release. |
| Unstable (sid) | Where new versions arrive first. Expect breakage. |

Debian is also the foundation of many other distributions. Ubuntu is built on Debian, and Lubuntu is an official Ubuntu flavor with a lightweight graphical desktop (LXQt):

```
Linux kernel
 └── Debian
      └── Ubuntu
           └── Lubuntu
```

Because they share the same roots, they also share the same package system and many of the same commands, so what you learn on Debian transfers directly.

With hundreds of distributions to choose from, the decision should follow criteria, not fashion. **DistroWatch** (distrowatch.com) is a site that lists, describes and tracks distributions and their versions. Its popularity ranking is based on page visits, not installations, so treat it as a discovery tool and not as market data. Useful criteria are the intended purpose, the release and support lifecycle, the size and health of the community, and the resources the machine has available.

| Purpose | What matters most | Example in this lab |
|---------|-------------------|---------------------|
| Server | Stability, long support, timely security updates | Debian (SRV) |
| Desktop | Ease of use, hardware support | Ubuntu |
| Lightweight system | Low memory and CPU usage, old hardware | Lubuntu (client) |

### 2.6 Malware

Malware is software designed to harm, spy on or take control of a system without the owner's consent. The more an organization depends on technology, the larger the surface malware has to work with.

| Type | What it does | How it typically spreads |
|------|-------------|--------------------------|
| Virus | Attaches itself to legitimate files and runs when they run | Infected files, removable media |
| Worm | Copies itself across the network without user action | Unpatched network services |
| Trojan | Pretends to be something useful while doing something else | Fake installers, cracked software |
| Ransomware | Encrypts data and demands payment for the key | Phishing e-mail, exposed services |
| Spyware | Quietly collects information (credentials, activity) | Bundled software, malicious downloads |

The damage falls into four areas: data loss, downtime, financial cost and loss of reputation. Common entry points are e-mail attachments, downloads, unpatched vulnerabilities and weak credentials.

One difference is worth keeping in mind. On a workstation, malware usually arrives through a user's click. On a server, nobody is browsing the web or opening attachments, so it mostly arrives through exposed services, outdated software and stolen or guessed passwords. That is why server security depends so heavily on what is installed, what is listening, and how current it is.

### 2.7 Anatomy of a cyberattack

An attack is rarely a single event. It follows phases, and that is good news for defenders: if you know the phases, you can put a defense in each of them.

| Phase | What the attacker does | Defender's question |
|-------|-----------------------|---------------------|
| Reconnaissance | Collects information: IP ranges, open ports, software versions | What does the outside world see of us? |
| Intrusion | Gets a first foothold: stolen password, phishing, exposed service | Which doors are open, and who guards them? |
| Exploitation | Abuses a vulnerability or misconfiguration to gain control | Is everything patched and correctly configured? |
| Persistence and lateral movement | Stays in, creates accounts, moves to other machines | Would we notice a new account or unusual traffic? |
| Impact | Steals, encrypts or destroys data | Can we recover? |

This is a simplified model. Frameworks such as the Lockheed Martin Cyber Kill Chain and MITRE ATT&CK break it down into much finer steps, but the logic is the same: an attack that is stopped early costs far less than one that reaches the last phase.

### 2.8 Cybersecurity as a continuous process

Security is not a product you buy and install. It is a continuous process involving people, procedures and technology. There is no total security, only risk management: reducing the probability of an incident and its impact. The cycle has four activities:

| Activity | Goal | How it shows up in this lab |
|----------|------|-----------------------------|
| Prevent | Make attacks harder | Minimal installation, updates, secure configuration |
| Detect | Notice when something is wrong | Checking which ports are listening, reading logs |
| Respond | Contain and fix an incident | Response plans (discussed, not practiced here) |
| Recover | Return to normal operation | Backups, and snapshots of the virtual machine |

Three principles run through everything that follows:

- **CIA triad** -- confidentiality (only authorized people read the data), integrity (data is not altered without authorization) and availability (the service works when needed). Every attack targets at least one of them.
- **Least privilege** -- every user and process gets only the access it needs, nothing more.
- **Defense in depth** -- several independent layers of protection, so that one failure does not mean total compromise.

### 2.9 Virtualization

Virtualization lets one physical computer run several virtual machines (VMs), each behaving like a separate computer with its own operating system. The physical machine is the **host**, the VMs are **guests**, and the software that creates and manages them is the **hypervisor**. VirtualBox, the one we use, is a hosted (Type 2) hypervisor: it runs as an application on top of a normal OS. Bare-metal (Type 1) hypervisors, such as VMware ESXi or Microsoft Hyper-V, run directly on the hardware and are what data centers use.

For learning, the benefits are practical: guests are isolated from the host, so a mistake does not damage the real computer. If you break a VM, you delete it and create another, or return to a **snapshot** (a saved state of the VM). In companies, virtualization also reduces the number of physical servers and the cost of running them.

The virtual network adapter of a VM can work in different modes, and the choice defines who can see the VM:

| Mode | VM reaches the outside | Other devices reach the VM | Typical use |
|------|-----------------------|---------------------------|-------------|
| NAT | Yes | No (unless ports are forwarded) | Browsing and updates from an isolated VM |
| Bridged | Yes | Yes | Server that other devices must reach |
| Host-only | No | Only the host | Private tests between host and VM |
| Internal | No | Only other VMs on the same internal network | Fully isolated lab networks |

In **bridged** mode the VM behaves as one more device on the physical network: it has its own MAC address and gets its own IP address from the same network as the host, usually through DHCP (Lab 01, section 2.5). That is what we need for a server that other machines have to reach. But the same visibility that makes it useful also exposes it, which is a good reason to harden it afterwards.

### 2.10 Attack surface and minimal installation

The **attack surface** is the sum of all the points where an attacker could try to get in: installed software, running services, open ports, user accounts. Every package you install is code that may contain a vulnerability, and code you have to keep updated. The simplest and most effective way to protect a server is to reduce this surface from the start.

This is why a server is installed **without a graphical environment**. Nobody needs a desktop to serve a web page, and a desktop brings hundreds of extra packages. A server without one is lighter (less RAM, CPU and disk), faster, and has fewer components to attack. The price is that all administration happens through the command line, which is a skill worth having anyway.

The same logic applies during installation. Select only what the server's role requires: in this lab, an SSH server and the standard system utilities. Anything else can be added later, deliberately, when there is a reason.

### 2.11 Users, root and least privilege

Linux separates users sharply. Every account has its own identity, files and permissions. At the top is **root**, the superuser, with unrestricted access to the whole system. Root can read any file, stop any process and delete anything, including the system itself. There is no recycle bin and no confirmation.

That is why the principle of least privilege matters so much here:

- Use a normal account for daily work, where mistakes are contained.
- Elevate to root only for a specific task that requires it, and leave as soon as it is done.
- Never expose root directly to the network. If an attacker gets the root password, the server is lost.

The shell prompt tells you which one you are using: `$` for a normal user, `#` for root. Treat the `#` as a warning sign.

There are two common ways to elevate privileges: `su -` opens a full session as root (and asks for root's password), while `sudo` runs a single command with elevated rights (and asks for your own password). The hyphen in `su -` matters, because it loads root's own environment. On Debian, `sudo` is only configured by default if you leave the root password empty during installation; if you set one, you use `su -`, as we do in this lab.

### 2.12 Remote administration with SSH

A server in a data center, or in a VM without a screen, is administered remotely. **SSH** (Secure Shell) is the standard tool for this. It follows the client-server model: an SSH server (`sshd`) listens on port 22/TCP, and an SSH client connects to it and gets a remote command line.

All the traffic inside an SSH session is encrypted. This is the opposite of **Telnet**, the older tool SSH replaced, which sent everything, passwords included, in plaintext over port 23. Lab 01, section 12, showed what a sniffer sees in plaintext; with Telnet, an administrator's password would be right there in the capture.

Two details are worth knowing:

- **Host key check.** The first time you connect to a server, the client shows its key fingerprint and asks you to confirm. This is how SSH protects you from connecting to an impostor. If the fingerprint changes later without reason, treat it as a red flag.
- **Authentication.** SSH accepts passwords, but also cryptographic key pairs, which are much harder to guess or brute-force.

SSH is also the most frequent target of automated attacks. Any server with port 22 exposed to the internet will see constant login attempts within minutes. Strong passwords, no direct root login, and key-based authentication are the first defenses. By default, a Debian server does not allow root to log in over SSH with a password.

### 2.13 Packages and repositories

On Linux, software is installed through a **package manager**. On Debian and its derivatives this is APT, which works on top of `dpkg`. A package is a bundle of software plus the information needed to install it and its dependencies. APT downloads packages from **repositories**, servers that host collections of packages, defined in `/etc/apt/sources.list` and in files under `/etc/apt/sources.list.d/`.

Official repositories sign their packages cryptographically, and APT verifies the signature before installing. That is why installing software only from official sources is a first line of defense: you know who published the package and that nobody altered it on the way. Adding unknown repositories, or downloading installers from random sites, hands control of your server to someone you have no reason to trust. This is a supply chain risk.

Two operations are often confused:

- **`apt update`** refreshes the list of available packages. It installs nothing.
- **`apt upgrade`** installs the newer versions of the packages already on the system.

Regular updates matter because most successful attacks exploit vulnerabilities that are already known and already fixed. Public vulnerabilities receive a **CVE** identifier (Common Vulnerabilities and Exposures), and the Debian security team publishes fixes for supported releases. An unpatched server is usually a server that missed an update, not one hit by an unknown flaw.

### 2.14 Services, ports and listening sockets

A **service** is a program that runs in the background waiting for requests, usually called a daemon on Linux. For a client to reach it, it **listens** on a port, as we saw in Lab 01. On modern Debian, daemons are managed by `systemd`, which starts them at boot, stops them and reports their status.

Two things determine who can reach a listening service:

- **The port** -- the number that identifies the service.
- **The address it listens on** -- `0.0.0.0` means every network interface (reachable from the network), while `127.0.0.1` means the loopback interface only (reachable from the machine itself).

The consequence for security is direct: **every listening port is a possible entry point**. You cannot protect what you do not know is there, so an administrator regularly lists which ports are open and which process owns each one. If a port appears that nobody recognizes, that is a reason to investigate. The classic tool for this is `netstat`, from the `net-tools` package; its modern replacement is `ss`. As Lab 01 pointed out, a port number does not prove which application is behind it, which is why the process name matters as much as the port.

The server in this lab runs three services:

| Service | Daemon (Debian) | Protocol and port | Role |
|---------|-----------------|-------------------|------|
| SSH | `sshd` | TCP 22 | Remote administration |
| Web (Apache HTTP Server) | `apache2` | TCP 80 | Delivers web pages over HTTP |
| DNS (BIND) | `named` | UDP/TCP 53 | Translates names to IP addresses |

Apache is one of the oldest and most widely used web servers. On Debian it is installed from the `apache2` package, starts automatically, and serves a default page from `/var/www/html`. It is where the client-server model from section 2.2 becomes concrete: a browser asks, Apache answers. Since it communicates over plain HTTP, everything said in Lab 01 about plaintext applies to it too.

DNS was covered in Lab 01, section 2.6, from the client side. Here we run the server side. Its role is the same, but it adds a new concern: DNS answers are trusted blindly by clients, so an attacker who can forge or hijack them can send users to fake sites. A DNS server deserves careful configuration for that reason.

Every service added is a new entry on the attack surface from section 2.10. It should be there because the server's role requires it, and it should be kept up to date.

### 2.15 Hardening and the CIS Benchmarks

**Hardening** is the process of reducing a system's attack surface and tightening its configuration beyond the defaults. Default installations favor convenience and compatibility. Hardening shifts the balance toward security, and it is worth remembering that every restriction has a cost in usability, so the right level depends on what the server does.

Hardening should follow a recognized standard, not improvisation. The **CIS Benchmarks**, published by the Center for Internet Security, are configuration guides developed by a community of experts for a wide range of operating systems and software, including Debian. They list individual checks, each with a rationale and instructions to verify and apply it. They define two profiles: **Level 1**, practical measures with little impact on functionality, and **Level 2**, for environments that need stronger security and can accept some loss of functionality.

| Area | Example measure | Why it matters |
|------|----------------|----------------|
| Software | Remove packages and services nobody needs | Fewer vulnerabilities to exploit |
| Access | No root login over SSH, strong passwords, least privilege | Stops the easiest intrusions |
| Network | Firewall allowing only required ports | Closes doors that are not needed |
| Updates | Regular patching from official repositories | Fixes known vulnerabilities |
| Monitoring | Review of logs and listening ports | Detects what prevention missed |

Back to the cycle in section 2.8: hardening is mostly prevention, but it also depends on detection, because you need to see what is exposed before you can close it. And it is not a one-time task. New software, new vulnerabilities and new requirements keep changing the picture, so hardening is a routine, not an event.
