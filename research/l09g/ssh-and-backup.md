# L09G — Persistent SSH and partition backup

This procedure replaces `infos-and-ssh.md` with a single, ordered workflow: first obtain a
persistent root shell **without touching `boot`/`recovery`**, then back up every NAND partition.

> **Warning**
> `boot` and `recovery` are signed and their payloads are encrypted. Do not modify them casually.
> This guide writes only to the `data` partition (UBI, `mtd7`) and its overlay. Always keep a backup
> and working UART/U-Boot access.

---

## 0. Prerequisites

- Xiaomi L09G with accessible UART (`J5`: `RX`, `GND`, `TX`; 115200 8N1).
- **3.3 V** USB-TTL adapter (CP2102, PL2303, FT232 3.3 V…).
- Soldering iron and a 2.54 mm header or thin wires, if `J5` is not populated.
- A Linux/macOS computer with `tio`, `picocom` or `screen`.
- Optional: a FAT32 USB stick and a soldered USB host port, only for the U-Boot dump method.

Hardware details: `hardware-uart-usb.en.md`.

---

## 1. Open the U-Boot console

Power the speaker with its original supply while the serial console is open. Interrupt the boot by
pressing Enter repeatedly until the prompt appears:

```text
axg_s420_v1_gva#
```

With `tio`, to avoid losing characters when pasting long lines:

```bash
tio -b 115200 -o 5 /dev/ttyUSB0
```

If you lack permissions on the device:

```bash
sudo setfacl -m u:$USER:rw /dev/ttyUSB0
```

Verify that you are in U-Boot:

```text
printenv bootargs
mtdparts
```

---

## 2. Enter a root shell without touching boot/recovery

Paste **one line** at the U-Boot prompt. It sets bootargs in RAM only: do not use `saveenv`.

```text
setenv bootargs 'rootfstype=ramfs rdinit=/sbin/busybox console=ttyS0,115200 no_console_suspend earlycon=aml_uart,0xff803000 -- ash -c "/sbin/busybox mkdir -p /proc /sys /dev /system /bin; /sbin/busybox mount -t proc proc /proc; /sbin/busybox mount -t sysfs sysfs /sys; /sbin/busybox mount -t devtmpfs devtmpfs /dev; /sbin/busybox mount -t squashfs -o ro /dev/mtdblock4 /system; /sbin/busybox --install -s /bin; export LD_LIBRARY_PATH=/system/lib; exec /bin/ash"'
```

Check that the variable was written completely:

```text
printenv bootargs
```

It must end with:

```text
exec /bin/ash"
```

Boot:

```text
run storeboot
```

You get a root shell (`#`). If something goes wrong, remove power and restart: the bootargs were not
saved.

> The shell runs in ramfs; it is not the full Android system. Some commands (for example
> `wpa_supplicant`) do not work there and are only used during a normal boot.

---

## 3. Mount the persistent partition

`data` is UBI on `mtd7`. Mount it as `/pers`:

```sh
/sbin/ubiattach /dev/ubi_ctrl -m 7
mkdir -p /pers
mount -t ubifs /dev/ubi0_0 /pers
ls -la /pers
```

You should see:

```text
/pers/.data/              -> /data in normal operation
/pers/.system.overlay/    -> writable overlay over /system (squashfs mtd4)
```

According to `infos-and-ssh.md`, in normal operation `/pers` corresponds to `/cache` and `/data` is
`/cache/.data`. The exact mapping can vary by build: verify with `mount` and `ls` before writing.

---

## 4. Passwordless root

Create `/etc/passwd` in the overlay:

```sh
mkdir -p /pers/.system.overlay/etc /pers/.data/root
echo 'root::0:0:root:/data/root:/bin/sh' > /pers/.system.overlay/etc/passwd
chmod 0644 /pers/.system.overlay/etc/passwd
```

The empty password field means root has no password. `sshd` must run with `-B` to accept it.

---

## 5. SSH host key: use ecdsa, not ed25519

The firmware does not support ed25519. Generate an **ecdsa** key:

```sh
mkdir -p /pers/.data/ssh
/system/usr/bin/sshd_keys -t ecdsa -f /pers/.data/ssh/dropbear_ecdsa_host_key
```

---

## 6. SSH start script + update blocker

Create `/pers/.data/ssh/start_ssh.sh`:

```sh
cat > /pers/.data/ssh/start_ssh.sh <<'EOS'
#!/bin/sh
/system/usr/bin/sshd -R -r /data/ssh/dropbear_ecdsa_host_key -B -p 2222 -E >> /data/ssh/dropbear.log 2>&1
( while : ; do /system/bin/killall update_engine 2>/dev/null ; /system/bin/sleep 5 ; done ) &
EOS
chmod 0755 /pers/.data/ssh/start_ssh.sh
```

