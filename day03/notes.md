# Linux Fundamentals for Security: Piping, SSH, Cron, Permissions + Bandit Practice

A self-study guide covering shell I/O redirection, SSH basics, scheduled jobs, file permission/SUID concepts, and hands-on practice via OverTheWire Bandit (levels 11–20).

---

## Table of Contents

1. [Piping and Redirection](#1-piping-and-redirection)
2. [SSH Basics](#2-ssh-basics)
3. [Cron Jobs](#3-cron-jobs)
4. [File Permissions and SUID](#4-file-permissions-and-suid)
5. [Practice: OverTheWire Bandit 11–20](#5-practice-overthewire-bandit-1120)
6. [Resources](#6-resources)
7. [Self-Check Checklist](#7-self-check-checklist)

---

## 1. Piping and Redirection

### 1.1 Core concept

Every process has three standard streams:

| Stream | Number (fd) | Purpose |
|---|---|---|
| stdin  | 0 | input to a program |
| stdout | 1 | normal output |
| stderr | 2 | error output |

Redirection and piping let you control **where these streams go** instead of the default (keyboard in, terminal out).

### 1.2 Redirection operators

| Operator | Meaning | Example |
|---|---|---|
| `>` | Redirect stdout to a file, **overwrite** | `echo "hi" > file.txt` |
| `>>` | Redirect stdout to a file, **append** | `echo "more" >> file.txt` |
| `<` | Redirect stdin from a file | `sort < names.txt` |
| `2>` | Redirect stderr only, overwrite | `ls badfile 2> errors.txt` |
| `2>>` | Redirect stderr only, append | `ls badfile 2>> errors.txt` |
| `&>` or `> file 2>&1` | Redirect **both** stdout and stderr | `cmd &> all.log` |
| `2>&1` | Send stderr to wherever stdout is currently going | `cmd > out.log 2>&1` |
| `<<EOF ... EOF` | Here-document: feed multi-line stdin inline | see below |
| `<<<` | Here-string: feed a single string as stdin | `grep foo <<< "$var"` |

**Order matters** with `2>&1`:
```bash
cmd > out.log 2>&1   # correct: stdout -> out.log, then stderr follows stdout -> out.log
cmd 2>&1 > out.log   # wrong: stderr goes to old stdout (terminal), then stdout -> out.log
```

**Here-document example:**
```bash
cat <<EOF > script.txt
Line one
Line two
EOF
```

### 1.3 Piping (`|`)

A pipe connects the **stdout of one command to the stdin of the next**, letting you chain small tools into a pipeline.

```bash
cat access.log | grep "ERROR" | sort | uniq -c | sort -nr | head -5
```
This reads a log, filters error lines, sorts them, counts duplicates, sorts by frequency, and shows the top 5 — a very common recon/analysis pattern.

Key points:
- Only stdout is piped by default — stderr still goes to the terminal unless you add `2>&1` before the pipe.
- `tee` lets you split a stream: write to a file **and** pass it along.
  ```bash
  cmd | tee output.txt | grep "pattern"
  ```
- `xargs` converts piped input into arguments for another command (useful when a command doesn't read stdin directly):
  ```bash
  find . -name "*.txt" | xargs grep "TODO"
  ```

### 1.4 Practice drills
1. Write the last 20 lines of `/var/log/syslog` (or any log) to `recent.log` using `>`.
2. Append your username and date to a file called `log.txt` every time you run a script.
3. Redirect only errors from `ls /root /home` into `errors.txt` while letting normal output print to screen.
4. Use a pipeline to find the 3 most common words in a text file:
   ```bash
   cat file.txt | tr ' ' '\n' | sort | uniq -c | sort -nr | head -3
   ```
5. Use `tee` to simultaneously log command output to a file and grep it live.

---

## 2. SSH Basics

### 2.1 What SSH does
SSH (Secure Shell) provides an encrypted channel to log into and run commands on a remote machine, replacing insecure protocols like telnet/rlogin.

### 2.2 Basic connection
```bash
ssh username@hostname
ssh username@hostname -p 2222        # non-default port
ssh -i ~/.ssh/id_ed25519 user@host   # specify a private key
```

### 2.3 Key-based authentication
1. Generate a key pair:
   ```bash
   ssh-keygen -t ed25519 -C "your_email@example.com"
   ```
   Produces `id_ed25519` (private, **never share**) and `id_ed25519.pub` (public).
2. Copy the public key to the server:
   ```bash
   ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
   ```
   Or manually append it to `~/.ssh/authorized_keys` on the server.
3. Now you can log in without a password (and can disable password auth server-side for better security).

### 2.4 Useful SSH features
- **Copying files:**
  ```bash
  scp file.txt user@host:/remote/path/
  scp -P 2222 user@host:/remote/file.txt ./local/
  ```
  Or with rsync (better for large/incremental transfers):
  ```bash
  rsync -avz -e "ssh -p 2222" ./localdir/ user@host:/remote/dir/
  ```
- **Config file** (`~/.ssh/config`) to save connection shortcuts:
  ```
  Host bandit
      HostName bandit.labs.overthewire.org
      User bandit0
      Port 2220
  ```
  Then just run `ssh bandit`.
- **Port forwarding / tunneling:**
  ```bash
  ssh -L 8080:localhost:80 user@host   # local port forward
  ```
- `known_hosts` file: stores fingerprints of servers you've connected to before, to detect man-in-the-middle attacks (you'll see a warning if a host key changes unexpectedly).

### 2.5 Practice drills
1. Generate an SSH key pair and inspect both files (`cat id_ed25519.pub`).
2. Connect to a test server on a non-standard port using `-p`.
3. Create an `~/.ssh/config` entry for a server you use often.
4. Copy a file to and from a remote host using `scp`.

---

## 3. Cron Jobs

### 3.1 What cron is
`cron` is a Linux daemon that runs commands automatically on a schedule. Each user can have their own **crontab** (cron table).

### 3.2 Crontab syntax
```
* * * * * command-to-run
│ │ │ │ │
│ │ │ │ └── day of week (0–7, both 0 and 7 = Sunday)
│ │ │ └──── month (1–12)
│ │ └────── day of month (1–31)
│ └──────── hour (0–23)
└────────── minute (0–59)
```

Examples:
| Schedule | Meaning |
|---|---|
| `0 * * * *` | Every hour, on the hour |
| `*/15 * * * *` | Every 15 minutes |
| `0 2 * * *` | Every day at 2:00 AM |
| `0 0 * * 0` | Every Sunday at midnight |
| `30 6 1 * *` | 6:30 AM on the 1st of every month |

### 3.3 Managing crontabs
```bash
crontab -l         # list your current crontab
crontab -e         # edit your crontab
crontab -r         # remove your crontab
sudo crontab -u username -l   # view another user's crontab (as root)
```
System-wide jobs also live in `/etc/crontab`, `/etc/cron.d/`, and `/etc/cron.{hourly,daily,weekly,monthly}/`.

### 3.4 Security relevance
Cron is a classic **privilege escalation** vector in CTFs and real systems:
- If a root cron job runs a script that is **writable by a non-root user**, that user can inject commands that execute as root.
- If a cron job calls a binary using a relative path (no full path) and `PATH` is manipulable, an attacker can plant a malicious binary earlier in `PATH`.
- Always check `ls -la` permissions on any script referenced by `/etc/crontab` or `/etc/cron.d/*`.

### 3.5 Practice drills
1. Add a cron job that appends the current date to a file every minute; watch it run with `cat`.
2. Write a job that runs a backup script every day at 3 AM.
3. Find and read `/etc/crontab` on a system you have access to; identify which scripts run as root and check their write permissions.

---

## 4. File Permissions and SUID

### 4.1 Basic permission model
```
-rwxr-xr--  1 alice devs  4096 Jan 1 10:00 script.sh
```
- First character: file type (`-` file, `d` directory, `l` symlink)
- Next 9 characters, in 3 groups of 3: **owner / group / others**, each `r` (read), `w` (write), `x` (execute)

```bash
chmod 750 file        # owner: rwx, group: r-x, others: ---
chmod u+x file        # add execute for owner
chmod g-w file        # remove write for group
chown alice:devs file # change owner and group
```

Numeric permission reference:
| Value | Meaning |
|---|---|
| 4 | read |
| 2 | write |
| 1 | execute |
| 7 = 4+2+1 | rwx |
| 5 = 4+1 | r-x |

### 4.2 SUID, SGID, Sticky Bit

**SUID (Set User ID)** — when set on an executable, the program runs with the **file owner's** privileges, not the privileges of the user running it.
```bash
chmod u+s /path/to/binary     # set SUID
chmod 4755 /path/to/binary    # leading 4 = SUID
```
Shown as `s` in the owner execute position: `-rwsr-xr-x`.

Classic legitimate example: `/usr/bin/passwd` is SUID root, so a regular user can update `/etc/shadow` (which they can't write directly) via a controlled program.

**SGID (Set Group ID)** — similar, but runs with the file's **group** privileges (or, on a directory, new files inherit the directory's group).
```bash
chmod g+s /path/to/binary   # set SGID
chmod 2755 /path/to/binary  # leading 2 = SGID
```

**Sticky bit** — mainly used on directories (like `/tmp`) so that only a file's owner (or root) can delete/rename it, even if others have write access to the directory.
```bash
chmod +t /some/dir
chmod 1777 /some/dir
```

### 4.3 SUID as a privilege-escalation vector
This is one of the most common CTF/real-world escalation techniques:

1. **Find SUID binaries:**
   ```bash
   find / -perm -4000 -type f 2>/dev/null
   ```
2. **Check if any are unusual** (not a standard system binary like `passwd`, `sudo`, `mount`). Compare against [GTFOBins](https://gtfobins.github.io/) to see if a known binary can be abused to spawn a shell, read files, or write files as its owner.
3. **Example abuse pattern:** if `find` itself is SUID root (misconfigured), you can escalate:
   ```bash
   find . -exec /bin/sh -p \; -quit
   ```
4. **Why this works:** the kernel grants the process the file owner's effective UID for the duration of execution — if that binary lets you execute arbitrary commands or read/write arbitrary files, you inherit that owner's power.

### 4.4 Practice drills
1. Create a file, set permissions to `750` numerically and verify with `ls -l`.
2. Create a small C program or script, `chown root` it in a test VM, set the SUID bit, and observe how it runs with root privileges even when executed by a normal user.
3. Run `find / -perm -4000 -type f 2>/dev/null` on a lab VM and identify what each result does using GTFOBins.
4. Set the sticky bit on a shared test directory and confirm another user can't delete your file inside it.

---

## 5. Practice: OverTheWire Bandit 11–20

**Setup:** `ssh banditN@bandit.labs.overthewire.org -p 2220` (replace `N` with the level number). Each level's password is the login password for the next level.

| Level | Skill focus | What you'll practice |
|---|---|---|
| **11** | Character rotation (ROT13) | Reading a file and decoding a simple substitution cipher (`tr` command) |
| **12** | File compression/archives | Repeated decompression — hexdump, gzip, bzip2, tar, xxd — reversing a multi-layer compressed file |
| **13** | SSH private keys | Using a provided SSH private key (`ssh -i`) to log into the next level directly |
| **14** | Local port / netcat basics | Submitting a password to a local port using `nc localhost <port>` |
| **15** | SSL/TLS connections | Using `openssl s_client` to connect to a port over SSL and submit data |
| **16** | Port scanning | Using `nmap` to scan a range of ports and find the one running SSL, then connect to it |
| **17** | File diffing | Using `diff` to compare two nearly-identical files and spot the password change |
| **18** | Restricted shell / `.bashrc` tricks | Logging in when a custom command runs on login and immediately disconnects you — using `ssh host command` to bypass it |
| **19** | SUID binaries | Using a provided SUID binary to read a file you don't otherwise have permission to read (directly applies Section 4.3 above!) |
| **20** | Networking + SUID binary that connects back | Running a program that connects to a port you control (`nc -l`) to pass data between levels |

### Level-by-level approach notes (hints, not full solutions — work them out!)

- **Level 11:** Look at file content with `cat`; recognize ROT13 pattern (letters shifted by 13); use `tr 'A-Za-z' 'N-ZA-Mn-za-m'` to decode.
- **Level 12:** Copy the file to a writable temp directory (`mktemp -d`) before working — you can't write in the home dir. Use `file <filename>` repeatedly to identify each new format after decompressing (it might alternate gzip/bzip2/tar/zip).
- **Level 13:** `chmod 600` the provided private key first, or SSH will refuse to use it due to permissions being "too open."
- **Level 14:** `nc localhost <port>` then type/paste the current password and press enter.
- **Level 15:** `openssl s_client -connect localhost:<port>` then paste the password.
- **Level 16:** `nmap -p 31000-32000 localhost` to find open ports in range, then test which respond with SSL vs plain text; the "server" that gives you the next level's private key uses SSL.
- **Level 17:** `diff passwords.old passwords.new` to spot exactly which line/password changed.
- **Level 18:** Use `ssh bandit18@... -p 2220 cat readme` as a single non-interactive command so it runs and exits before the `.bashrc` kicks you out.
- **Level 19:** Practice piping is: `find` the SUID binary in your home directory, run it (`./binaryname`), and understand it lets you execute a command *as the next level's user* because it's SUID that user — apply exactly what you learned in Section 4.3.
- **Level 20:** Read the setuid binary's behavior (it connects out to a port on localhost). Set up a listener first with `nc -l -p <port>`, then run the binary so it connects to you and sends the password.

### Extra practice problems (beyond Bandit) to deepen each concept
1. **Piping:** Parse a `/var/log/auth.log`-style file to count failed login attempts per IP address (`grep`, `awk`, `sort`, `uniq -c`).
2. **SSH:** Set up two local VMs (or containers) and configure passwordless SSH between them using key-based auth end-to-end.
3. **Cron:** Deliberately create a "vulnerable" cron job (root cron running a world-writable script) in a lab VM, then escalate privileges by editing that script.
4. **SUID:** Write your own tiny C program `int main(){ setuid(0); system("/bin/sh"); }`, compile it, `chown root`, `chmod u+s` it, and use it to escalate as a low-privilege user — then compare with how Bandit level 19/20 do it.

---

## 6. Resources

- **OverTheWire Bandit** — https://overthewire.org/wargames/bandit/ (the wargame used above; levels 0–34 total)
- **GTFOBins** — https://gtfobins.github.io/ (abuse patterns for SUID/sudo binaries)
- `man bash` → search for `REDIRECTION` section for the authoritative piping/redirection reference
- `man ssh`, `man ssh-keygen`, `man crontab`, `man 5 crontab` for command-level details
- **crontab.guru** — https://crontab.guru/ (interactive cron schedule builder/checker)

---

## 7. Self-Check Checklist

Mark each once you can do it **without looking anything up**:

- [ ] Explain the difference between `>`, `>>`, and `2>&1`, and predict output for a redirection with mixed order of operators
- [ ] Build a 4-stage pipeline combining `grep`, `sort`, `uniq -c`, and `head`
- [ ] Generate an SSH key pair and set up passwordless login to a remote host
- [ ] Write and interpret a crontab line for "every 10 minutes" and "3 AM every Monday"
- [ ] Identify a privilege-escalation risk in a root cron job referencing a writable script
- [ ] Convert between symbolic (`rwxr-xr--`) and numeric (`754`) permission notation from memory
- [ ] Explain why SUID `passwd` is safe but SUID `find` is dangerous
- [ ] Complete Bandit levels 11 through 20 unaided
