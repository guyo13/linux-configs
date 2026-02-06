# **Fedora 43 Server Hardening SOP**

**Target:** DigitalOcean Droplet (Minimal)

**Author:** Server Admin

**Baseline Version:** 1.0

This document outlines the exact commands and configuration settings used to create a "Gold Master" security baseline for Fedora 43\.

## **1\. Initial User Setup**

Execute these commands to move away from the root user.

\# Create user and set password  
adduser \<yourname\>  
passwd \<yourname\>

\# Add to sudoers (wheel group)  
usermod \-aG wheel \<yourname\>

\# Migrate SSH keys  
mkdir \-p /home/\<yourname\>/.ssh  
cp /root/.ssh/authorized\_keys /home/\<yourname\>/.ssh/  
chown \-R \<yourname\>:\<yourname\> /home/\<yourname\>/.ssh  
chmod 700 /home/\<yourname\>/.ssh  
chmod 600 /home/\<yourname\>/.ssh/authorized\_keys

\# Fix SELinux context for the keys  
restorecon \-Rv /home/\<yourname\>/.ssh

## **2\. SSH Hardening**

Edit /etc/ssh/sshd\_config and ensure the following lines are set (uncommented).

PermitRootLogin no  
PasswordAuthentication no  
MaxAuthTries 3  
\# Add this to the very bottom:  
AllowUsers \<yourname\>

**Verify and Apply:**

sudo sshd \-t  
sudo systemctl reload sshd

## **3\. Firewall & Fail2Ban**

Fedora 43 uses firewalld and systemd-journald.

### **Firewall Configuration**

sudo dnf install firewalld \-y  
sudo systemctl enable \--now firewalld

\# Harden services and target  
sudo firewall-cmd \--permanent \--remove-service=cockpit  
sudo firewall-cmd \--permanent \--remove-service=dhcpv6-client  
sudo firewall-cmd \--permanent \--zone=public \--set-target=DROP  
sudo firewall-cmd \--reload

### **Fail2Ban Setup**

sudo dnf install fail2ban \-y

\# Create configuration: /etc/fail2ban/jail.local  
\# Contents:  
\[DEFAULT\]  
bantime   \= 1h  
findtime  \= 10m  
maxretry  \= 3  
backend   \= systemd

\[sshd\]  
enabled \= true

\# Apply:  
sudo systemctl enable \--now fail2ban

## **4\. Kernel & Filesystem Hardening**

### **Network Parameters (Sysctl)**

Create /etc/sysctl.d/99-hardened.conf and add:

\# Ignore ICMP redirects (MITM protection)  
net.ipv4.conf.all.accept\_redirects \= 0  
net.ipv6.conf.all.accept\_redirects \= 0

\# Ignore broadcast ICMP  
net.ipv4.icmp\_echo\_ignore\_broadcasts \= 1

\# Enable Strict IP spoofing protection  
net.ipv4.conf.all.rp\_filter \= 1  
net.ipv4.conf.default.rp\_filter \= 1

\# Disable Source Routing  
net.ipv4.conf.all.accept\_source\_route \= 0

\# Log packets with impossible addresses  
net.ipv4.conf.all.log\_martians \= 1

**Apply:** sudo sysctl \--system

### **Mount Security (fstab)**

Add these lines to /etc/fstab to prevent execution in temp directories:

tmpfs   /dev/shm   tmpfs   defaults,nodev,nosuid,noexec   0 0  
tmpfs   /tmp       tmpfs   defaults,nodev,nosuid,noexec   0 0

**Apply:** sudo mount \-a

## **5\. Automatic Security Updates**

sudo dnf install dnf5-plugin-automatic \-y  
\# Edit /etc/dnf/automatic.conf:  
\# Set: upgrade\_type \= security  
\# Set: apply\_updates \= yes

sudo systemctl enable \--now dnf5-automatic.timer

## **6\. Pre-Snapshot Sanitization**

Run these commands immediately before powering off for the snapshot.

\# Clear DNF cache  
sudo dnf clean all

\# Clear Logs  
sudo journalctl \--vacuum-time=1s

\# Remove unique machine ID (forces regeneration on new instances)  
sudo truncate \-s 0 /etc/machine-id

\# Clear /tmp and /var/tmp  
sudo find /tmp \-mindepth 1 \-delete 2\>/dev/null  
sudo find /var/tmp \-mindepth 1 \-delete 2\>/dev/null

\# Remove current host keys (ensures unique keys per droplet)  
sudo rm \-f /etc/ssh/ssh\_host\_\*

\# Clear Bash History (The absolute final command)  
cat /dev/null \> \~/.bash\_history && history \-c && history \-w && exit  
