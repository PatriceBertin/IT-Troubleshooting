# **NordFabric GmbH: Complete Enterprise IT Troubleshooting Project**

## **Company Profile**
**Name:** NordFabric GmbH  
**Headquarters:** Munich, Germany  
**Industry:** Automotive Textile Manufacturing  
**Employees:** 2,400 (1,200 production, 800 office, 400 remote)  
**Facilities:** 
- 4 production plants (Germany, Poland, Czech Republic) 
- Central data center in Frankfurt  
**Critical Systems:**  
- SAP S/4HANA (Production Planning)  
- AutoCAD/CATIA Workstations  
- IoT-enabled Looms & Weaving Machines  
- Citrix Virtual Desktop Infrastructure  

---

## **1. SAP Production Server Outage (Full Resolution)**

### **Incident Report**
**Date:** March 15, 2024 @ 06:30 CET  
**Reported By:** Berlin Plant Manager  
**Impact:** All production lines halted, unable to process work orders  

### **Detailed Troubleshooting Process**

#### **Step 1: Initial Connectivity Checks**
```bash
# Basic network connectivity
ping -c 4 sap-prod-01.nordfabric.local
# PING sap-prod-01.nordfabric.local (10.20.1.100) 56(84) bytes of data
# Request timeout for icmp_seq 0...

# Check if SSH port is responsive
nc -zv 10.20.1.100 22
# Connection to 10.20.1.100 port 22 [tcp/ssh] succeeded
```

#### **Step 2: Remote Server Access**
```bash
# Connect via SSH to investigate
ssh root@10.20.1.100

# Check running processes
top -n 1 -b | head -20
# Shows high CPU usage by jbd2 process

# Check disk space
df -hT / /sap
# Filesystem     Type  Size  Used Avail Use% Mounted on
# /dev/mapper/vg-root xfs   50G   49G  1.0G  98% /
# /dev/mapper/vg-sap  xfs  200G  198G  2.0G  99% /sap
```

#### **Step 3: Filesystem Repair**
```bash
# Unmount affected filesystems
umount /sap
umount /var

# Run filesystem check
fsck -y /dev/mapper/vg-sap
# Phase 1: Check inodes, blocks, and sizes
# Phase 2: Check directory structure
# Phase 3: Check directory connectivity
# Phase 4: Check reference counts
# Phase 5: Check group summary information
# Free inodes count wrong (9821, counted=9783)
# Fix? yes

# Repair XFS filesystem (if needed)
xfs_repair /dev/mapper/vg-sap
```

#### **Step 4: Storage System Diagnostics**
```bash
# Check Pure Storage array status
purestorage array list --detail
# Name       Status  Version  Controllers
# purefa-01  DEGRADED 6.1.5   [A:OFFLINE, B:ACTIVE]

# Check volume status
purestorage volume show --iops
# Name          Size  IOPS  Latency  Status
# sap_data      10TB  0     N/A      DEGRADED
```

#### **Step 5: Database Recovery**
```bash
# Connect to Oracle
su - oracle
sqlplus / as sysdba

# Recovery commands
SQL> STARTUP MOUNT;
SQL> RECOVER DATABASE USING BACKUP CONTROLFILE;
SQL> ALTER DATABASE OPEN RESETLOGS;
```

#### **Step 6: Service Restoration**
```bash
# Start SAP services
systemctl start sap-application
systemctl start sap-database

# Verify SAP processes
sapcontrol -nr 00 -function GetProcessList
# OK: All processes running
```

### **Root Cause Analysis**
1. Primary storage controller failure during nightly batch processing
2. Filesystem corruption due to incomplete writes
3. Cascading database failures

### **Preventative Measures**
```bash
# Configure storage multipathing
purestorage array set --controller-redundancy high

# Implement monitoring
cat <<EOF > /etc/nagios/check_pure_battery.sh
#!/bin/bash
if purestorage array list | grep -q DEGRADED; then
  echo "CRITICAL: Storage array degraded"
  exit 2
fi
EOF

# Add to crontab for regular filesystem checks
echo "0 3 * * * /usr/sbin/xfs_check /dev/mapper/vg-sap" >> /etc/crontab
```

---

## **2. Engineering Workstation Performance Issues**

### **Detailed Troubleshooting**

#### **Memory Analysis**
```bash
# Check memory usage
free -m
#               total        used        free
# Mem:          64250       63821         429
# Swap:         32768       30720        2048

# Check for memory leaks
valgrind --leak-check=full /opt/CATIA/BN-30/Code/bin/CATIA
```

#### **Process Inspection**
```bash
# Detailed process analysis
ps aux --sort=-%mem | head -10
# USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# engineer  4471 98.3 43.2 29874532 27845124 ?   Rl   Mar15 45:23 /opt/CATIA/BN-30/Code/bin/CATIA

# Check thread count
cat /proc/4471/status | grep Threads
# Threads: 137
```

#### **Storage Performance**
```bash
# Real-time disk I/O monitoring
iostat -x 1 5
# Device            r/s     w/s     rkB/s     wkB/s   await svctm  %util
# nvme0n1          45.2    130.5    180.8    522.0   12.34   6.78  99.80

# Check for disk errors
smartctl -a /dev/nvme0n1
```

### **Resolution Script**
```bash
#!/bin/bash
# CATIA performance optimization script

# Increase JVM heap size
sed -i 's/Xmx[0-9]*G/Xmx48G/' /opt/CATIA/BN-30/Code/bin/CATIA.ini

# Configure filesystem cache
sysctl -w vm.swappiness=10
sysctl -w vm.vfs_cache_pressure=50

# Add SSD caching
lvcreate -L 100G -n catia_cache vg_ssd
mkfs.xfs /dev/vg_ssd/catia_cache
mount /dev/vg_ssd/catia_cache /opt/CATIA/cache
```

