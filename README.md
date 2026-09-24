# Linux Server Setup

Learn to set up and secure a Linux server from scratch.

**Project Reference:** [roadmap.sh/projects/linux-server-setup](https://roadmap.sh/projects/linux-server-setup)

## Table of Contents

* [About](#about)
* [Objectives](#objectives)
* [Architecture](#architecture)
* [Requirements](#requirements)
* [Initial Connection](#initial-connection)
* [Configuration steps](#configuration-steps)
* [1. User Setup](#1-user-setup)
* [2. SSH Configuration](#2-ssh-configuration)
* [3. Firewall Configuration](#3-firewall-configuration)
* [4. System Updates](#4-system-updates)
* [5. Basic Hardening with Fail2Ban](#5-basic-hardening-with-fail2ban)
* [6. Server Configuration](#6-server-configuration)
* [7. Service Management with systemctl](#7-service-management-with-systemctl)
* [8. Log Inspection](#8-log-inspection)
* [9. Security Verification](#9-security-verification)
* [How It Works](#how-it-works)
* [Useful Commands](#useful-commands)
* [Lessons Learned](#lessons-learned)
* [Roadmap](#roadmap)
* [Author](#author)
* [License](#license)

### About

The goal of this project is to manually configure a fresh Linux server before deploying applications on it.

The server is an Ubuntu 24.04 LTS virtual machine provisioned on Oracle Cloud Infrastructure.

The configuration covers fundamental Linux system administration and security concepts:

* Linux users and groups
* `sudo` privileges
* SSH authentication
* SSH hardening
* Network ports
* Firewall configuration with UFW
* Package management with APT
* Automatic security updates
* Fail2Ban
* Timezone and hostname configuration
* `systemctl` and systemd services
* Linux logs and `journalctl`
* Basic server security verification

This project is intentionally performed manually before using automation tools such as Ansible.

### Objectives

The server should satisfy the following requirements:

1. Create a non-root administrative user.
2. Give this user `sudo` privileges.
3. Configure SSH key-based authentication.
4. Disable SSH password authentication.
5. Configure UFW with a default-deny incoming policy.
6. Allow SSH through TCP port 22.
7. Update system packages.
8. Configure automatic security updates.
9. Install and configure Fail2Ban for SSH protection.
10. Configure the server timezone.
11. Configure a meaningful hostname.
12. Demonstrate basic `systemctl` operations.
13. Inspect system and authentication logs.
14. Perform a final security verification.

### Architecture

The resulting server can be represented conceptually as:

```text
                                         INTERNET
                                            │
                                            │
                                     ┌──────▼──────┐
                                     │     UFW     │
                                     │  Firewall   │
                                     └──────┬──────┘
                                            │
                                    TCP port 22 only
                                            │
                                     ┌──────▼──────┐
                                     │     SSH     │
                                     └──────┬──────┘
                                            │
                                    SSH key authentication
                                            │
                                     ┌──────▼──────┐
                                     │    Ubuntu   │
                                     │    Server   │
                                     └──────┬──────┘
                                            │
                               ┌────────────┼────────────┐
                               │            │            │
                             UFW        Fail2Ban       systemd
                               │            │            │
                          Network       SSH abuse     Services
                          filtering     detection     management
```

UFW and Fail2Ban have different roles:

* **UFW** controls which network traffic is allowed to reach the server.
* **Fail2Ban** monitors authentication-related logs and can temporarily ban IP addresses that exhibit repeated failed authentication attempts.

### Requirements

#### Local machine

* SSH client
* An SSH private key associated with the public key installed on the VM
* Network access to the server

#### Server

* Ubuntu 24.04 LTS
* A user with administrative access
* Internet access for package installation and updates

### Initial Connection

The VM was provisioned with an SSH public key.

The private key remains on the local machine and is used to authenticate to the server.

Example:

```bash
ssh -i ~/.ssh/roadmapsh_key ubuntu@<SERVER_IP>
```

Where:

* `-i` specifies the private key to use.
* `~/.ssh/roadmapsh_key` is the private key.
* `ubuntu` is the initial Ubuntu user.
* `<SERVER_IP>` is the public IP address of the VM.

The private key must never be uploaded to the server or committed to Git.

### Configuration steps

#### 1. User Setup

##### 1.1 Check the current user

Before changing anything, identify the current account:

```bash
whoami
```

Example:

```text
ubuntu
```

`whoami` answers:

> Which user am I currently logged in as?

Then inspect the user's identity and groups:

```bash
id
```

Example:

```text
uid=1001(ubuntu) gid=1001(ubuntu) groups=1001(ubuntu),4(adm),24(cdrom),27(sudo),30(dip),101(lxd)
```

Important information:

* `uid` → user ID
* `gid` → primary group ID
* `groups` → groups to which the user belongs
* `sudo` in the groups list → the user has administrative privileges through `sudo`

##### 1.2 Create a non-root administrative user

Create the new user:

```bash
sudo adduser jescaude
```

`adduser` creates:

* the user account
* the user's home directory
* the primary group
* the necessary account configuration

The command interactively asks for a password and optional user information.

##### 1.3 Give the user sudo privileges

Add the user to the `sudo` group:

```bash
sudo usermod -aG sudo jescaude
```

The options mean:

* `-a` → append the group instead of replacing the user's existing supplementary groups.
* `-G sudo` → add the user to the `sudo` group.

The command can be understood as:

```text
usermod
   │
   ├── -a  → add without removing existing groups
   └── -G sudo → add to sudo group
```

Verify:

```bash
id jescaude
```

The `sudo` group should appear in the output.

##### 1.4 Test the new account

Open a new SSH session using the new user:

```bash
ssh -i ~/.ssh/roadmapsh_key jescaude@<SERVER_IP>
```

Then verify:

```bash
whoami
```

Expected:

```text
jescaude
```

Test administrative access:

```bash
sudo whoami
```

Expected:

```text
root
```

This does **not** mean that `jescaude` is root.

It means that the user is allowed to execute a specific command with root privileges through `sudo`.

#### 2. SSH Configuration

##### 2.1 SSH key authentication

The VM was provisioned with the public SSH key corresponding to the private key:

```text
~/.ssh/roadmapsh_key
```

The public key was provided to OCI during VM creation.

The private key remains on the local machine.

The authentication flow is:

```text
Local machine
    │
    │ private key
    ▼
SSH client
    │
    │ SSH connection
    ▼
Server
    │
    │ checks authorized public key
    ▼
Authentication
```

The private key itself is never sent to the server as the authentication credential.

##### 2.2 Verify SSH authentication

Connect using:

```bash
ssh -i ~/.ssh/roadmapsh_key jescaude@<SERVER_IP>
```

If the connection succeeds without asking for the server user's password, key-based authentication is working.

##### 2.3 Disable password authentication

Check the effective SSH configuration:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication'
```

Expected:

```text
pubkeyauthentication yes
passwordauthentication no
```

This means:

* public-key authentication is enabled;
* password authentication is disabled.

This prevents SSH logins using a normal account password.

#### 3. Firewall Configuration

##### 3.1 Why a firewall is needed

A server connected to the Internet can potentially receive network connection attempts on many ports.

A firewall controls which incoming connections are allowed.

For this project, the desired policy is:

```text
Incoming connections
        │
        ├── SSH :22/TCP  → ALLOW
        │
        └── Other ports  → DENY
```

UFW (Uncomplicated Firewall) provides a simpler interface for configuring the Linux firewall.

##### 3.2 Install UFW

If UFW is not installed:

```bash
sudo apt install ufw
```

Check its status:

```bash
sudo ufw status verbose
```

Initially:

```text
Status: inactive
```

##### 3.3 Allow SSH before enabling the firewall

This step is important.

SSH currently uses port 22. If the firewall is enabled before SSH is allowed, the existing SSH connection could be interrupted.

Allow SSH:

```bash
sudo ufw allow 22/tcp
```

Expected:

```text
Rules updated
Rules updated (v6)
```

This creates an incoming rule for:

* IPv4
* IPv6

Verify:

```bash
sudo ufw status numbered
```

##### 3.4 Enable UFW

Once SSH has been explicitly allowed:

```bash
sudo ufw enable
```

UFW warns that enabling the firewall may disrupt SSH connections.

Confirm:

```text
y
```

Expected:

```text
Firewall is active and enabled on system startup
```

This means:

* UFW is currently active.
* UFW will automatically start after a reboot.

##### 3.5 Verify the firewall

```bash
sudo ufw status verbose
```

Example result:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

##### Reading the configuration

`Status: active`

→ UFW is currently applying its rules.

`Logging: on (low)`

→ UFW records selected firewall events in the system logs.

`Default: deny (incoming)`

→ incoming connections are denied unless a rule explicitly allows them.

`allow (outgoing)`

→ outgoing connections from the server are allowed by default.

`22/tcp ALLOW IN`

→ incoming TCP connections to port 22 are allowed.

`Anywhere`

→ the rule is not restricted to a specific source IP.

#### 4. System Updates

##### 4.1 Refresh the package index

Run:

```bash
sudo apt update
```

This downloads the latest package information from the configured Ubuntu repositories.

Important distinction:

```text
apt update
    ↓
updates package information

apt upgrade
    ↓
installs available package updates
```

`apt update` does not itself upgrade installed packages.

##### 4.2 Upgrade installed packages

Run:

```bash
sudo apt upgrade
```

This installs available updates for packages already installed on the system.

These commands were performed on the server as part of the project.

##### 4.3 Automatic security updates

The project also requires automatic security updates using `unattended-upgrades`.

Check whether it is installed:

```bash
apt policy unattended-upgrades
```

If it is not installed:

```bash
sudo apt install unattended-upgrades
```

`unattended-upgrades` provides automated installation of configured package updates, particularly security updates.

The conceptual difference is:

```text
Manual:
sudo apt update
sudo apt upgrade

Automatic:
unattended-upgrades
        ↓
automatically installs eligible configured updates
```

#### 5. Basic Hardening with Fail2Ban

##### 5.1 Install Fail2Ban

Check whether it is installed:

```bash
apt policy fail2ban
```

Install it if necessary:

```bash
sudo apt install fail2ban
```

##### 5.2 Check the service

```bash
sudo systemctl status fail2ban
```

The expected state is:

```text
Active: active (running)
```

The service was also configured to start automatically:

```text
enabled
```

##### 5.3 Check active jails

```bash
sudo fail2ban-client status
```

The server configuration used in this project reported:

```text
Status
|- Number of jail: 1
`- Jail list: sshd
```

The `sshd` jail is responsible for monitoring SSH authentication attempts.

##### 5.4 Inspect the SSH jail

```bash
sudo fail2ban-client status sshd
```

The project server initially reported:

```text
Currently failed: 0
Total failed:     0

Currently banned: 0
Total banned:     0
Banned IP list:
```

This means that no IP address had been banned at the time of verification.

##### 5.5 Check effective Fail2Ban parameters

The effective configuration was checked using:

```bash
sudo fail2ban-client get sshd bantime
sudo fail2ban-client get sshd findtime
sudo fail2ban-client get sshd maxretry
```

The server returned:

```text
bantime  = 600
findtime = 600
maxretry = 5
```

The values mean:

* `bantime = 600` → an offending IP can be banned for 600 seconds (10 minutes).
* `findtime = 600` → Fail2Ban evaluates failures within a 600-second (10-minute) window.
* `maxretry = 5` → five failures within the relevant window can trigger the ban.

Conceptually:

```text
5 failed SSH attempts
within 10 minutes
        ↓
Fail2Ban
        ↓
IP can be banned
        ↓
10-minute ban
```

Fail2Ban complements UFW:

```text
UFW
└── controls which network ports can be reached

Fail2Ban
└── detects repeated authentication failures
    and can temporarily ban offending IPs
```

#### 6. Server Configuration

##### 6.1 Check the current timezone

Use:

```bash
timedatectl
```

The initial configuration was:

```text
Time zone: Etc/UTC (UTC, +0000)
```

For a server operated from Benin, the local timezone can be configured as:

```text
Africa/Porto-Novo
```

Verify that the timezone exists:

```bash
timedatectl list-timezones | grep Africa/Porto-Novo
```

Set it with:

```bash
sudo timedatectl set-timezone Africa/Porto-Novo
```

Verify:

```bash
timedatectl
```

Expected timezone:

```text
Time zone: Africa/Porto-Novo (WAT, +0100)
```

##### 6.2 Check the hostname

The hostname identifies the machine at the operating-system level.

Check it with:

```bash
hostname
```

or:

```bash
hostnamectl
```

The initial hostname was:

```text
roadmapsh-vnic2
```

It appeared in the shell prompt:

```text
jescaude@roadmapsh-vnic2:~$
```

##### 6.3 Set a meaningful hostname

For example:

```bash
sudo hostnamectl set-hostname roadmapsh-server
```

Verify:

```bash
hostnamectl
```

The new hostname should appear as:

```text
Static hostname: roadmapsh-server
```

The hostname is different from the OCI VM `display_name`.

```text
OCI display_name
    ↓
name shown for the VM in OCI

Linux hostname
    ↓
name assigned to the operating system
```

#### 7. Service Management with systemctl

Ubuntu uses systemd to manage many system services.

`systemctl` is the main command used to interact with these services.

##### Check service status

```bash
sudo systemctl status fail2ban
```

##### Start a service

```bash
sudo systemctl start <service>
```

Starts the service immediately.

##### Stop a service

```bash
sudo systemctl stop <service>
```

Stops the service immediately.

##### Restart a service

```bash
sudo systemctl restart <service>
```

Stops and starts the service again.

##### Enable a service at boot

```bash
sudo systemctl enable <service>
```

This configures the service to start automatically when the system boots.

##### Disable automatic startup

```bash
sudo systemctl disable <service>
```

This prevents automatic startup at boot.

##### Start now and enable at boot

```bash
sudo systemctl enable --now <service>
```

This combines:

```text
enable → start automatically after reboot
start  → run now
```

#### 8. Log Inspection

Linux services generate logs that help administrators understand what is happening on the server.

Two important sources are:

* systemd journal
* `/var/log/`

##### 8.1 Inspect the system journal

```bash
sudo journalctl
```

This displays the systemd journal.

Because a server can contain a large number of entries, filtering is usually more practical.

##### 8.2 Inspect Fail2Ban logs

```bash
sudo journalctl -u fail2ban
```

The `-u` option filters the journal by systemd unit.

Therefore:

```text
-u fail2ban
   ↓
show logs related to fail2ban.service
```

##### 8.3 Inspect SSH logs

Depending on the system configuration:

```bash
sudo journalctl -u ssh
```

or:

```bash
sudo journalctl -u sshd
```

The Fail2Ban SSH jail on this server uses the systemd journal and matches:

```text
_SYSTEMD_UNIT=sshd.service + _COMM=sshd
```

This means Fail2Ban can analyze SSH-related events directly from the journal.

##### 8.4 Inspect `/var/log`

List the available log files:

```bash
ls /var/log/
```

A commonly useful authentication log is:

```text
/var/log/auth.log
```

View its latest entries:

```bash
sudo tail /var/log/auth.log
```

View the last 20 lines:

```bash
sudo tail -n 20 /var/log/auth.log
```

`tail` displays the end of a file, which is useful when looking for recent events.

#### 9. Security Verification

The final step is to verify that the server configuration matches the project requirements.

##### User

Check the current user:

```bash
whoami
```

Check identity and groups:

```bash
id
```

Verify sudo access:

```bash
sudo whoami
```

Expected:

```text
root
```

##### SSH

Check effective SSH configuration:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication'
```

Expected:

```text
pubkeyauthentication yes
passwordauthentication no
```

Test SSH from the local machine:

```bash
ssh -i ~/.ssh/roadmapsh_key jescaude@<SERVER_IP>
```

##### Firewall

```bash
sudo ufw status verbose
```

Expected:

```text
Status: active
```

The firewall should have:

```text
Default: deny (incoming)
```

and:

```text
22/tcp ALLOW IN
```

##### Fail2Ban

Check the service:

```bash
sudo systemctl status fail2ban
```

Check active jails:

```bash
sudo fail2ban-client status
```

Check SSH protection:

```bash
sudo fail2ban-client status sshd
```

Verify effective parameters:

```bash
sudo fail2ban-client get sshd bantime
sudo fail2ban-client get sshd findtime
sudo fail2ban-client get sshd maxretry
```

##### Timezone

```bash
timedatectl
```

Verify that the configured timezone is appropriate for the server's intended operation.

##### Hostname

```bash
hostnamectl
```

Verify that the hostname is meaningful and correctly configured.

##### Services

Use:

```bash
systemctl status <service>
```

to verify important services are running.

##### Logs

Verify that logs can be accessed:

```bash
sudo journalctl -u fail2ban
```

and:

```bash
ls /var/log/
```

### How It Works

The final security model is based on several complementary layers.

```text
                                             INTERNET
                                                │
                                                ▼
                                        ┌──────────────┐
                                        │     UFW      │
                                        │  Firewall    │
                                        └──────┬───────┘
                                               │
                                         TCP :22 only
                                               │
                                               ▼
                                        ┌──────────────┐
                                        │     SSH      │
                                        │  Public Key  │
                                        │Authentication│
                                        └──────┬───────┘
                                               │
                                     failed authentication
                                               │
                                               ▼
                                        ┌──────────────┐
                                        │  Fail2Ban    │
                                        └──────┬───────┘
                                               │
                                         repeated abuse
                                               │
                                               ▼
                                        temporary IP ban
```

The security mechanisms have different responsibilities:

#### SSH

Provides secure remote administration.

#### SSH public-key authentication

Authenticates administrators without relying on SSH passwords.

#### UFW

Controls incoming and outgoing network traffic according to firewall rules.

#### Fail2Ban

Monitors authentication events and reacts to repeated failed attempts.

#### systemd

Manages system services and their lifecycle.

#### journald

Collects system and service logs that can be inspected with `journalctl`.

#### APT

Manages Ubuntu packages and their updates.

#### unattended-upgrades

Automates installation of configured eligible updates, particularly security updates.

### Useful Commands

#### Identity

```bash
whoami
id
```

#### Users

```bash
adduser <username>
usermod -aG sudo <username>
```

#### SSH

```bash
ssh -i <private-key> <user>@<server-ip>
```

#### Firewall

```bash
sudo ufw status verbose
sudo ufw allow 22/tcp
sudo ufw enable
```

#### Updates

```bash
sudo apt update
sudo apt upgrade
apt policy unattended-upgrades
```

#### Fail2Ban

```bash
sudo systemctl status fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

#### Time and hostname

```bash
timedatectl
hostname
hostnamectl
```

#### Services

```bash
sudo systemctl status <service>
sudo systemctl start <service>
sudo systemctl stop <service>
sudo systemctl restart <service>
sudo systemctl enable <service>
sudo systemctl disable <service>
```

#### Logs

```bash
sudo journalctl
sudo journalctl -u <service>
sudo tail /var/log/<file>
```

### Lessons Learned

This project introduced the following fundamental server administration concepts:

* A Linux server should not be administered routinely as `root`.
* `sudo` provides controlled administrative privileges.
* SSH public-key authentication is preferable to password-based SSH authentication for this setup.
* A firewall controls network access at the port/protocol level.
* `deny incoming` means that incoming traffic is rejected unless explicitly allowed.
* Port 22/TCP is commonly used for SSH.
* Firewall configuration must be performed carefully when administering a remote server.
* `systemctl` manages systemd services.
* `journalctl` provides access to systemd logs.
* Fail2Ban and UFW provide different layers of protection.
* Package updates and automatic security updates serve different operational purposes.
* Timezone configuration affects timestamps used by logs and applications.
* A hostname identifies the server at the operating-system level.

### Roadmap

Possible extensions after this project:

* Configure Nginx.
* Open ports 80 and 443 only when required.
* Configure HTTPS/TLS.
* Deploy a Node.js service.
* Use Ansible to reproduce the manual server configuration automatically.
* Replace manual firewall and Fail2Ban configuration with Ansible tasks.
* Add monitoring and system metrics.
* Introduce centralized log management.
* Containerize the application with Docker.
* Deploy the application using a CI/CD pipeline.

The manual configuration performed in this project serves as a baseline for the later Ansible automation work.

### Author

**Created by**: Jessica MOUSSOUGAN

**Email**: [jessicamoussougan@gmail.com](mailto:jessicamoussougan@gmail.com)

**GitHub**: [@JescAude18](https://github.com/JescAude18)

### License

No license yet.

This project is currently for personal training and learning.