Notes:

- `-R` generates missing host keys; `-r` uses our ecdsa key;
- `-B` allows an empty password;
- `-p 2222` is the SSH port;
- the loop keeps `update_engine` dead, otherwise `process_manager` respawns it.

The firmware shell has no `[`/`test`: avoid `[ ... ]` in scripts and use `case`/`killall`.

---

## 7. Hook into normal boot

Normal boot runs `/usr/bin/insert_modules.sh` as root (it also loads the WLAN driver). Copy it into
the overlay and append the launcher, **without removing the original content**:

```sh
mkdir -p /pers/.system.overlay/usr/bin
cp /system/usr/bin/insert_modules.sh /pers/.system.overlay/usr/bin/insert_modules.sh
printf '\n# --- persistent ssh + no-ota ---\nsh /data/ssh/start_ssh.sh &\n' >> /pers/.system.overlay/usr/bin/insert_modules.sh
chmod 0755 /pers/.system.overlay/usr/bin/insert_modules.sh
```

`insert_modules.sh` is the hook used by `infos-and-ssh.md`. Some builds may also use
`set_usb_serialnumber.sh`; check which one the system actually calls and use that.

---

## 8. Wi-Fi without the app

Pre-write the Wi-Fi configuration in `data`:

```sh
mkdir -p /pers/.data/wifi
cat > /pers/.data/wifi/wpa_supplicant.conf <<'EOF'
ctrl_interface=/data/wifi
update_config=1
country=IT
network={
	ssid="YOUR_WIFI"
	psk="YOUR_PASSWORD"
}
EOF
chmod 0644 /pers/.data/wifi/wpa_supplicant.conf
```

The firmware’s `wpa_supplicant` is an Android build and exits with error 255 in the ramfs shell: do
not start it there. On normal boot the system will use the file and connect.

---

## 9. Verify, sync, reboot

```sh
cat /pers/.system.overlay/etc/passwd
cat /pers/.data/ssh/start_ssh.sh
tail -5 /pers/.system.overlay/usr/bin/insert_modules.sh
grep ssid /pers/.data/wifi/wpa_supplicant.conf
sync
reboot
```

After reboot, find the IP from the router (hostname similar to Chromecast) or scan port 2222:

```bash
for i in $(seq 1 254); do
  nc -z -w1 192.168.1.$i 2222 2>/dev/null && echo "2222 open: 192.168.1.$i"
done
```

Expected banner:

```bash
printf '' | nc -w2 <IP> 2222
# SSH-2.0-dropbear_...
```

Login:

```bash
ssh -p 2222 root@<IP>
# empty password: press Enter
```

For non-interactive scripts:

```bash
printf '#!/bin/sh\necho ""\n' > askpass.sh && chmod +x askpass.sh
SSH_ASKPASS=./askpass.sh SSH_ASKPASS_REQUIRE=force setsid -w \
  ssh -p 2222 -o PubkeyAuthentication=no -o PreferredAuthentications=password \
  -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  root@<IP> 'id; uname -a'
```

Reserve a static IP in the router so the SSH address stays stable.

---

## 10. Back up every partition

### 10.1 Recommended method: SSH to the host

With root SSH active, read the MTD block devices and stream them directly to the computer. **Do not
store the dumps in `/data`**: only ~110 MB are free and the partitions total ~250 MB.

```bash
HOST=root@192.168.1.50   # <-- replace with the speaker IP
PORT=2222
OUT=backup-l09g-$(date +%Y%m%d-%H%M)
mkdir -p "$OUT"

for n in 0 1 2 3 4 5 6 7; do
  echo "Dump mtd$n..."
  ssh -T -p "$PORT" "$HOST" "cat /dev/mtdblock$n" > "$OUT/mtd$n.bin"
  shasum -a 256 "$OUT/mtd$n.bin" >> "$OUT/SHA256SUMS"
done

ssh -T -p "$PORT" "$HOST" "cat /proc/mtd" > "$OUT/proc-mtd.txt"
ssh -T -p "$PORT" "$HOST" "cat /proc/cmdline" > "$OUT/proc-cmdline.txt"
```

On Linux use `sha256sum` instead of `shasum -a 256`.

If `cat` fails, use `dd`:

```bash
ssh -T -p "$PORT" "$HOST" "dd if=/dev/mtdblock$n bs=1M 2>/dev/null" > "$OUT/mtd$n.bin"
```

If the block devices do not exist, try the char devices:

```bash
ssh -T -p "$PORT" "$HOST" "cat /dev/mtd$n" > "$OUT/mtd$n.bin"
```

### 10.2 Partition table

