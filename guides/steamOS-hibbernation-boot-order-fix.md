# SteamOS Hibernation Boot Order Fix

## Problem Description

After multiple hibernation/resume cycles (typically 4-5), SteamOS shows a boot failure screen offering to select a previous OS version. This occurs because:

1. SteamOS uses a boot counter that increments on each resume
2. The counter isn't properly reset after hibernation
3. The EFI boot order may change during hibernation cycles
4. After reaching a threshold, the system defaults to a "previous" OS version

## Solution Overview

We'll implement a two-part solution:

1. A script that reliably identifies and selects the "current" OS boot entry
2. A systemd service that runs this script after every resume

## Implementation Guide

### 1. Create executable Boot Order Script

#### 1.2. Create script file

```bash
sudo nano /home/deck/.local/bin/ensure-current-boot.sh
```

#### 1.2. Paste script

```bash
#!/bin/bash

# Enhanced SteamOS Boot Order Fixer
# Always selects the CURRENT OS version, not just first in BootOrder

LOG_TAG="SteamOS-Boot-Fix"

# Function to find the current OS boot entry
find_current_boot_entry() {
    # Method 1: Look for SteamOS-specific boot entry
    CURRENT_ENTRY=$(efibootmgr | grep -i 'SteamOS' | grep -i 'current' | head -1 | awk '{print $1}' | sed 's/Boot//' | sed 's/\*//')

    # Method 2: If not found, look for generic Linux loader
    if [ -z "$CURRENT_ENTRY" ]; then
        CURRENT_ENTRY=$(efibootmgr | grep -i 'Linux' | grep -i 'loader' | head -1 | awk '{print $1}' | sed 's/Boot//' | sed 's/\*//')
    fi

    # Method 3: Fallback to first boot entry if still not found
    if [ -z "$CURRENT_ENTRY" ]; then
        CURRENT_ENTRY=$(efibootmgr | grep 'BootOrder' | awk '{print $2}' | cut -d, -f1)
        logger -t $LOG_TAG "Warning: Using first boot entry as fallback: $CURRENT_ENTRY"
    fi

    echo "$CURRENT_ENTRY"
}

# Main execution
CURRENT_BOOT=$(find_current_boot_entry)

if [ -n "$CURRENT_BOOT" ]; then
    # Set next boot to current OS
    efibootmgr -n "$CURRENT_BOOT"

    # Reset boot counter if possible (SteamOS specific)
    if [ -f /usr/share/steamos-bootmenu/bootcounter ]; then
        echo 0 > /usr/share/steamos-bootmenu/bootcounter
    fi

    logger -t $LOG_TAG "Set next boot to current OS: Boot$CURRENT_BOOT"
else
    logger -t $LOG_TAG "Error: Could not identify current boot entry!"
    exit 1
fi

exit 0
```

#### 1.3. Make it executable

```bash
sudo chmod +x /home/deck/.local/bin/ensure-current-boot.sh
```

### 2. Create the Systemd Service Unit

#### 2.1. Create a service file

```bash
sudo nano /etc/systemd/system/reset-boot-order.service
```

#### 2.2. Paste service definition

```ini
[Unit]
Description=SteamOS Boot Order Fix Service
After=suspend.target hibernate.target hybrid-sleep.target
Requires=systemd-efi-boot-status.service
ConditionPathExists=/sys/firmware/efi/efivars

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ensure-current-boot.sh
ExecStartPost=/bin/sleep 3  # Allow hardware to initialize
TimeoutSec=30
RestartSec=5
Restart=on-failure

[Install]
WantedBy=suspend.target hibernate.target hybrid-sleep.target
```

#### 2.3. Enable the service

```bash
sudo systemctl enable reset-boot-order.service
```

### 3. Verification Steps

#### 3.1. Check service status:

```bash
systemctl status steamos-bootfix.service
```

#### 3.2. View logs

```bash
journalctl -t SteamOS-Boot-Fix -b
```

#### 3.3. Test manually

```bash
sudo /usr/local/bin/ensure-current-boot.sh
efibootmgr
```

###Troubleshooting
If issues persist:

Check all EFI boot entries:

```bash
efibootmgr -v
```

Verify SteamOS boot counter:

```bash
cat /usr/share/steamos-bootmenu/bootcounter
```

Test EFI boot entry setting:

```bash
sudo efibootmgr -n XXXX  # Replace with your current boot number
```

Check systemd logs:

```bash
journalctl -u steamos-bootfix.service -b --no-pager
```
