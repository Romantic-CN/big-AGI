# Guide: Transfer Files Between VM/Host and Serial Console Using rz/sz (ZMODEM)

This guide explains how to send a file from a Linux VM (or WSL) host to a target device connected over a serial link (e.g. `/dev/ttyACM0` or `/dev/ttyUSB0`) using the ZMODEM protocol (`sz`/`rz`). It covers fixing the common "permission denied" error, step-by-step commands for both sides, WSL-specific notes, and cleanup afterwards.

---

## Prerequisites

| Side | Requirement |
|------|-------------|
| Host (VM / WSL) | `lrzsz` package installed (`sz` command available) |
| Target device | `lrzsz` package installed (`rz` command available), root or sufficient shell access |
| Both | A working serial link between host and target (e.g. USB-to-serial cable, USB ACM device) |

Install `lrzsz` on the host if it is missing:

```bash
# Debian / Ubuntu / WSL Ubuntu
sudo apt update && sudo apt install -y lrzsz

# RHEL / CentOS / Fedora
sudo dnf install -y lrzsz
```

---

## Step 1 - Identify the Serial Port on the Host

```bash
# List available serial/USB devices
ls -l /dev/tty{ACM,USB}*
# or
dmesg | tail -20    # look for "ttyACM" or "ttyUSB" after plugging in the device
```

A typical device path looks like `/dev/ttyACM0` or `/dev/ttyUSB0`. Note the exact name - you will need it in the commands below.

---

## Step 2 - Fix "Permission Denied" on /dev/ttyACM0

The error `bash: /dev/ttyACM0: 权限不够` (permission denied) means your current user does not have read/write access to the serial device.

### Check current ownership and permissions

```bash
ls -l /dev/ttyACM0
# Example output:
# crw-rw---- 1 root dialout 166, 0 Apr 19 10:00 /dev/ttyACM0
```

The device is typically owned by `root` and belongs to the `dialout` group (sometimes `uucp` on older systems).

### Option A (recommended) - Add your user to the `dialout` group

This is the permanent solution that does not require `sudo` for every command:

```bash
sudo usermod -aG dialout "$USER"
```

**Important:** You must log out and log back in (or start a new login shell) for the group membership to take effect:

```bash
# Verify the group is active in the new session
groups | grep dialout
```

If you need the change to take effect immediately without logging out, start a new shell with:

```bash
newgrp dialout
```

### Option B (temporary) - Change permissions for this session only

Use this only if you cannot or do not want to modify group membership:

```bash
sudo chmod a+rw /dev/ttyACM0
```

Remember to revert this after the transfer (see Step 6).

### Option C - Run the transfer command with sudo

If neither option above is practical, prefix the `sz` command with `sudo`:

```bash
sudo sz -b <file> > /dev/ttyACM0 < /dev/ttyACM0
```

---

## Step 3 - Prepare the Target Device (Start rz First)

Connect to the target device through your serial terminal (minicom, screen, picocom, etc.). At the target's shell prompt run:

```bash
# On the target device (root shell)
rz
```

The target is now waiting to receive a file over ZMODEM. **Do not press any other keys on the target until the transfer is complete.**

---

## Step 4 - Send the File from the Host

Open a **separate terminal** on the host (while the target is waiting in `rz`). Run `sz` and redirect both stdin and stdout to the serial device:

```bash
# Replace /dev/ttyACM0 with your actual device path
# Replace ~/path/to/your-file.bin with the actual path to the file you want to send
# Example: ~/linux/ATK-DLMP257B/alientek_linux/linux-6.6.48-v1.2/linux/build_image/stm32mp257d-atk-ddr-2GB.dtb
sz -b ~/path/to/your-file.bin \
   > /dev/ttyACM0 < /dev/ttyACM0
```

Flag reference:

| Flag | Meaning |
|------|---------|
| `-b` | Binary transfer mode (required for non-text files such as `.dtb`, images, archives) |
| `-e` | Escape control characters (add this if the link uses XON/XOFF flow control and the transfer stalls) |
| `-v` | Verbose: print progress to stderr |

---

## Step 5 - Confirm the Transfer

On the **target device**, after `rz` finishes:

```bash
# List the received file in the current directory (replace your-file.bin with the actual filename)
ls -lh your-file.bin

# Verify file integrity with md5sum (run the same command on the host and compare)
md5sum your-file.bin
```

On the **host**, compute the checksum of the original file:

```bash
# Replace ~/path/to/your-file.bin with your actual file path
# Example: ~/linux/ATK-DLMP257B/alientek_linux/linux-6.6.48-v1.2/linux/build_image/stm32mp257d-atk-ddr-2GB.dtb
md5sum ~/path/to/your-file.bin
```