| mtd | device | name | size |
|---|---|---|---:|
| 0 | `/dev/mtdblock0` | `bootloader` | 2 MiB |
| 1 | `/dev/mtdblock1` | `tpl` | 8 MiB |
| 2 | `/dev/mtdblock2` | `recovery` | 10 MiB |
| 3 | `/dev/mtdblock3` | `boot` | 12 MiB |
| 4 | `/dev/mtdblock4` | `system` | 16 MiB |
| 5 | `/dev/mtdblock5` | `chrome` | ~68.1 MiB |
| 6 | `/dev/mtdblock6` | `factory` | 4 MiB |
| 7 | `/dev/mtdblock7` | `data` | ~129.9 MiB |

Total ~250 MiB; the NAND is 256 MiB, with an unmapped `0x200000`–`0x800000` gap.

### 10.3 `data` consistency

`mtd7` is UBI/UBIFS and is normally mounted. For a consistent dump:

- stop writers and run `sync`, or
- dump `mtd7` from U-Boot with the system powered off.

`boot`, `recovery`, `system`, `chrome`, `factory`, `bootloader`, `tpl` do not have this problem
(they are read-only or not mounted).

### 10.4 Verification

```bash
ls -lh "$OUT"
file "$OUT"/mtd*.bin
binwalk "$OUT"/mtd2.bin "$OUT"/mtd3.bin
unsquashfs -s "$OUT"/mtd4.bin
unsquashfs -s "$OUT"/mtd5.bin
```

Expected:

- `mtd2`/`mtd3`: Android bootimg;
- `mtd4`/`mtd5`: squashfs;
- `mtd6`/`mtd7`: UBI/UBIFS;
- `mtd0`/`mtd1`: raw bootloader/tpl data.

---

## 11. U-Boot backup (fallback, no SSH)

If SSH is unavailable, U-Boot can read NAND and write to a FAT32 USB stick. The USB host port must
be soldered (see `hardware-uart-usb.en.md`).

```text
usb start
fatls usb 0
setenv addr 0x02000000
```

For each partition:

```text
nand read ${addr} <offset> <size>
fatwrite usb 0:1 <filename> ${addr} <size>
```

Example for `system`:

```text
nand read ${addr} 0x02600000 0x01000000
fatwrite usb 0:1 mtd4_system.bin ${addr} 0x01000000
```

Offsets/sizes:

| Partition | Offset | Size |
|---|---:|---:|
| `bootloader` | `0x00000000` | `0x00200000` |
| `tpl` | `0x00800000` | `0x00800000` |
| `recovery` | `0x01000000` | `0x00A00000` |
| `boot` | `0x01A00000` | `0x00C00000` |
| `system` | `0x02600000` | `0x01000000` |
| `chrome` | `0x03600000` | `0x04420000` |
| `factory` | `0x07A20000` | `0x00400000` |
| `data` | `0x07E20000` | `0x081E0000` |

Before doing everything, test a small chunk and verify the file magic. If `fatwrite` is not enabled:

```text
help fatwrite
```

Alternatively use `nand read.raw` + `fatwrite` in 16–32 MiB chunks. Avoid page-by-page `md.b`: it is
the ~23-hour method described in the thread.

### Full-chip dump

`mtd0`–`mtd7` do not cover the `0x200000`–`0x800000` gap and do not include OOB data. For a forensic
clone of the whole NAND:

- contiguous dump from `0x00000000` to `0x10000000` in 16–32 MiB chunks from U-Boot, or
- `nanddump` with OOB/bad-block handling on each `/dev/mtdN` from Linux.

For a partition restore backup, the `mtd0`–`mtd7` files are enough.

---

## 12. Security and management

- Do not expose port 2222 to the Internet.
- If the device is reachable from untrusted networks, add an SSH key and disable the empty password;
  passwordless root is convenient but dangerous.
- Block WAN at the router: it is the most reliable defence against automatic updates.
- Keep two copies of the dumps and record SHA-256.
- Do not flash `boot`/`recovery` without a backup, UART and recoverable U-Boot.

---

## 13. Troubleshooting

| Problem | Fix |
|---|---|
| `sshd_keys -t ed25519` fails | use `ecdsa` |
| `[: not found` in scripts | the shell has no `[`/`test`; use `case`/`killall` |
| `wpa_supplicant` exits 255 in ramfs | it is the Android build; keep the config and reboot normally |
| Serial loses characters | use `tio -o 5`, short lines, avoid long heredocs |
| `update_engine` comes back | `killall` loop + WAN block |
| Login denied with empty password | `sshd` must run with `-B`; `/etc/passwd` must have an empty password field |
| `/dev/mtdblock*` missing | create the nodes or use `/dev/mtdN`; check `cat /proc/mtd` |
| Dump in `/data` fails for space | stream to the host or use USB |
