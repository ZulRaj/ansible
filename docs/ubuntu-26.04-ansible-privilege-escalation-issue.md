# Troubleshooting Ansible Become Timeout on Ubuntu 26.04

## Overview

This document describes how I diagnosed and resolved an Ansible privilege-escalation issue when managing an Ubuntu 26.04 server from an Ansible control node.

**Environment**

- **Control node:** Ubuntu 26.04
- **Managed host:** Ubuntu 26.04 (`machine2`)
- **Second managed host:** AlmaLinux 10 (`machine3`)
- **Ansible Core:** 2.20.1
- **Python on control node:** 3.14.4
- **Sudo implementation on Ubuntu:** `sudo-rs 0.2.13-0ubuntu1.2`

## 1. The problem

SSH connectivity and ordinary Ansible commands worked successfully. However, commands requiring privilege escalation with `--become` failed.

For example:

```bash
ansible 'machine2@192.168.84.133' -m ping
```

The command succeeded without privilege escalation.

However, running:

```bash
ansible 'machine2@192.168.84.133' -m ping --become
```

produced the following error:

```text
Task failed: Timeout (12s) waiting for privilege escalation prompt
```

The same problem occurred when using `--ask-become-pass`.

## 2. Troubleshooting steps

### Step 1: Verify SSH connectivity

I confirmed that Ansible could connect to the managed host and execute the ping module without privilege escalation.

This established that basic SSH connectivity was working.

### Step 2: Verify sudo manually

I connected to the managed host and executed:

```bash
sudo whoami
```

After entering the correct password, the command returned:

```text
root
```

This confirmed that manual sudo authentication worked.

### Step 3: Test the sudo prompt

I tested sudo with a custom prompt:

```bash
sudo -k
sudo -S -p "TEST-PROMPT:" -u root whoami
```

The command successfully authenticated and returned `root`.

I also tested a command using an Ansible-style prompt:

```bash
sudo -k
sudo -S -p "[sudo via ansible, key=testkey] password:" \
  -u root /bin/sh -c 'echo BECOME-SUCCESS-testkey; whoami'
```

This worked manually as well.

Therefore, the sudo executable could accept the password and execute the command in a manual test, even though Ansible's privilege-escalation process was failing.

### Step 4: Inspect the sudo logs

On the managed host, I monitored sudo logs using:

```bash
sudo journalctl -f _COMM=sudo
```

During the investigation, the logs included:

```text
pam_unix(sudo:auth): conversation failed
pam_unix(sudo:auth): auth could not identify password for [machine2]
```

These messages indicated that a sudo authentication attempt failed because a usable password was not obtained.

### Step 5: Check the sudo implementation

I ran:

```bash
sudo --version
```

The output identified the implementation as:

```text
sudo-rs 0.2.13-0ubuntu1.2
```

The managed host also had the traditional sudo executable available at:

```text
/usr/bin/sudo.ws
```

I verified its existence from the control node:

```bash
ssh machine2@192.168.84.133 'ls -l /usr/bin/sudo.ws'
```

### Step 6: Test an alternative sudo executable

Rather than changing the inventory or repeatedly testing passwords, I instructed Ansible to use the traditional sudo executable directly:

```bash
ansible 'machine2@192.168.84.133' \
  -m ping \
  --become \
  --ask-become-pass \
  -e ansible_become_exe=/usr/bin/sudo.ws
```

**Result: The command succeeded.**

This isolated the problem to the interaction between Ansible's privilege-escalation mechanism and the default sudo implementation, rather than basic SSH connectivity or the user's sudo permissions.

## 3. Permanent solution

I changed the default sudo alternative on the managed Ubuntu host to use the traditional sudo implementation.

First, I checked the available alternatives:

```bash
sudo update-alternatives --display sudo
```

Then I selected the traditional sudo executable:

```bash
sudo update-alternatives --set sudo /usr/bin/sudo.ws
```

I verified the change with:

```bash
sudo --version
```

After switching the default, I returned to the Ansible control node and tested the original command:

```bash
ansible 'machine2@192.168.84.133' \
  -m ping \
  --become \
  --ask-become-pass
```

**Result: Ansible privilege escalation worked successfully without the explicit `ansible_become_exe` override.**

## 4. Alternative solution: Configure Ansible per host

If changing the system-wide sudo default is undesirable, Ansible can instead be configured to use the working executable for this host.

For example, in the inventory:

```ini
machine2@192.168.84.133 ansible_become_exe=/usr/bin/sudo.ws
```

This approach limits the change to Ansible's handling of that host and leaves the system-wide default unchanged.

## 5. Rollback

To restore the system's automatically selected sudo alternative, run this on the managed host:

```bash
sudo update-alternatives --auto sudo
```

Verify the resulting configuration before assuming which implementation is selected.

## 6. Lessons learned

- Successful SSH connectivity does not guarantee successful privilege escalation.
- A working manual sudo command does not necessarily prove that Ansible can handle the same authentication interaction.
- Verbose Ansible output and system logs are valuable for narrowing down the failure.
- Testing an alternative executable can isolate a compatibility issue without immediately changing the inventory or reinstalling packages.
- A successful workaround should be tested before making a permanent configuration change.
- Troubleshooting documentation should record the evidence, the commands used, the confirmed workaround, and any remaining uncertainty.

## Conclusion

The issue was resolved by using `/usr/bin/sudo.ws` instead of the default `sudo-rs` implementation on Ubuntu 26.04.

The investigation showed how to troubleshoot an Ansible `--become` timeout systematically, using manual sudo tests, Ansible verbose output, system logs, and a targeted executable override.

*Note: The successful workaround establishes that changing the sudo executable resolved this environment's failure. It does not, by itself, establish the precise underlying software defect in `sudo-rs` or Ansible.*