---

## **3. Plant Network Outage (Full Resolution)**

### **Detailed Troubleshooting**

#### **Layer 1 Diagnostics**
```bash
# Check physical link status
ethtool eth0
# Settings for eth0:
#   Supported ports: [ FIBRE ]
#   Link detected: no

# Verify SFP module
ethtool -m eth0
# Identifier: 0x11 (QSFP)
# Extended identifier: 0x00
# Connector: 0x23 (LC)
# Transceiver type: 10G Ethernet

# Check switch port status
ssh admin@switch-pl-01 "show interface GigabitEthernet1/0/24"
# Port Name               Status     Vlan       Duplex Speed Type
# Gi1/0/24                down       1          full   1000  1000BaseSX
```

#### **Fiber Optic Testing**
```bash
# OTDR fiber test
fluke DTX-1800 --test fiber --length 350 --wavelength 1310
# Test Results:
#   Length: 350.2m
#   Loss: -34.2dB
#   Events:
#     1: 187.5m (-12.3dB) - Severe bend
```

### **Restoration Process**
```bash
# Temporary wireless bridge setup
iwconfig wlan0 mode ad-hoc essid "temp-bridge" channel 6
ifconfig wlan0 192.168.100.1 netmask 255.255.255.0

# Permanent fiber repair
# (Executed by certified fiber technicians)
```

---

## **4. VPN Connectivity Issues (Complete Fix)**

### **Detailed Troubleshooting**

#### **Certificate Management**
```bash
# Check certificate expiration
openssl x509 -in /etc/ssl/certs/vpn.nordfabric.crt -noout -dates
# notBefore=Mar 15 00:00:00 2023 GMT
# notAfter=Mar 14 23:59:59 2024 GMT

# Verify certificate chain
openssl verify -CAfile /etc/ssl/certs/ca-bundle.crt /etc/ssl/certs/vpn.nordfabric.crt
# error 10 at 0 depth lookup: certificate has expired

# Generate new certificates
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/vpn.nordfabric.key \
  -out /etc/ssl/certs/vpn.nordfabric.crt \
  -subj "/CN=vpn.nordfabric.local/O=NordFabric GmbH/C=DE"
```

#### **VPN Service Restoration**
```bash
# Restart VPN services
systemctl restart openvpn@server
systemctl restart radiusd

# Verify service status
ss -tulnp | grep -E '(openvpn|radius)'
# tcp   LISTEN 0      128         0.0.0.0:1194       0.0.0.0:*    users:(("openvpn",pid=1123,fd=6))
# udp   LISTEN 0      128         0.0.0.0:1812       0.0.0.0:*    users:(("radiusd",pid=1156,fd=7))
```

---

## **5. Label Printer Failure (Complete Fix)**

### **Detailed Troubleshooting**

#### **CUPS Debugging**
```bash
# Check printer status
lpstat -t
# scheduler is running
# system default destination: label-printer-01
# device for label-printer-01: socket://label-printer-01:9100
# label-printer-01 accepting requests since Mon Mar 15 06:00:01 2024
# printer label-printer-01 disabled since Mon Mar 15 06:00:01 2024 -
#   83 jobs queued

# Enable debug logging
cupsctl --debug-logging
tail -f /var/log/cups/error_log
```

#### **Driver Reinstallation**
```bash
# Remove broken printer
lpadmin -x label-printer-01

# Reinstall with correct driver
lpadmin -p label-printer-01 -v socket://label-printer-01:9100 \
  -m drv:///samsung/laser.ppd \
  -o printer-is-shared=false \
  -o job-sheets-default=none,none \
  -o sides=one-sided

# Test print
echo "TEST PRINT" | lp -d label-printer-01
```

---

## **Complete Monitoring Solution**

### **Nagios Configuration**
```bash
# SAP monitoring plugin
cat <<EOF > /usr/lib/nagios/plugins/check_sap
#!/bin/bash
STATUS=$(sapcontrol -nr 00 -function GetProcessList | grep -c GREEN)
if [ \$STATUS -lt 4 ]; then
  echo "CRITICAL: SAP processes down"
  exit 2
else
  echo "OK: All SAP processes running"
  exit 0
fi
EOF

# Storage monitoring
cat <<EOF > /usr/lib/nagios/plugins/check_pure
#!/bin/bash
if purestorage array list | grep -q DEGRADED; then
  echo "CRITICAL: Storage array degraded"
  exit 2
fi
echo "OK: Storage array healthy"
exit 0
EOF
```

### **Automated Recovery System**
```python
#!/usr/bin/env python3
# sap_watchdog.py

import subprocess
import smtplib
from datetime import datetime

def check_sap():
    try:
        result = subprocess.run(
            ['sapcontrol', '-nr', '00', '-function', 'GetProcessList'],
            capture_output=True,
            text=True,
            timeout=30
        )
        if "GREEN" not in result.stdout:
            restart_sap()
    except Exception as e:
        alert_team(f"SAP check failed: {str(e)}")

def restart_sap():
    subprocess.run(['systemctl', 'restart', 'sap-application'])
    log_event("SAP services restarted")

def alert_team(message):
    with smtplib.SMTP('smtp.nordfabric.local') as server:
        server.sendmail(
            'it-alerts@nordfabric.local',
            'sap-team@nordfabric.local',
            f"Subject: SAP Alert\n\n{message}"
        )

if __name__ == "__main__":
    check_sap()
```

