# Guide: Transfer File Between VM Host and Serial Console Using rz/sz (ZMODEM)

> **Problem solved:** `bash: /dev/ttyACM0: Permission denied` when running `sz ... > /dev/ttyACM0 < /dev/ttyACM0`

This guide walks you through sending a file from a Linux/WSL VM host to an embedded target device
over a serial connection (ZMODEM protocol), and explains every permission and USB-passthrough
pitfall you may encounter.

---

## Table of Contents

1. [Quick-Start - Minimal Working Commands](#1-quick-start---minimal-working-commands)
2. [Fix: Permission Denied on /dev/ttyACM0](#2-fix-permission-denied-on-devttyacm0)
3. [Check What Is Using the Port](#3-check-what-is-using-the-port)
4. [WSL and VM USB Passthrough Notes](#4-wsl-and-vm-usb-passthrough-notes)
5. [Debugging Commands](#5-debugging-commands)
6. [Persistent Permissions - udev Rule](#6-persistent-permissions---udev-rule)
7. [Alternative Transfer Methods](#7-alternative-transfer-methods)
8. [Security Notes](#8-security-notes)

---

## 1. Quick-Start - Minimal Working Commands

Replace the path and device name to match your setup.

```
FILE=~/linux/ATK-DLMP257B/alientek_linux/linux-6.6.48-v1.2/linux/build_image/stm32mp257d-atk-ddr-2GB.dtb
DEVICE=/dev/ttyACM0
```

### Step 1 - On the target device (serial console)

In your serial terminal emulator (minicom, PuTTY, screen, etc.) type:

```sh
rz -b
```

The target is now waiting to receive a binary file.

### Step 2 - On the host (immediately after Step 1)

**Option A - preferred:** add your user to `dialout` first (one-time, requires re-login; see
Section 2), then run:

```sh
sz -b "$FILE" > "$DEVICE" < "$DEVICE"
```

**Option B - no group change needed:** use `sudo sh -c` to redirect inside an elevated shell:

```sh
sudo sh -c "sz -b '$FILE' > $DEVICE < $DEVICE"
```

**Option C - pipe through `sudo tee`:**

```sh
sz -b "$FILE" | sudo tee "$DEVICE" > /dev/null
```

> Note: Options B and C work immediately without logging out. Option A is the cleanest long-term
> solution.

---

## 2. Fix: Permission Denied on /dev/ttyACM0

### 2a. Inspect device ownership

```sh
ls -l /dev/ttyACM0
```

Typical output:

```
crw-rw---- 1 root dialout 166, 0 Apr 19 07:00 /dev/ttyACM0
```

The device is owned by `root` and group `dialout` with mode `660`. Your regular user does not have
write permission unless they belong to `dialout`.

### 2b. Add your user to the dialout group (recommended fix)

```sh
sudo usermod -aG dialout "$USER"
```

Then **log out and log back in** (or start a new login shell with `newgrp dialout`) for the group
membership to take effect:

```sh
newgrp dialout   # temporary: effective only in this shell session
```

Verify:

```sh
groups           # dialout should appear in the list
```

### 2c. Use sudo sh -c for redirection (no logout required)

Shell redirection (`>`, `<`) is handled by the shell before `sudo` sees the command, so plain
`sudo sz ... > /dev/ttyACM0` does NOT elevate the redirection. Use this pattern instead:

```sh
sudo sh -c "sz -b '$FILE' > $DEVICE < $DEVICE"
```

This runs the entire command string, including redirects, inside a root shell.

---

## 3. Check What Is Using the Port

If `sz` complains about a busy device or the transfer hangs, another process may have the port
open (e.g. minicom, screen, PuTTY, ModemManager).

### Find the PID

```sh
sudo fuser /dev/ttyACM0
```

Example output: `ttyACM0: 12345`

```sh
sudo lsof /dev/ttyACM0
```

### Identify and kill the process

```sh
ps -p 12345 -o pid,comm,args   # replace 12345 with the actual PID
sudo kill 12345
```

### Stop ModemManager (common culprit on desktop Linux)

```sh
sudo systemctl stop ModemManager
```

To prevent it from grabbing serial ports automatically, you can disable it permanently:

```sh
sudo systemctl disable ModemManager
```

---

## 4. WSL and VM USB Passthrough Notes

When your host is Windows running WSL2 or a VM (VirtualBox, VMware), the serial device may not
appear automatically inside the Linux environment.

### 4a. WSL2 - usbipd-win

1. Install **usbipd-win** on Windows (https://github.com/dorssel/usbipd-win).
2. In an **elevated** Windows PowerShell:

   ```powershell
   usbipd list                      # find your USB-serial adapter
   usbipd bind --busid <BUSID>      # allow WSL to claim it (one-time)
   usbipd attach --wsl --busid <BUSID>   # attach to WSL
   ```

3. Inside WSL, verify the device appeared:

   ```sh
   ls /dev/ttyACM* /dev/ttyUSB*
   dmesg | tail -20
   ```

4. Detach when done:

   ```powershell
   usbipd detach --busid <BUSID>
   ```

### 4b. VirtualBox

1. Open **VM Settings > USB > USB 2.0/3.0 Controller**.
2. Add a USB Device Filter for your USB-serial adapter (Vendor/Product ID from `lsusb`).
3. With the VM running, use **Devices > USB** to attach the adapter; it will appear as
   `/dev/ttyACM0` or `/dev/ttyUSB0` inside the VM.

### 4c. VMware

1. Go to **VM > Removable Devices** and connect the USB-serial adapter.
2. Or use **VM > Settings > USB Controller** and set USB compatibility to 3.1.

### 4d. Device name differences

| Adapter type | Typical device name |
|---|---|
| CDC-ACM (most dev boards) | `/dev/ttyACM0` |
| CH340 / CP210x / FTDI | `/dev/ttyUSB0` |
| By stable ID | `/dev/serial/by-id/usb-FTDI_FT232R_USB_UART_A12345-if00-port0` |

Using `/dev/serial/by-id/...` avoids numbering races when multiple adapters are connected.
The exact name is generated from the adapter's USB descriptor strings; run
`ls /dev/serial/by-id/` after plugging in to see the full name for your device.

---

## 5. Debugging Commands

### Check if the device is recognized by the kernel

```sh
dmesg | grep -E 'tty|ACM|USB|cdc'
lsusb
```

### List all serial ports

```sh
ls -l /dev/ttyACM* /dev/ttyUSB* /dev/serial/by-id/
```

### Monitor serial output in real time (without blocking the port for sz)

Stop any existing terminal emulator first, then use:

```sh
sudo cat /dev/ttyACM0        # raw - press Ctrl+C to stop
```

Or configure baud rate first:

```sh
sudo stty -F /dev/ttyACM0 115200 raw
sudo cat /dev/ttyACM0
```

### Full process check

```sh
sudo fuser -v /dev/ttyACM0
sudo lsof /dev/ttyACM0
ps aux | grep -E 'minicom|screen|picocom|ttyACM'
```

---

## 6. Persistent Permissions - udev Rule

Instead of manually running `sudo` or `usermod` each time, create a udev rule that automatically
sets the group and permissions whenever the device is plugged in.

### Find Vendor ID and Product ID

```sh
lsusb | grep -i serial   # e.g. "Bus 001 Device 003: ID 0403:6001 Future Technology ..."
```

Or from dmesg:

```sh
dmesg | grep -i 'idVendor\|idProduct'
```

### Create the rule file

```sh
sudo tee /etc/udev/rules.d/99-usb-serial.rules > /dev/null << 'EOF'
# Set group=dialout and mode=0660 for a specific USB-serial adapter
# Replace ATTRS{idVendor} and ATTRS{idProduct} with your device's IDs
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", \
    GROUP="dialout", MODE="0660"
EOF
```

### Reload rules

```sh
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Unplug and replug the adapter; it should now be accessible by any `dialout` group member without
`sudo`.

---

## 7. Alternative Transfer Methods

### 7a. SCP over network (recommended when network is available)

If the target device has an IP address and an SSH server running:

```sh
scp stm32mp257d-atk-ddr-2GB.dtb root@<TARGET_IP>:/boot/
```

Or in the reverse direction (pull from host to target, run on target):

```sh
scp user@<HOST_IP>:~/linux/.../stm32mp257d-atk-ddr-2GB.dtb /boot/
```

### 7b. Base64 over serial (last resort - slow, no special tools needed)

**On the host**, encode the file and copy the output:

```sh
base64 stm32mp257d-atk-ddr-2GB.dtb | split -b 60 - chunk_
# paste each chunk manually into the serial terminal, or pipe with a delay
```

**On the target**, collect the base64 text into a file and decode:

```sh
# In the serial console on the target:
cat > /tmp/file.b64    # paste the base64 text, then press Ctrl+D
base64 -d /tmp/file.b64 > /boot/stm32mp257d-atk-ddr-2GB.dtb
```

Verify the file arrived intact:

```sh
# On host:
md5sum stm32mp257d-atk-ddr-2GB.dtb

# On target:
md5sum /boot/stm32mp257d-atk-ddr-2GB.dtb
```

### 7c. TFTP (common in embedded development)

On the host install and start a TFTP server, copy the file into the TFTP root, then on the target:

```sh
tftp -g -r stm32mp257d-atk-ddr-2GB.dtb <HOST_IP>
```

---

## 8. Security Notes

### Temporary chmod 666 - use with caution

```sh
sudo chmod 666 /dev/ttyACM0   # grants world read+write
```

This allows any user or process on the system to read from and write to the serial port. It is
acceptable on a single-user workstation for a quick test, but **revert it immediately after**:

```sh
sudo chmod 660 /dev/ttyACM0
```

Or simply unplug and replug the adapter to restore udev-assigned permissions.

### Prefer group membership over chmod 666

Adding your user to `dialout` and using mode `660` gives the same convenience without exposing
the port to all processes. This is the recommended long-term approach.

### sudo with redirection

When using `sudo sh -c '... > $DEVICE'`, make sure the command string does not contain
untrusted input - construct it only from known, validated paths. Use **single quotes** inside
the `sh -c` argument (as shown in Sections 1 and 2c) rather than double-quoting variable
expansions, to avoid shell injection if a path contains special characters.

---

## Summary: Fastest Fix for Permission Denied

```sh
# One-time setup (requires re-login or newgrp):
sudo usermod -aG dialout "$USER"
newgrp dialout   # or log out and back in

# Send the file:
sz -b ~/linux/ATK-DLMP257B/alientek_linux/linux-6.6.48-v1.2/linux/build_image/stm32mp257d-atk-ddr-2GB.dtb \
   > /dev/ttyACM0 < /dev/ttyACM0

# If you cannot re-login right now, use sudo sh -c instead:
sudo sh -c "sz -b ~/linux/ATK-DLMP257B/alientek_linux/linux-6.6.48-v1.2/linux/build_image/stm32mp257d-atk-ddr-2GB.dtb > /dev/ttyACM0 < /dev/ttyACM0"
```

Start `rz -b` on the target **before** running `sz` on the host.
