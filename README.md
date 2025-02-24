<img width="200" src="https://github.com/Rashodc36/fail2ban-auto-ban/blob/main/s-laiba-ali-C0_7D50wZQ0-unsplash.jpg?raw=true" alt="Padlock security image"/>

# fail2ban-auto-ban
- [Scenario Creation](https://github.com/Rashodc36/fail2ban-auto-ban)

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
- Install fail2ban - ```sudo dnf install fail2ban -y```
- Enable fail2ban service - ```sudo systemctl enable --now fail2ban```
- Configure SSH Protection in fail2ban - ```sudo vim /etc/fail2ban/jail.local```
-     Enter the following text, save, and exit:
```
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/secure
maxretry = 3
bantime = 600
findtime = 600
```
- Restart fail2ban service - ```sudo systemctl restart fail2ban```
- Check status of fail2ban - ```sudo fail2ban-client status sshd```

### 2. Configure Firewalld on Main Server Table
- Enable Firewalld - ```sudo systemctl enable --now firewalld```
- Add SSH Protection to Rule - ```sudo firewall-cmd --permanent --add-service=ssh```
- Reload all rules - ```sudo firewall-cmd --reload```
- Block an IP Manually to ensure it's working - ```sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.109.18.13" reject'```
- Reload all rules, which should add 10.109.18.13 - ```sudo firewall-cmd --reload```
- List All Blocked IPs - ````sudo firewall-cmd --list-all````
  &nbsp;&nbsp;&nbsp;If you see 10.109.18.13, great job! you are on the right trach. If you don't see it, repeat your steps...You got this!)




### 3. Set Up Log Monitoring for All Servers
-  Install Logwatch on Each Server - sudo dnf install logwatch -y
-  Configure Logwatch to Send Alerts to the Main Server - ```sudo vim /etc/logwatch/conf/logwatch.conf```
-  Enter the following text, save, and exit
```
Output = mail
MailTo = admin@yourdomain.com
Format = detailed
```
- Test Logwatch - sudo logwatch --mailto admin@yourdomain.com --detail high

---

### 4. Automate Email Alerts for Security Events
- Set Up a .forward File on Main Server - ```echo "your-email@example.com" > ~/.forward```
- Change Permissions for .forward file (critical) - ```chmod 600 ~/.forward```
- Configure Email Sending - sudo dnf install postfix -y
- Enable and start the service - sudo systemctl enable --now postfix
- Send a test email - echo "Test alert from IDS system" | mail -s "Security Alert" your-email@example.com
- Ensure that you have replaced your-email@example.com with your actual email address where you want to receive the message. Once done, wait a few seconds to check if you receive an email
---

### 5. Implement Automatic Blocking for Repeated Attacks
-  Create the Script - ```sudo nano /usr/local/bin/block_attackers.sh```
```
- Enter the following text, save, and exit:
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

### 6. Test and Monitor the Setup
- Test Fail2Ban ```sudo fail2ban-client status sshd```
- Verify Firewall Blocks - ```sudo firewall-cmd --list-all```
- Check Logwatch Reports - ```sudo logwatch --mailto admin@yourdomain.com --detail high```


---


## Summary

In this lab, you have configured Fail2Ban to enhance security on a Linux server by automatically detecting and banning suspicious SSH login attempts. You have also integrate Firewalld for additional protection, set up Logwatch for monitoring, and configured automated email alerts for security notifications. Finally, you have implement a script to dynamically block attackers based on repeated failed login attempts.

By completing this lab, you've gained hands-on experience in intrusion detection, log monitoring, and automating security measures to protect Linux servers from brute-force attacks.

---

## Response Taken

As part of the response, a thorough examination of the blocked IP addresses was conducted to identify any patterns or trends. This involved analyzing the geographical origin of the IP addresses to determine if they were consistently coming from specific regions, which may pose a higher risk. If a common theme was identified, such as multiple failed login attempts originating from a particular part of the world, the corresponding IP range or region could be proactively blocked to further mitigate potential threats

---
