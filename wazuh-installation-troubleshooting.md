# Troubleshooting a Wazuh SIEM Installation on Ubuntu ARM64 Using UTM: A Real-World Home Lab Experience

*A practical troubleshooting guide for cybersecurity beginners building a SIEM home lab on an Apple Silicon Mac.*

## 1. Introduction

Building a cybersecurity home lab is one of the best ways to develop practical security engineering skills. A Security Information and Event Management (SIEM) platform such as Wazuh allows security practitioners to explore log collection, security monitoring, endpoint visibility and threat detection.

However, setting up a SIEM platform can be challenging, particularly for beginners working with virtual machines and unfamiliar Linux commands.

During my attempt to deploy Wazuh on an Ubuntu virtual machine running on an Apple Silicon MacBook, I encountered several installation problems. These included insufficient disk space, a partially completed installation, ports already in use and a Wazuh manager startup error.

Rather than abandoning the deployment, I investigated the errors, examined the system's state and continued troubleshooting until I could access the Wazuh dashboard.

This article documents the problems encountered, the diagnostic commands used and the lessons learned. The goal is to help other cybersecurity beginners understand how to approach similar issues systematically.

## 2. Understanding the lab environment

Before troubleshooting an installation, it is important to understand the environment in which the software is running.

My lab used the following setup:

- **Host computer:** Apple Silicon MacBook
- **Virtualisation platform:** UTM
- **Guest operating system:** Ubuntu Server ARM64
- **Reported architecture:** `aarch64`
- **Security platform:** Wazuh
- **Deployment objective:** Wazuh indexer, server and dashboard on one virtual machine

Apple Silicon Macs use ARM-based processors. Consequently, the guest operating system and software packages must support the selected architecture.

During the initial setup, Ubuntu 26.04.1 LTS was encountered. We subsequently moved towards Ubuntu Server 24.04.5 ARM64 for the Wazuh deployment because of compatibility concerns.

Always check the supported operating systems and architectures in the official documentation for the specific Wazuh release you intend to install.

## 3. Problem one: The dashboard installation failed because of insufficient disk space

### The error message

During installation, the indexer and manager stages completed, but the dashboard installation failed with this error:

`E: Write error - write (28: No space left on device)`

### What does this error mean?

Linux error code 28 indicates that a write operation could not complete because the relevant filesystem had insufficient available space.

A virtual machine can have a virtual disk attached and still run out of usable space inside Ubuntu.

It is important to distinguish between:

- The maximum size configured for the virtual disk.
- The capacity available to the Ubuntu filesystem.
- The amount of free space remaining on that filesystem.

These values are not necessarily the same.

### Step 1: Clean the APT package cache

We ran:

```bash
sudo apt clean
```

APT is Ubuntu's package management system. It downloads software packages and stores cached package files locally.

The `apt clean` command removes downloaded package files from the local cache. It does not uninstall installed applications.

This can recover disk space, although it will not solve every storage-related problem.

### Step 2: Check disk usage

We then ran:

```bash
df -h
```

The `df` command displays filesystem capacity, used space and available space. The `-h` option makes the output easier to read.

After cleaning the cache, the root filesystem showed approximately 23 GB total capacity, 6.9 GB used and 15 GB available.

This provided useful evidence that substantial space was available at that point.

However, free space alone does not prove that the dashboard installation will succeed. The installation process must still be checked for additional errors.

**Lesson learned:** When an installation reports that no space is available, investigate the filesystem before assuming the software package itself is faulty.

## 4. Problem two: The installer reported that Wazuh was already installed

After the initial installation failed, we attempted to run the all-in-one installation command again:

```bash
sudo bash ./wazuh-install.sh -a
```

The installer returned the following message:

`ERROR: Wazuh manager already installed`

### Why was this a problem?

The earlier attempt had already installed at least some Wazuh components.

The dashboard failure did not mean that the entire installation had been rolled back. The machine was now in a partially installed state.

This is an important concept for beginners: an installation can fail halfway through, leaving some services and packages installed while other components remain incomplete.

Repeating the original command may therefore produce a different error from the first attempt.

### What about the overwrite option?

The Wazuh installation script also supports an overwrite option:

```bash
sudo bash ./wazuh-install.sh -a -o
```

This option should not be used blindly as a general repair command. Its effects depend on the installer and existing installation state.

Before using it, review the documentation for the relevant Wazuh release and understand what may be replaced or reconfigured.

In our troubleshooting process, the subsequent attempt exposed another issue: ports required by Wazuh were already occupied.

**Lesson learned:** Before repeating an installation, determine what has already been installed and whether the installer expects a clean system.

## 5. Problem three: Ports 1515 and 55000 were already in use

During the subsequent installation attempt, the installer reported that ports 1515 and 55000 were already in use.

A network port is a numbered endpoint that an application or service can use to communicate with other applications or systems.

When a service listens on a port, another process may be unable to bind to the same address and port combination.

### Step 1: Investigate the listening ports

Rather than immediately terminating processes, we ran:

```bash
sudo ss -ltnp | grep -E '1515|55000'
```

Here is what the command does:

- `sudo` runs the command with elevated privileges.
- `ss` displays socket information.
- `-l` displays listening sockets.
- `-t` selects TCP sockets.
- `-n` displays numeric port numbers.
- `-p` displays process information where available.
- `grep -E` filters the output for either port 1515 or port 55000.

The output identified:

- `wazuh-authd` listening on port 1515.
- A `python3` process listening on port 55000.

This was more useful than simply knowing that the ports were occupied. We had information about which processes were listening.

### Step 2: Understand the ports

**Port 1515/TCP**

