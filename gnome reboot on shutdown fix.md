# Fedora GNOME: Shutdown Triggers a Reboot Instead of Powering Off

## Problem
Clicking **Power Off** in GNOME on Fedora didn't shut the machine down, it rebooted instead, showing my (customized) GRUB screen twice before landing back on the desktop.

## Diagnosis

**1. Tested a plain shutdown from the terminal:**
```bash
systemctl poweroff
```
This powered off cleanly. So it wasn't an ACPI/firmware/BIOS issue, the problem was specific to how GNOME itself handles shutdown.

**2. Reproduced the exact code path the GUI button uses:**
```bash
gnome-session-quit --power-off --no-prompt
```
This also rebooted, confirming the bug was in `gnome-session`.

**3. Checked the previous boot's journal for what actually happened:**
```bash
journalctl -b -1 -o short-precise | grep -iE "system-update|offline|reached target|trigger"
```
Found the cause:
```
Queued start job for default target system-update.target.
...
dnf5[651]: The system has been modified since the offline transaction was prepared. To reschedule, run: dnf5daemon-server
dnf5-offline-transaction.service: Main process exited, code=exited, status=1/FAILURE
Reached target reboot.target - System Reboot.
```

**4. Confirmed directly with dnf5:**
```bash
sudo dnf5 offline status
```
```
The system has been modified since the offline transaction was prepared.
The offline transaction ... is no longer valid.
To clean up, run `dnf5 offline clean`.
```

## Cause
At some point I updated the system another way (e.g. `dnf upgrade` from a terminal), which made that staged offline transaction stale/invalid.

**Result: two reboots → two GRUB screens → no shutdown.**

## Fix
```bash
# clear the stale staged transaction
sudo dnf5 offline clean

# bring the system fully up to date so nothing is left staged/mismatched
sudo dnf upgrade
```



## Result
After clearing the stale transaction, Power Off does a normal, single shutdown, no reboot and no double GRUB screen.
