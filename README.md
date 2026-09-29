*This project has been created as part of the 42 curriculum by ciparren.*

# Born2beRoot

## Description

Born2beRoot is a system administration project. The goal is to set up a minimal, secure Linux server inside a virtual machine, following strict rules: encrypted partitions with LVM, a strong password policy, a hardened `sudo` configuration, an SSH service on a non-standard port, a firewall, a mandatory access control system, and a monitoring script that broadcasts system information every 10 minutes.

No graphical environment is installed: the server is managed exclusively from the command line, locally or through SSH.

The repository only contains this `README.md` and `signature.txt`, which holds the SHA1 signature of the virtual machine's disk (`.vdi`). The virtual machine itself is not included.

## Instructions

### Requirements

- VirtualBox
- The virtual disk `ciparren.vdi` (stored locally, not in this repository)

### Verifying the signature

From the folder where the VM is stored (`~/sgoinfre/ciparren/`):

```bash
sha1sum ciparren.vdi
shasum ciparren.vdi           # macOS
certUtil -hashfile ciparren.vdi sha1   # Windows
```

The output must be identical to the content of `signature.txt`:

```bash
diff <(sha1sum ciparren.vdi | cut -d' ' -f1) signature.txt
```

> Booting the VM modifies the disk and changes its signature. A backup copy of the original `.vdi` is restored after each evaluation so the signature stays valid.

### Starting the server

1. Start the VM in VirtualBox (no snapshots must exist).
2. Enter the disk encryption passphrase.
3. Log in as `ciparren` (never as root).

### Connecting through SSH

With a NAT port forwarding rule (host 4242 → guest 4242) configured in VirtualBox:

```bash
ssh ciparren@localhost -p 4242
```

Root login through SSH is disabled.

## Project description

### Choice of operating system: Debian

I chose **Debian** (latest stable release) over Rocky Linux.

- **Advantages:** very stable, huge community and documentation, simple package management with `apt`, recommended for people new to system administration, and AppArmor is easier to configure than SELinux.
- **Disadvantages:** packages in the stable branch are sometimes older, and it is less common than the RHEL family in some enterprise environments.

### Main design choices

#### Partitioning

The disk uses a separate unencrypted `/boot` partition and an encrypted partition (LUKS) containing an LVM volume group with several logical volumes:

| Logical volume | Mount point | Purpose |
|---|---|---|
| `root` | `/` | Operating system |
| `swap_1` | `[SWAP]` | Swap memory |
| `home` | `/home` | User data |

Separating `/home` from `/` prevents user data from filling the system partition. Encrypting the LVM partition protects the data if the disk is stolen or copied. LVM allows logical volumes to be resized or added later without repartitioning the physical disk. Sizes were chosen to keep the system functional while avoiding unnecessary disk usage.

#### Security policies

**Password policy**

- `/etc/login.defs`: passwords expire every 30 days (`PASS_MAX_DAYS 30`), the minimum delay between changes is 2 days (`PASS_MIN_DAYS 2`), and users are warned 7 days before expiration (`PASS_WARN_AGE 7`). These values only apply to new users, so `chage` was used to apply them to the existing users (`root` and `ciparren`).
- `/etc/pam.d/common-password` (with `libpam-pwquality`): minimum 10 characters, at least one uppercase letter, one lowercase letter and one digit, no more than 3 identical consecutive characters, the username cannot appear in the password, at least 7 characters must differ from the previous password (not applied to root), and the rules are enforced for root as well.
- All passwords were changed after the policy was configured.

**sudo policy** (configured with `visudo` in `/etc/sudoers.d/`)

- Maximum of 3 authentication attempts.
- Custom error message on a wrong password.
- Every command is logged, including inputs and outputs, in `/var/log/sudo/`.
- `requiretty` is enabled: sudo can only be used from a real terminal.
- `secure_path` restricts the directories sudo can execute from.

