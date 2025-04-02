# internetguard

**Title:** Enforcing Internet Access Control with a FIDO Key on macOS (Public Guide)

---

**Overview:**
This guide walks through setting up internet access control on macOS using a physical FIDO2 security key (such as a YubiKey). The system ensures that the machine only maintains Wi-Fi connectivity when the key is physically connected via USB. This can be used for productivity discipline, personal accountability, or as an extra layer of physical access control.

---

**Goals:**
- Disable Wi-Fi when no FIDO2 key is present.
- Automatically enable Wi-Fi only when a physical key is connected.
- Use unspoofable hardware detection (based on USB Vendor ID).
- Automatically run the detection script every 30 seconds using a LaunchAgent.
- Allow rollback or override in case of device loss or failure.

---

**1. File Structure**

Two key components:
- `~/Scripts/fidonetguard.sh` — Shell script that toggles Wi-Fi based on YubiKey presence
- `~/Library/LaunchAgents/com.fido.internetguard.plist` — LaunchAgent to run the script repeatedly

---

**2. Requirements**
- macOS with Terminal access
- FIDO2 USB key (e.g., any YubiKey)
- Admin account access to configure `sudoers`
- A daily user account to run the LaunchAgent

---

**3. Configure Passwordless Sudo**

1. Open Terminal from an admin account:
```bash
sudo visudo
```
2. Add the following line at the bottom (replace `yourusername` with your actual short username):
```bash
yourusername ALL=(ALL) NOPASSWD: /usr/sbin/networksetup
```
3. Save and exit (`Esc`, then `:wq`).

Test from your standard user:
```bash
sudo /usr/sbin/networksetup -getinfo Wi-Fi
```
It should not ask for a password.

---

**4. Create the Internet Guard Script**

1. From the user account:
```bash
mkdir -p ~/Scripts
nano ~/Scripts/fidonetguard.sh
```
2. Paste this code:
```bash
#!/bin/bash

LOGFILE="$HOME/Scripts/fidonetguard.log"
INTERFACE="Wi-Fi"

KEY_PRESENT=$(ioreg -p IOUSB -l | grep -Ei '("USB Vendor Name" = "Yubico"|"kUSBVendorString" = "Yubico")')

{
  echo "$(date): KEY_PRESENT='$KEY_PRESENT'"
  if [ -z "$KEY_PRESENT" ]; then
    echo "FIDO key NOT found - disabling internet"
    sudo /usr/sbin/networksetup -setnetworkserviceenabled "$INTERFACE" off
  else
    echo "FIDO key found - enabling internet"
    sudo /usr/sbin/networksetup -setnetworkserviceenabled "$INTERFACE" on
  fi
} >> "$LOGFILE" 2>&1
```
3. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).
4. Make it executable:
```bash
chmod +x ~/Scripts/fidonetguard.sh
```

---

**5. Automate the Script with LaunchAgent**

1. Create the LaunchAgent plist:
```bash
nano ~/Library/LaunchAgents/com.fido.internetguard.plist
```
2. Paste the following (edit path as needed):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
   "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.fido.internetguard</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/YOURUSERNAME/Scripts/fidonetguard.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>30</integer>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```
3. Save and set permissions:
```bash
chmod 644 ~/Library/LaunchAgents/com.fido.internetguard.plist
```
4. Load the agent:
```bash
launchctl load ~/Library/LaunchAgents/com.fido.internetguard.plist
```

---

**6. Rollback or Recovery**

- Unload the agent:
```bash
launchctl unload ~/Library/LaunchAgents/com.fido.internetguard.plist
```
- Restore internet manually:
```bash
sudo /usr/sbin/networksetup -setnetworkserviceenabled "Wi-Fi" on
```
- Remove the LaunchAgent and script:
```bash
rm ~/Library/LaunchAgents/com.fido.internetguard.plist
rm ~/Scripts/fidonetguard.sh
```
- Remove sudo permission by editing `sudo visudo` again and deleting the line

---

**7. Tips and Extras**

- Confirm interface name with:
```bash
networksetup -listallnetworkservices
```
- View real-time log:
```bash
tail -f ~/Scripts/fidonetguard.log
```
- Consider adding VPN/Ethernet detection
- Use `osascript` for notifications or `cron` as an alternative to LaunchAgent

---

**Maintainer:** Anonymous Community Contribution
**Version:** Public release, April 2025

