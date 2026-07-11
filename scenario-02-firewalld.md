# Scenario 02: Opening a Firewall Port for a Web Application (firewalld)

## Problem Statement

The Nautilus system administrators rolled out a new backup utility
web UI on an application server. The application listens on port
5000, but incoming traffic was being blocked at the firewall level,
preventing access to the tool.

## Environment

- OS: RHEL-based (AlmaLinux)
- Target: Application server
- Requirement:
  - Install and enable `firewalld`
  - Allow incoming connections on port `5000/tcp`
  - Ensure the rule applies to the `public` zone

## Solution

### Step 1 -- Install firewalld

```bash
sudo yum install firewalld -y
```

### Step 2 -- Start the service immediately

```bash
sudo systemctl start firewalld
```

### Step 3 -- Enable the service to persist across reboots

```bash
sudo systemctl enable firewalld
```

### Step 4 -- Verify the service is active

```bash
sudo systemctl status firewalld
```

### Step 5 -- Open the required port on the public zone

```bash
sudo firewall-cmd --zone=public --add-port=5000/tcp --permanent
```

### Step 6 -- Reload to apply the permanent rule

```bash
sudo firewall-cmd --reload
```

### Step 7 -- Verify the port is open

```bash
sudo firewall-cmd --zone=public --list-ports
```

Expected output includes `5000/tcp` in the list.

## Key Concepts

### start vs enable

| Command | Effect |
|---------|--------|
| `systemctl start`  | Runs the service now only |
| `systemctl enable` | Makes the service start automatically on every boot |

Both are typically needed together: `start` for immediate effect,
`enable` for persistence across reboots.

### --permanent vs --reload

Firewalld rules exist in two states:
- **Runtime** -- the currently active rule set in memory
- **Permanent** -- the rule set saved to configuration files

Using `--permanent` writes the rule to config **but does not apply it
immediately**. `--reload` re-reads the permanent configuration and
applies it to the runtime state, without dropping existing
connections.

**Correct order is always:** add the rule with `--permanent`, then
`--reload`. Reversing this order, or forgetting `--reload` entirely,
means the rule is saved but not actually active -- a very common
mistake.

## What I Learned

- The distinction between a service **running now** vs **running
  persistently after reboot** (`start` vs `enable`) is a recurring
  pattern across many Linux services, not just firewalld.
- Firewalld's permanent/runtime split mirrors a broader Linux concept
  I encountered again later with SELinux (`/etc/selinux/config` vs
  `setenforce`) and with `ulimits` (config file vs active session) --
  configuration files define the *intended* state, while a separate
  mechanism activates it into the *current* state.
- Verification should check the **actual applied state**
  (`--list-ports`), not just assume the command succeeded.

## Relevance

Opening a specific port for a specific service, rather than disabling
the firewall broadly, reflects the principle of least privilege --
a standard practice in production environments and a frequently
tested RHCSA-level skill.