Both checksums must match. If they differ, repeat the transfer.

---

## Step 6 - Revert Temporary Permission Changes

If you used Option B (temporary `chmod`) in Step 2, restore the original permissions after the transfer:

```bash
sudo chmod o-rw /dev/ttyACM0
# Verify
ls -l /dev/ttyACM0
```

---

## WSL-Specific Notes

Windows Subsystem for Linux does not expose COM ports as `/dev/ttyACMx` automatically. You need to attach the COM port to WSL using `usbipd-win` or use a serial terminal on the Windows side.

### Attaching a USB serial device to WSL 2 with usbipd-win

1. Install [usbipd-win](https://github.com/dorssel/usbipd-win) on Windows (requires Windows 11 or Windows 10 22H2+).

2. In an **elevated** Windows PowerShell:

   ```powershell
   # List USB devices and note the BUSID of your serial adapter
   usbipd list

   # Attach the device to WSL (replace 1-3 with your BUSID)
   usbipd attach --wsl --busid 1-3
   ```

3. Inside WSL, the device now appears as `/dev/ttyACM0` (or `/dev/ttyUSB0`). Verify with:

   ```bash
   ls /dev/tty{ACM,USB}*
   ```

4. Follow Steps 2-5 above normally.

5. When finished, detach the device from WSL:

   ```powershell
   usbipd detach --busid 1-3
   ```

### Alternative: Use a Windows serial terminal for the ZMODEM transfer

If attaching the USB device to WSL is not practical, you can use a Windows-native terminal emulator that supports ZMODEM (e.g. TeraTerm, SecureCRT, or PuTTY with a ZMODEM plugin). The concepts are the same: start `rz` on the target first, then initiate the ZMODEM send from the Windows terminal's menu or command.

---

## Common Failure Points and Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `bash: /dev/ttyACM0: 权限不够` | User not in `dialout` group | Step 2 - add user to group or use `sudo` |
| `sz: error ... cannot open /dev/ttyACM0` | Wrong device path | Check `ls /dev/tty*` or `dmesg` |
| Transfer starts but stalls immediately | `rz` not started on target before `sz` on host | Always start `rz` on the target **first** |
| Transfer corrupted (md5 mismatch) | Missing `-b` flag (text mode mangled binary) | Always use `-b` for binary files |
| `sz: command not found` on host | `lrzsz` not installed | `sudo apt install lrzsz` |
| `rz: command not found` on target | `lrzsz` not installed on target | `apt install lrzsz` or `opkg install lrzsz` |
| Very slow transfer | Default serial baud rate is low | Set a higher baud rate on both sides (e.g. 115200); both sides must match |
| Transfer hangs with XON/XOFF links | Software flow control eating ZMODEM control bytes | Add `-e` flag to `sz` |

---

## Quick Reference (Copy-Paste Commands)

### Target device

```bash
# 1. Start receiver (do this FIRST, before running sz on host)
rz
```

### Host / VM

```bash
# 2a. Fix permissions permanently (log out and back in after this)
sudo usermod -aG dialout "$USER"

# 2b. Or fix permissions temporarily for this session only
sudo chmod a+rw /dev/ttyACM0

# 3. Send the file (binary mode, replace device path and file path as needed)
# Example file: ~/linux/ATK-DLMP257B/alientek_linux/linux-6.6.48-v1.2/linux/build_image/stm32mp257d-atk-ddr-2GB.dtb
sz -b ~/path/to/your-file.bin \
   > /dev/ttyACM0 < /dev/ttyACM0

# 4. Compute checksum on host to compare with target
md5sum ~/path/to/your-file.bin

# 5. Revert temporary permission change (if used option 2b)
sudo chmod o-rw /dev/ttyACM0
```

### Target device - after transfer

```bash
# Verify file arrived and check size (replace your-file.bin with the actual filename)
ls -lh your-file.bin

# Compare checksum with host output
md5sum your-file.bin
```

---

## Security Considerations

- **Prefer group membership** (`dialout`) over world-writable permissions. Opening the serial device to all users (`chmod a+rw`) is a broader permission grant than necessary.
- **Revert temporary `chmod` changes** as soon as the transfer is complete (Step 6).
- **Avoid running `sz`/`rz` as root** unless strictly necessary. Running as a member of the `dialout` group is sufficient and safer.
- **Do not leave the serial device writable** when it is not in use, as other processes or users on the same system could send unexpected data to the target.
