# Scenario 01: Enforcing Per-User Process Limits (ulimits)

## Problem Statement

An application server in a datacenter environment was experiencing
performance degradation due to excessive processes held by a specific
service account (`nfsuser`). The requirement was to enforce hard limits
on the number of processes this user could spawn, preventing resource
exhaustion without needing to restart the server.

## Environment

- OS: RHEL-based (AlmaLinux)
- Target user: `nfsuser`
- Requirement:
  - Soft limit (nproc): 1027
  - Hard limit (nproc): 2025

## Solution

Per-user resource limits in Linux are controlled via
`/etc/security/limits.conf`, which is enforced through the PAM
`pam_limits.so` module at login time.

### Step 1 -- Edit the limits configuration file

```bash
sudo vi /etc/security/limits.conf
```

Added the following lines at the end of the file (after all commented
documentation lines, before `# End of file`):

```
nfsuser    soft    nproc    1027
nfsuser    hard    nproc    2025
```

### Step 2 -- Start a fresh login session for the target user

Limits are read at session/login time, not applied retroactively to
already-open shells. A fresh session is required after editing the file:

```bash
sudo su - nfsuser
```

### Step 3 -- Verify the applied limits

```bash
ulimit -Su   # soft limit for processes (nproc)
ulimit -Hu   # hard limit for processes (nproc)
```

Expected output:
```
1027
2025
```

## Common Pitfall (and how I caught it)

My first verification attempt used `ulimit -Sp` / `-Hp`, which returned
`8` regardless of the change. This was misleading at first, because `-p`
does **not** refer to process count -- it reports the **pipe buffer
size** (in 512-byte blocks). The correct flag for process count
(`nproc`) is `-u`.

| Flag | Resource                             |
|------|---------------------------------------|
| `-p` | Pipe size (buffer for `\|`)            |
| `-u` | Max number of user processes (nproc)  |

This is a well-known point of confusion in `ulimit` usage because both
flags look similar and the man page lists them close together.

## What I Learned

- The distinction between **soft** limits (adjustable up to the hard
  limit by the user) and **hard** limits (ceiling, root-only to raise).
- `limits.conf` changes require a **new login session** to take effect
  -- editing the file alone does not affect already-running shells.
- PAM's `pam_limits.so` (included via `session include system-auth` in
  `/etc/pam.d/su`) is what actually enforces these values -- without it,
  `limits.conf` entries would be silently ignored.
- The full list of controllable resources in `limits.conf`: `core`,
  `data`, `fsize`, `memlock`, `nofile`, `rss`, `stack`, `cpu`, `nproc`,
  `as`, `maxlogins`, `priority`, `locks`, `sigpending`, `msgqueue`,
  `nice`, `rtprio`.

## Relevance

This scenario reflects a real system administration concern:
preventing a single account (often a service or automation account)
from destabilizing a shared server through uncontrolled process
spawning -- a common cause of production incidents.
