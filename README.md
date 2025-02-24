<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# fail2ban-auto-ban
- [Scenario Creation](https://github.com/joshmadakor0/threat-hunting-scenario-tor/blob/main/threat-hunting-scenario-tor-event-creation.md)

## Platforms and Languages Leveraged
- Red Hat 9/Ubuntu Virtual Machines
- Fail2ban: Red Hat 9 CLI
- Bash Scripting Language
- Gmail

##  Scenario

As a Linux System Administrator, you are responsible for securing SSH access to critical infrastructure. To mitigate unauthorized login attempts and potential brute-force attacks, you have implemented Fail2Ban across multiple servers. Your goal is to configure Fail2Ban to monitor authentication logs, automatically ban suspicious IP addresses, and send email alerts upon repeated failed login attempt

---

## Steps Taken

### 1. Configure Fail2Ban to detect SSH login failures and enforce bans
- Install fail2ban - sudo dnf install fail2ban -y
- Enable fail2ban service - sudo systemctl enable --now fail2ban
- Configure SSH Protection in fail2ban - sudo vim /etc/fail2ban/jail.local
```kql
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 600
findtime = 600
```
- Restart fail2ban service - sudo systemctl restart fail2ban
- Check status of fail2ban - sudo fail2ban-client status sshd

**Query used to locate events:**

```
<img width="1212" alt="image" src="https://github.com/user-attachments/assets/71402e84-8767-44f8-908c-1805be31122d">

---
```
### 2. Configure Firewalld on Main Server Table
- Enable Firewalld - ```sudo systemctl enable --now firewalld```
- Add SSH Protection to Rule -sudo firewall-cmd --permanent --add-service=ssh
- Reload all rules - sudo firewall-cmd --reload
- Block an IP Manually to ensure it's working - sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.109.18.13" reject'
- Reload all rules, which should add 10.109.18.13 - sudo firewall-cmd --reload
- List All Blocked IPs - sudo firewall-cmd --list-all (If you see 10.109.18.13, great job! you are on the right trach. If you don't see it, repeat your steps...You got this!)

Searched for any `ProcessCommandLine` that contained the string "tor-browser-windows-x86_64-portable-14.0.1.exe". Based on the logs returned, at `2024-11-08T22:16:47.4484567Z`, an employee on the "threat-hunt-lab" device ran the file `tor-browser-windows-x86_64-portable-14.0.1.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

<img width="1212" alt="image" src="https://github.com/user-attachments/assets/b07ac4b4-9cb3-4834-8fac-9f5f29709d78">

---

### 3. Set Up Log Monitoring for All Servers
-  Install Logwatch on Each Server - sudo dnf install logwatch -y
-  Configure Logwatch to Send Alerts to the Main Server - sudo vim /etc/logwatch/conf/logwatch.conf
```
Output = mail
MailTo = admin@yourdomain.com
Format = detailed
```
- Test Logwatch - sudo logwatch --mailto admin@yourdomain.com --detail high


<img width="1212" alt="image" src="https://github.com/user-attachments/assets/b13707ae-8c2d-4081-a381-2b521d3a0d8f">

---

### 4. Automate Email Alerts for Security Events
- Set Up a .forward File on Main Server - ```echo "your-email@example.com" > ~/.forward```
- Change Permissions for .forward file (critical) - ```chmod 600 ~/.forward```
- Configure Email Sending - sudo dnf install postfix -y
- Enable and start the service - sudo systemctl enable --now postfix
- Send a test email - echo "Test alert from IDS system" | mail -s "Security Alert" your-email@example.com
- MAke sure that you have changed the your-email@example.com to your actual email where you want the email to be sent to. After complete, wait a couple second to see if you have receieved an email

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `2024-11-08T22:18:01.1246358Z`, an employee on the "threat-hunt-lab" device successfully established a connection to the remote IP address `176.198.159.33` on port `9001`. The connection was initiated by the process `tor.exe`, located in the folder `c:\users\employee\desktop\tor browser\browser\torbrowser\tor\tor.exe`. There were a couple of other connections to sites over port `443`.

<img width="1212" alt="image" src="https://github.com/user-attachments/assets/87a02b5b-7d12-4f53-9255-f5e750d0e3cb">

### 4. Implement Automatic Blocking for Repeated Attacks
-  Create the Script - ```sudo nano /usr/local/bin/block_attackers.sh```
```
#!/bin/bash
LOG=/var/log/secure
ATTEMPTS=5
BAN_TIME=3600  # 1 hour

grep "Failed password" $LOG | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | while read count ip; do
    if [ $count -gt $ATTEMPTS ]; then
        echo "Blocking $ip due to $count failed login attempts"
        sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='$ip' reject"
        sudo firewall-cmd --reload
    fi
done
```
- Make It Executable - sudo chmod +x /usr/local/bin/block_attackers.sh
- Add to Crontab (Runs Every 10 Minutes) - ```sudo crontab -e```
  - Enter the following: ```*/10 * * * * /usr/local/bin/block_attackers.sh```



---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2024-11-08T22:14:48.6065231Z`
- **Event:** The user "employee" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.1.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\employee\Downloads\tor-browser-windows-x86_64-portable-14.0.1.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2024-11-08T22:16:47.4484567Z`
- **Event:** The user "employee" executed the file `tor-browser-windows-x86_64-portable-14.0.1.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.0.1.exe /S`
- **File Path:** `C:\Users\employee\Downloads\tor-browser-windows-x86_64-portable-14.0.1.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2024-11-08T22:17:21.6357935Z`
- **Event:** User "employee" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\employee\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `2024-11-08T22:18:01.1246358Z`
- **Event:** A network connection to IP `176.198.159.33` on port `9001` by user "employee" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `c:\users\employee\desktop\tor browser\browser\torbrowser\tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2024-11-08T22:18:08Z` - Connected to `194.164.169.85` on port `443`.
  - `2024-11-08T22:18:16Z` - Local connection to `127.0.0.1` on port `9150`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "employee" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2024-11-08T22:27:19.7259964Z`
- **Event:** The user "employee" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\employee\Desktop\tor-shopping-list.txt`

---

## Summary

The user "employee" on the "threat-hunt-lab" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `threat-hunt-lab` by the user `employee`. The device was isolated, and the user's direct manager was notified.

---
