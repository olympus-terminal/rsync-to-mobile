# Rsync to Mobile (Termux)

Sync files from your PC to your Android phone using rsync over SSH with Termux.

## Prerequisites

- Android phone with [Termux](https://f-droid.org/en/packages/com.termux/) installed (use F-Droid, not Play Store)
- PC and phone on the same WiFi network
- rsync installed on your PC

## Setup on Android (Termux)

### 1. Install required packages

```bash
pkg update && pkg upgrade
pkg install openssh rsync
```

### 2. Grant storage access

```bash
termux-setup-storage
```

This creates `~/storage/` with symlinks to your device storage.

### 3. Set a password

```bash
passwd
```

### 4. Start the SSH server

```bash
sshd
```

The SSH server runs on port **8022** (not the standard 22).

### 5. Find your phone's IP address

Due to Android permissions, `ip addr` may not work. Try these alternatives:

```bash
# Option 1: Check routing table
cat /proc/net/route
```

```bash
# Option 2: Install iproute2
pkg install iproute2
ip route
```

**Easiest method:** Go to Android **Settings → WiFi → tap your connected network** and look for "IP address".

## Sync Files from PC

### Basic rsync command

```bash
rsync -avz -e 'ssh -p 8022' /path/to/files/ user@PHONE_IP:/storage/emulated/0/Download/
```

### Examples

Sync MP3 files to Music folder:
```bash
rsync -avz -e 'ssh -p 8022' *.mp3 u0_a297@192.168.1.100:/storage/emulated/0/Music/
```

Sync a folder:
```bash
rsync -avz -e 'ssh -p 8022' ~/Documents/books/ u0_a297@192.168.1.100:/storage/emulated/0/Documents/books/
```

Sync with progress display:
```bash
rsync -avz --progress -e 'ssh -p 8022' ./photos/ u0_a297@192.168.1.100:/storage/emulated/0/DCIM/
```

### Rsync flags explained

| Flag | Description |
|------|-------------|
| `-a` | Archive mode (preserves permissions, timestamps, etc.) |
| `-v` | Verbose output |
| `-z` | Compress data during transfer |
| `-e 'ssh -p 8022'` | Use SSH on port 8022 |
| `--progress` | Show transfer progress |
| `--delete` | Delete files on destination that don't exist on source |

## Optional: SSH Key Authentication (No Password)

### On your PC

```bash
# Generate key if you don't have one
ssh-keygen -t ed25519

# Copy to phone
ssh-copy-id -p 8022 u0_a297@192.168.1.100
```

### Manual key setup

If `ssh-copy-id` doesn't work:

```bash
# On PC - display your public key
cat ~/.ssh/id_ed25519.pub
```

```bash
# On Termux - add the key
mkdir -p ~/.ssh
echo "YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

## Reverse Sync (Phone to PC)

Pull files from your phone to PC:

```bash
rsync -avz -e 'ssh -p 8022' u0_a297@192.168.1.100:/storage/emulated/0/DCIM/Camera/ ~/phone-photos/
```

## Troubleshooting

### "Connection refused"
- Make sure `sshd` is running in Termux
- Check that both devices are on the same network

### "Broken pipe" / Connection drops mid-transfer
This usually happens when your phone screen turns off and Android suspends Termux.

**Solution 1: Keep the connection alive**
```bash
rsync -Pvrt -e 'ssh -p 8022 -o ServerAliveInterval=60' *.mp3 user@PHONE_IP:~/
```

**Solution 2: Acquire a wake lock in Termux**
```bash
termux-wake-lock
```
This prevents Termux from sleeping when the screen is off.

**Solution 3: Use --partial to resume interrupted transfers**
```bash
rsync -Pvrt --partial -e 'ssh -p 8022 -o ServerAliveInterval=60' files/ user@PHONE_IP:~/
```

**Solution 4: Disable battery optimization**
Go to Android **Settings → Apps → Termux → Battery → Unrestricted**

### "Operation not permitted" / mkstemp failed
Android's scoped storage blocks direct writes to `/storage/emulated/0/`. Use Termux's home directory instead:

```bash
# Use ~/ instead of /storage/emulated/0/
rsync -avz -e 'ssh -p 8022' *.mp3 user@PHONE_IP:~/
```

Then move files on the phone:
```bash
mv ~/*.mp3 ~/storage/music/
```

### "Permission denied"
- Verify you ran `termux-setup-storage`
- Check the password is correct
- Ensure the destination path exists

### "Cannot bind netlink socket"
- Normal on newer Android versions
- Use alternative methods to find IP (see above)

### Find your Termux username
```bash
whoami
```

Usually something like `u0_a297`.

## Storage Paths in Termux

| Path | Description |
|------|-------------|
| `~/storage/shared/` | Internal storage root |
| `~/storage/dcim/` | Camera photos |
| `~/storage/downloads/` | Downloads folder |
| `~/storage/music/` | Music folder |
| `/storage/emulated/0/` | Alternative path to internal storage |

## Create a Sync Script

Save as `sync-to-phone.sh`:

```bash
#!/bin/bash
PHONE_IP="192.168.1.100"  # Replace with your phone's IP
PHONE_USER="u0_a297"       # Replace with your Termux username
PHONE_PORT="8022"

rsync -Pvrt --partial \
    -e "ssh -p $PHONE_PORT -o ServerAliveInterval=60" \
    "$1" \
    "$PHONE_USER@$PHONE_IP:~/"
```

Usage:
```bash
chmod +x sync-to-phone.sh
./sync-to-phone.sh myfile.pdf
```

## License

MIT
