# Ubuntu 26.04: Ansible Privilege Escalation Issue

## Overview

This document describes how I diagnosed and resolved an Ansible privilege-escalation issue when managing an Ubuntu 26.04 server.

The issue occurred when Ansible attempted to execute commands with elevated privileges using `--become`. SSH connectivity and ordinary Ansible commands worked, but privilege escalation timed out while waiting for the sudo password prompt.

### Environment

- **Control node:** Ubuntu 26.04 (`machine1`)
- **Managed host:** Ubuntu 26.04 (`machine2`)
- **Second managed host:** AlmaLinux 10 (`machine3`)
- **Ansible Core:** 2.20.1
- **Sudo implementation on machine2:** `sudo-rs 0.2.13-0ubuntu1.2`

## 1. The Problem

Ansible could connect to the managed host and execute commands without privilege escalation.

For example:

```bash
ansible 'machine2@192.168.84.133' -m ping
```

This command succeeded.

However, adding `--become` caused the task to fail:

```bash
ansible 'machine2@192.168.84.133' -m ping --become
```

The error was:

```text
Task failed: Timeout (12s) waiting for privilege escalation prompt
```

The same problem occurred when I used `--ask-become-pass` to enter the sudo password interactively.

## 2. Troubleshooting Steps

### Step 1: Verify SSH connectivity

I confirmed that Ansible could connect to `machine2` and run the ping module without privilege escalation.

This established that basic SSH connectivity was working.

### Step 2: Verify sudo manually

I connected to `machine2` and ran:

```bash
sudo whoami
```

After entering the correct password, the command returned:

```text
root
```

This confirmed that manual sudo authentication worked.

### Step 3: Test the sudo password prompt

I tested sudo using a custom prompt:

```bash
sudo -k
sudo -S -p "TEST-PROMPT:" -u root whoami
```

The command succeeded.

I also tested a command using an Ansible-style password prompt:

```bash
sudo -k
sudo -S -p "[sudo via ansible, key=testkey] password:" \
  -u root /bin/sh -c 'echo BECOME-SUCCESS-testkey; whoami'
```

This also succeeded manually and returned `root`.

Therefore, sudo could accept the password and execute the command in a manual test, even though Ansible's privilege-escalation process was failing.

### Step 4: Inspect the sudo logs

On `machine2`, I monitored the sudo logs:

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

I checked the installed sudo implementation:

```bash
sudo --version
```

The output identified it as:

```text
sudo-rs 0.2.13-0ubuntu1.2
```

I then checked whether the traditional sudo executable was available:

```bash
ssh machine2@192.168.84.133 'ls -l /usr/bin/sudo.ws'
```

The executable existed:

```text
-rwsr-xr-x 1 root root 282080 Oct 6 13:13 /usr/bin/sudo.ws
```

### Step 6: Test an alternative sudo executable

To isolate the issue, I instructed Ansible to use `/usr/bin/sudo.ws` directly without changing the inventory:

```bash
ansible 'machine2@192.168.84.133' \
  -m ping \
  --become \
  --ask-become-pass \
  -e ansible_become_exe=/usr/bin/sudo.ws
```

**Result: The command succeeded.**

This confirmed that using the traditional sudo executable resolved the privilege-escalation failure in this environment.

## 3. Solution: Configure the Ansible Inventory

Instead of changing the default sudo implementation on `machine2`, I configured Ansible to use `/usr/bin/sudo.ws` for this host.

I added `ansible_become_exe` to the existing inventory entry:

```ini
machine2@192.168.84.133 ansible_become_exe=/usr/bin/sudo.ws
```

I then tested privilege escalation:

```bash
ansible 'machine2@192.168.84.133' \
  -m ping \
  --become \
  --ask-become-pass
```

**Result: The command succeeded without needing the command-line override.**

This was the solution used to resolve the issue. It did not require changing the system-wide default sudo implementation on the managed host.

## 4. Alternative Solution: Change the Default Sudo Alternative

Another option is to change the default sudo alternative on the managed Ubuntu host. This makes the traditional sudo executable the default for system-wide use.

First, inspect the available alternatives on `machine2`:

```bash
sudo update-alternatives --display sudo
```

If `/usr/bin/sudo.ws` is listed as an available alternative, select it:

```bash
sudo update-alternatives --set sudo /usr/bin/sudo.ws
```

Verify the change:

```bash
sudo --version
```

Then return to `machine1` and test Ansible without specifying `ansible_become_exe`:

```bash
ansible 'machine2@192.168.84.133' \
  -m ping \
  --become \
  --ask-become-pass
```

To restore automatic alternative selection, run the following command on `machine2`:

```bash
sudo update-alternatives --auto sudo
```

**Important:** Changing the default alternative affects the system-wide sudo configuration. The inventory method is more targeted because it changes which executable Ansible uses for the specified host.

## 5. Lessons Learned

- Successful SSH connectivity does not guarantee successful privilege escalation.
- Manual sudo authentication can work even when Ansible's privilege-escalation process fails.
- Verbose Ansible output and system logs help narrow down the cause of a failure.
- Testing an alternative executable can isolate a compatibility problem without immediately changing system-wide configuration.
- Inventory variables allow host-specific configuration without changing global Ansible settings.
- A workaround should be tested before making permanent configuration changes.
- Troubleshooting documentation should distinguish verified results from assumptions about the underlying software defect.

## Conclusion

The Ansible privilege-escalation issue was resolved by configuring:

```ini
ansible_become_exe=/usr/bin/sudo.ws
```

in the inventory entry for the affected Ubuntu 26.04 host.

Using the traditional sudo executable allowed Ansible to execute tasks with elevated privileges successfully.

An alternative approach is to change the system-wide sudo alternative on the managed host, so Ansible can use the traditional executable without a host-specific inventory setting.

The successful workaround demonstrated a compatibility issue involving Ansible's privilege-escalation process and the default `sudo-rs` implementation in this environment. The precise underlying software defect was not independently established.
