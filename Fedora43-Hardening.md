Fedora 43 Server Hardening SOPTarget: DigitalOcean Droplet (Minimal)Author: Server AdminBaseline Version: 1.0This document outlines the exact commands and configuration settings used to create a "Gold Master" security baseline for Fedora 43.1. Initial User SetupExecute these commands to move away from the root user.# Create user and set password
adduser <yourname>
passwd <yourname>

# Add to sudoers (wheel group)
usermod -aG wheel <yourname>

# Migrate SSH keys
mkdir -p /home/<yourname>/.ssh
cp /root/.ssh/authorized_keys /home/<yourname>/.ssh/
chown -R <yourname>:<yourname> /home/<yourname>/.ssh
chmod 700 /home/<yourname>/.ssh
chmod 600 /home/<yourname>/.ssh/authorized_keys

# Fix SELinux context for the keys
restorecon -Rv /home/<yourname>/.ssh
2. SSH HardeningEdit /etc/ssh/sshd_config and ensure the following lines are set (uncommented).PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
# Add this to the very bottom:
AllowUsers <yourname>
Verify and Apply:sudo sshd -t
sudo systemctl reload sshd
3. Firewall & Fail2BanFedora 43 uses firewalld and systemd-journald.Firewall Configurationsudo dnf install firewalld -y
sudo systemctl enable --now firewalld

# Harden services and target
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --permanent --remove-service=dhcpv6-client
sudo firewall-cmd --permanent --zone=public --set-target=DROP
sudo firewall-cmd --reload
Fail2Ban Setupsudo dnf install fail2ban -y

# Create configuration: /etc/fail2ban/jail.local
# Contents:
[DEFAULT]
bantime   = 1h
findtime  = 10m
maxretry  = 3
backend   = systemd

[sshd]
enabled = true

# Apply:
sudo systemctl enable --now fail2ban
4. Kernel & Filesystem HardeningNetwork Parameters (Sysctl)Create /etc/sysctl.d/99-hardened.conf and add:# Ignore ICMP redirects (MITM protection)
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Ignore broadcast ICMP
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Enable Strict IP spoofing protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable Source Routing
net.ipv4.conf.all.accept_source_route = 0

# Log packets with impossible addresses
net.ipv4.conf.all.log_martians = 1
Apply: sudo sysctl --systemMount Security (fstab)Add these lines to /etc/fstab to prevent execution in temp directories:tmpfs   /dev/shm   tmpfs   defaults,nodev,nosuid,noexec   0 0
tmpfs   /tmp       tmpfs   defaults,nodev,nosuid,noexec   0 0
Apply: sudo mount -a5. Automatic Security Updatessudo dnf install dnf5-plugin-automatic -y
# Edit /etc/dnf/automatic.conf:
# Set: upgrade_type = security
# Set: apply_updates = yes

sudo systemctl enable --now dnf5-automatic.timer
6. Pre-Snapshot SanitizationRun these commands immediately before powering off for the snapshot.# Clear DNF cache
sudo dnf clean all

# Clear Logs
sudo journalctl --vacuum-time=1s

# Remove unique machine ID (forces regeneration on new instances)
sudo truncate -s 0 /etc/machine-id

# Clear /tmp and /var/tmp
sudo find /tmp -mindepth 1 -delete 2>/dev/null
sudo find /var/tmp -mindepth 1 -delete 2>/dev/null

# Remove current host keys (ensures unique keys per droplet)
sudo rm -f /etc/ssh/ssh_host_*

# Clear Bash History (The absolute final command)
cat /dev/null > ~/.bash_history && history -c && history -w && exit
