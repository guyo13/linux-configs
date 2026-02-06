# **Telegram SSH Login Notifications Guide**

**System:** Fedora 43 / RHEL-based

**Mechanism:** PAM (Pluggable Authentication Modules)

**Alert Type:** Instant Mobile Push

This guide documents the setup of an automated "tripwire" that sends a Telegram notification the moment a successful SSH session is opened.

## **1\. Prerequisites**

1. **Telegram Bot**: Created via @BotFather.  
2. **API Token**: Provided by BotFather after creation.  
3. **Chat ID**: Your unique Telegram user ID (obtained via @userinfobot).

## **2\. The Alert Script**

Create the script at /usr/local/bin/ssh\_alert.sh and ensure you replace the TOKEN and CHAT\_ID variables.

\#\!/bin/bash

\# Configuration  
TOKEN="YOUR\_TELEGRAM\_BOT\_TOKEN"  
CHAT\_ID="YOUR\_CHAT\_ID"  
HOSTNAME=$(hostname)

\# Only trigger on 'open\_session'  
if \[ "$PAM\_TYPE" \!= "open\_session" \]; then  
    exit 0  
fi

\# Gather metadata  
LOGIN\_IP="$PAM\_RHOST"  
LOGIN\_USER="$PAM\_USER"  
LOGIN\_DATE=$(date "+%Y-%m-%d %H:%M:%S")

\# Build the message  
MESSAGE="🚀 \*SSH Login Detected\!\*  
\-----------------------  
👤 \*User:\* $LOGIN\_USER  
🖥️ \*Host:\* $HOSTNAME  
🌐 \*IP:\* $LOGIN\_IP  
⏰ \*Time:\* $LOGIN\_DATE"

\# Send via Curl  
curl \-s \-X POST "\[https://api.telegram.org/bot$TOKEN/sendMessage\](https://api.telegram.org/bot$TOKEN/sendMessage)" \\  
    \-d chat\_id="$CHAT\_ID" \\  
    \-d text="$MESSAGE" \\  
    \-d parse\_mode="Markdown" \> /dev/null 2\>&1

exit 0

## **3\. Installation & Permissions**

### **Step A: Make the script executable**

sudo chmod \+x /usr/local/bin/ssh\_alert.sh

### **Step B: Hook into PAM**

PAM (Pluggable Authentication Modules) handles system entries. By adding the script to the SSH stack, it triggers before the user even sees a shell prompt.

1. Open the SSH PAM configuration:  
   sudo nano /etc/pam.d/sshd  
2. Add the following line to the **very bottom** of the file:  
   session optional pam\_exec.so /usr/local/bin/ssh\_alert.sh

## **4\. Why use PAM over .bashrc?**

* **Security:** Scripts in .bashrc or .profile can be bypassed if an attacker uses ssh user@ip /bin/sh or SFTP. PAM triggers regardless of the shell or command being executed.  
* **Invisible:** It does not appear in the user's shell history.  
* **Reliability:** It gathers environment variables ($PAM\_RHOST, $PAM\_USER) directly from the authentication stack.

## **5\. Troubleshooting**

If you don't receive a notification:

1. **Manual Test:** Run bash /usr/local/bin/ssh\_alert.sh (Note: It will exit immediately unless you manually export PAM\_TYPE="open\_session" first).  
2. **SELinux:** On Fedora, if the script is blocked, run:  
   sudo restorecon \-v /usr/local/bin/ssh\_alert.sh  
3. **Logs:** Check for errors in the system journal:  
   sudo journalctl \-t pam\_exec