Wazuh commonly uses this port for agent enrollment. It allows agents to register with the Wazuh manager when enrollment is configured to use this service.

**Port 55000/TCP**

Wazuh commonly uses this port for its server API, which supports management and integration operations.

The exact process and service configuration should always be checked against the installed version.

### Step 3: Why should you avoid killing processes immediately?

A process listening on a port is not automatically a problem. It might be a legitimate service that is already running correctly.

Before stopping a process, investigate:

1. Which application owns the process?
2. Is the corresponding service supposed to be running?
3. Does the installation log explain why the port is needed?
4. Would stopping the process interrupt an existing installation or service?

The safer approach is to understand the existing system state before making changes.

**Lesson learned:** A port conflict is a symptom that needs investigation. Identifying the listening process provides a starting point for determining the appropriate fix.

## 6. Problem four: The Wazuh manager reported a missing executable

During subsequent troubleshooting, we also encountered an error involving a missing `wazuh-keystore` executable and a Wazuh manager that failed to start.

This was a separate problem from the original disk-space and port-conflict errors.

### Why does this matter?

An error involving a missing executable may indicate an incomplete installation, a missing package file, an incorrect executable path or another installation-state problem.

The exact cause cannot be determined from the executable's name alone.

A disciplined investigation would involve:

- Capturing the complete error message.
- Checking the Wazuh manager's service status.
- Reviewing the relevant service and installation logs.
- Checking whether the executable exists in the expected location.
- Confirming that the installed packages and configuration are consistent with the Wazuh version.

It is important not to assume that resolving one installation error automatically resolves another.

In this case study, the missing-executable error is documented as part of the troubleshooting history, but its individual root cause has not been conclusively established.

**Lesson learned:** Treat each error as a separate diagnostic problem until there is evidence that the underlying cause is shared.

## 7. Rebooting the virtual machine and checking the ports again

During troubleshooting, we rebooted the Ubuntu virtual machine and repeated the port check:

```bash
sudo ss -ltnp | grep -E '1515|55000'
```

The command returned no output for those ports after the reboot.

This established that the earlier matching listening sockets were no longer present at the time of the check.

However, it is important to interpret this result correctly. No output from the command does not prove that Wazuh has been completely installed or that all its services are healthy.

A reboot can clear transient process state, but it cannot repair every possible configuration or installation problem.

The important principle is to verify what changed after each troubleshooting step instead of assuming that a single successful check resolves the entire deployment.

## 8. Final result: Accessing the Wazuh dashboard

After continuing the troubleshooting process, I was able to access the Wazuh dashboard successfully.

This was an important milestone in the lab deployment because the dashboard provides the interface through which an administrator can explore security events, review information collected by the platform and manage monitored endpoints.

Nevertheless, dashboard accessibility is not sufficient evidence that every backend component is healthy.

To verify the deployment more thoroughly on a systemd-based Ubuntu installation, inspect the status of the three main services:

```bash
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```

Run these commands individually and examine the output for errors.

A more complete verification should establish that:

- The Wazuh indexer is running.
- The Wazuh manager is running.
- The dashboard service is running.
- The services communicate correctly.
- The dashboard can retrieve information from the backend.
- The deployment remains operational after a controlled reboot.

The exact verification procedure should follow the documentation for the installed Wazuh release.

**Confirmed outcome:** I was able to open the Wazuh dashboard and access the interface successfully.

**Additional verification:** The individual service statuses and backend health should be recorded before claiming that every component has been fully validated.

## 9. Lessons for cybersecurity beginners

This troubleshooting experience reinforced several important principles.

### 1. Read the error message carefully

A disk-space error, an existing installation, a port conflict and a missing executable represent different failure conditions. Each requires an appropriate investigation.

### 2. Understand the environment before making changes

Knowing the operating system, processor architecture, virtualisation platform and available disk space helps narrow down potential causes.

### 3. Investigate before repeating installation commands

A failed installation may leave behind packages, services and processes. Check the current state before deciding whether to retry, repair or reinstall.

### 4. Avoid unnecessary process termination

Identify what is listening on a port and understand why it is running before stopping it.

### 5. Verify every troubleshooting step

If a port is no longer occupied, record that result. If the dashboard opens, record that too. These are separate observations, and neither should be presented as proof of something that has not been checked.

### 6. Keep a record of the investigation

Save error messages, commands, outputs, configuration details and final verification results. This creates a reproducible troubleshooting record that can help other people facing similar challenges.

## 10. Conclusion

Building a cybersecurity home lab is about more than installing security software. It is also an opportunity to develop practical skills in Linux administration, service management, networking and systematic troubleshooting.

During this Wazuh deployment on Ubuntu ARM64 using UTM, I encountered disk-space limitations, a partial installation, occupied ports and a manager startup error.

By investigating the environment and continuing the troubleshooting process, I ultimately gained access to the Wazuh dashboard.

The main lesson is that installation failures should be approached methodically. Identify the problem, collect evidence, make changes carefully and verify the outcome.

I hope this case study helps other cybersecurity beginners approach their own SIEM deployment challenges with greater confidence and a clearer troubleshooting process.

## References

- Wazuh documentation: https://documentation.wazuh.com/
- Ubuntu Server documentation: https://documentation.ubuntu.com/server/
- Linux `df` manual: https://man7.org/linux/man-pages/man1/df.1.html
- Linux `ss` manual: https://man7.org/linux/man-pages/man8/ss.8.html
- Linux `systemctl` manual: https://www.freedesktop.org/software/systemd/man/latest/systemctl.html

*Author's note: This article documents a personal lab troubleshooting experience. Observed results are distinguished from additional verification steps so that readers can understand both the progress made and the checks required for a fully validated deployment.*