**AppArmor** is enabled at boot. It is a Mandatory Access Control system that confines programs to a set of allowed resources through per-program profiles.

**UFW** is enabled at boot and only port 4242 is open.

**SSH** listens only on port 4242 and root login is disabled (`PermitRootLogin no`).

#### User management

- `root` and `ciparren` are present.
- `ciparren` belongs to the `sudo` and `user42` groups.
- The hostname is `ciparren42`.

#### Services installed

- `openssh-server`: remote access on port 4242.
- `ufw`: firewall.
- `sudo`: controlled privilege escalation.
- `libpam-pwquality`: password strength rules.
- `apparmor`: mandatory access control.
- `cron`: runs `monitoring.sh` every 10 minutes.

#### monitoring.sh

A bash script that uses `wall` to broadcast the following on all terminals: architecture and kernel version, physical and virtual CPUs, RAM usage, disk usage, CPU load, last boot, LVM status, active TCP connections, logged users, IPv4 and MAC address, and number of commands run with sudo. It is scheduled with `cron` from root's crontab.

### Comparisons

#### Debian vs Rocky Linux

| | Debian | Rocky Linux |
|---|---|---|
| Origin | Community project, independent | RHEL-compatible rebuild (successor of CentOS) |
| Package manager | `apt` / `dpkg` (`.deb`) | `dnf` / `rpm` (`.rpm`) |
| Security module | AppArmor | SELinux |
| Firewall | UFW | firewalld |
| Strengths | Simplicity, stability, documentation | Enterprise compatibility, long support cycles |
| Weaknesses | Older packages in stable | More complex setup for beginners |

#### AppArmor vs SELinux

Both are Mandatory Access Control systems implemented as Linux Security Modules.

- **AppArmor** is path-based: profiles define which files and capabilities each program can access. It is simpler to read and write, and profiles can run in `enforce` or `complain` mode.
- **SELinux** is label-based: every file, process and port has a security context, and policies define which contexts can interact. It is more granular and powerful, but much more complex to configure and debug.

#### UFW vs firewalld

Both are front-ends for the kernel's packet filtering (netfilter, through iptables/nftables).

- **UFW** (Uncomplicated Firewall) uses simple commands (`ufw allow 4242`) and is ideal for a single server with few rules.
- **firewalld** is organized in zones and services, supports runtime and permanent configurations, and is better suited for complex or changing network setups.

#### VirtualBox vs UTM

- **VirtualBox** (Oracle) is a free, cross-platform type-2 hypervisor for x86 hosts (Windows, Linux, Intel macOS). It offers a complete GUI, snapshots and networking options.
- **UTM** is a macOS/iOS application based on QEMU. It is the alternative for Apple Silicon Macs, where it can virtualize ARM systems natively or emulate other architectures (with lower performance).

I used VirtualBox because it is the default tool required by the subject and my host supports it.

#### apt vs aptitude

- **apt** is a command-line tool, the standard high-level interface to `dpkg`.
- **aptitude** is also high-level but offers an interactive text interface and a more advanced dependency resolver that can propose alternative solutions to conflicts.

## Resources

- [Debian Administrator's Handbook](https://debian-handbook.info/)
- [Debian Wiki – LVM](https://wiki.debian.org/LVM)
- [Debian Wiki – AppArmor](https://wiki.debian.org/AppArmor)
- [Ubuntu – UFW documentation](https://help.ubuntu.com/community/UFW)
- `man sudoers`, `man login.defs`, `man pam_pwquality`, `man sshd_config`, `man crontab`, `man wall`
- [Rocky Linux documentation](https://docs.rockylinux.org/)
- [SELinux Project](https://github.com/SELinuxProject)

### Use of AI

An AI assistant (Claude) was used to:

- Draft and structure this README, which I then reviewed and adapted to my configuration.
- Review theoretical concepts (LVM, AppArmor, SELinux, sudo, PAM, cron) and prepare for the peer evaluation through mock defense questions.

The installation and configuration of the virtual machine were done by me.
