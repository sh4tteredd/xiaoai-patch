# L09G — SSH from a virgin unit (U-Boot + ramfs + patched Dropbear)

This is a complete first-time procedure for a stock L09G with no root shell, no telnet and no SSH.
It uses only the UART/U-Boot console, the kernel ramfs shell, and the writable `data` partition.
It does **not** modify `boot`, `recovery` or the signed squashfs partitions.

Three approaches are covered:

1. **Preferred:** patch `dropbearmulti` directly from the ramfs shell, then reboot into a normal
   system with SSH already working.
2. **Fallback:** if patching from the ramfs is not possible, set up an unauthenticated telnet shell
   first, reboot, then patch `dropbearmulti` and start SSH from telnet.
3. **Alternative:** do not patch the binary at all; override `ro.product.release.track` at boot by
   shadowing `/system/build.prop` through the overlay (section 5b). This is the method that has been
   verified in practice on this hardware.

The patch is the same in both cases:

```text
file offset 0x9290:  50 b9   (cbnz r0, #0x92a8)
                ->   0a e0   (b    #0x92a8)
```

This skips the `ro.product.release.track == stable-channel` check inside Dropbear's server main.

> **Warning**
> This guide writes only to the UBI `data` partition (`mtd7`) and its overlay. Keep a UART/U-Boot
> recovery path, use the original 12 V supply, and never connect 5 V to the 3.3 V UART header.
> Telnet on port 2323 is an unauthenticated root shell: use it only on a trusted LAN and close it
> after setup.

---

## 1. Prerequisites

- Xiaomi L09G with UART header `J5` (`RX`, `GND`, `TX`).
- **3.3 V** USB-TTL adapter (CP2102, PL2303, FT232 3.3 V, CH340 3.3 V).
- `tio`, `picocom` or `screen` on the computer.
- Original 12 V power supply.
- Wi-Fi SSID and password.
- Optional but recommended: a full NAND backup before changing anything (see
  [`ssh-and-backup.md`](ssh-and-backup.md) section 10).

Wiring:

```text
adapter TX  -> board RX
adapter RX  -> board TX
adapter GND -> board GND
adapter VCC -> do not connect
```

Serial settings:

```text
115200 8N1, no flow control
```

Example:

```bash
tio -b 115200 -o 5 /dev/ttyUSB0
```

`-o 5` adds a per-character delay and prevents dropped characters when pasting the long U-Boot
bootargs line.

---

## 2. Enter U-Boot

Power the speaker with the UART console open and press Enter repeatedly until the prompt appears:

```text
axg_s420_v1_gva#
```

Check the environment:

```text
printenv bootargs
mtdparts
```

Do **not** run `saveenv` at any point. The bootargs below live in RAM only; a power cycle restores
the normal boot.

---

## 3. Enter the ramfs root shell

Paste this as **one line** at the U-Boot prompt:

```text
setenv bootargs 'rootfstype=ramfs rdinit=/sbin/busybox console=ttyS0,115200 no_console_suspend earlycon=aml_uart,0xff803000 -- ash -c "/sbin/busybox mkdir -p /proc /sys /dev /system /bin; /sbin/busybox mount -t proc proc /proc; /sbin/busybox mount -t sysfs sysfs /sys; /sbin/busybox mount -t devtmpfs devtmpfs /dev; /sbin/busybox mount -t squashfs -o ro /dev/mtdblock4 /system; /sbin/busybox --install -s /bin; export LD_LIBRARY_PATH=/system/lib; exec /bin/ash"'
```

Verify it was written completely:

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

You get a root shell (`#`). This is a ramfs, not the full Android system. `/system` is mounted
read-only from `mtd4`.

Set a short alias for the next commands:

```sh
BB=/sbin/busybox
```

---

## 4. Mount the persistent `data` partition

```sh
$BB mkdir -p /pers /tmp
/sbin/ubiattach /dev/ubi_ctrl -m 7 2>/dev/null
$BB mount -t ubifs /dev/ubi0_0 /pers
```

Verify:

```sh
ls -la /pers
```

You should see:

```text
/pers/.data/              -> /data on normal boot
/pers/.system.overlay/    -> writable overlay over /system
```

If the names differ, use `mount` and `ls` to identify the actual paths before writing. On this
firmware `/pers` is `/cache` on normal boot, `/pers/.data` is `/data`, and
`/pers/.system.overlay` is the overlay upperdir for `/system`.

---

## 5. Preferred path — patch Dropbear from the ramfs

### 5.1 Verify the target binary

This guide is written for the 1.52.54 `dropbearmulti`:

```text
MD5    281e5dd72c70a886de8c2c864b6fe954
SHA256 205f848c8bfe8484fd5203af89b38e7e678bc2f3c05acc40ddeae541e560352f
size   182676 bytes
```

Check it from the ramfs:

```sh
$BB md5sum /system/usr/bin/dropbearmulti
$BB dd if=/system/usr/bin/dropbearmulti bs=1 skip=$((0x9290)) count=2 2>/dev/null | $BB od -A n -t x1
```

The second command must print:

```text
 50 b9
```

If the MD5 is different or the bytes are not `50 b9`, **stop**. The firmware is a different build
and the offset may not apply. Locate the check by searching for `disabled on stable-channel` and
the `cbnz` after the `bcmp`, or use the property-overlay method from
[`ssh-disabler.md`](ssh-disabler.md).

### 5.2 Back up the original

The original is always available read-only in `/system` (and in the squashfs), but keep a writable
copy on `data`:

```sh
$BB mkdir -p /pers/.data/ssh
$BB cp /system/usr/bin/dropbearmulti /pers/.data/ssh/dropbearmulti.orig
$BB chmod 0755 /pers/.data/ssh/dropbearmulti.orig
```

### 5.3 Build the patched binary in the overlay upperdir

The ramfs busybox `dd` is sufficient; no `conv=notrunc` is needed. Build the file as
prefix + two patched bytes + suffix:

```sh
$BB mkdir -p /pers/.system.overlay/usr/bin

$BB dd if=/system/usr/bin/dropbearmulti \
  of=/pers/.system.overlay/usr/bin/dropbearmulti \
  bs=1 count=$((0x9290)) 2>/dev/null

$BB printf '\012\340' >> /pers/.system.overlay/usr/bin/dropbearmulti

$BB dd if=/system/usr/bin/dropbearmulti bs=1 skip=$((0x9292)) \
  >> /pers/.system.overlay/usr/bin/dropbearmulti 2>/dev/null

$BB chmod 0755 /pers/.system.overlay/usr/bin/dropbearmulti
```

Verify:

```sh
$BB wc -c /system/usr/bin/dropbearmulti /pers/.system.overlay/usr/bin/dropbearmulti
# both must be 182676

$BB dd if=/pers/.system.overlay/usr/bin/dropbearmulti bs=1 skip=$((0x9290)) count=2 2>/dev/null | $BB od -A n -t x1
# must print: 0a e0

$BB md5sum /pers/.system.overlay/usr/bin/dropbearmulti
# must print: 4ce94db599ae2a4bd41eda171cc5281d
```

Do not continue until the size, the two bytes and the MD5 are correct.

### 5.4 Passwordless root

The system partition has no `/etc/passwd` or `/etc/shadow`; create the overlay passwd:

```sh
$BB mkdir -p /pers/.system.overlay/etc /pers/.data/root
$BB echo 'root::0:0:root:/data/root:/bin/sh' > /pers/.system.overlay/etc/passwd
$BB chmod 0644 /pers/.system.overlay/etc/passwd
$BB rm -f /pers/.system.overlay/etc/shadow
```

The empty password field is what `-B` allows. The shell is `/bin/sh`.

### 5.5 Host key

The firmware supports `ecdsa`, not `ed25519`:

```sh
$BB mkdir -p /pers/.data/ssh
/system/usr/bin/sshd_keys -t ecdsa -f /pers/.data/ssh/dropbear_ecdsa_host_key
$BB chmod 0600 /pers/.data/ssh/dropbear_ecdsa_host_key
```

If `sshd_keys` fails in the ramfs because the dynamic loader is not available, skip this step and
generate the key from the telnet fallback later (section 6), or let the start script use `-R`.

### 5.6 Start script (SSH + optional telnet fallback)

Create `/pers/.data/ssh/start_ssh.sh`:

```sh
cat > /pers/.data/ssh/start_ssh.sh <<'EOS'
#!/bin/sh
/system/usr/bin/sshd -R -r /data/ssh/dropbear_ecdsa_host_key -B -p 2222 -E >> /data/ssh/dropbear.log 2>&1
/sbin/busybox telnetd -p 2323 -l /bin/sh &
( while : ; do /system/bin/killall update_engine 2>/dev/null ; /system/bin/sleep 5 ; done ) &
EOS
$BB chmod 0755 /pers/.data/ssh/start_ssh.sh
```

Notes:

- `-R` generates missing host keys; `-r` uses the ecdsa key;
- `-B` allows the blank password;
- port 2222 is SSH;
- port 2323 is the telnet fallback; remove that line after setup if you do not want it;
- the `update_engine` kill loop is an extra guard against OTA replacing the system. Also block WAN
  at the router.

### 5.7 Hook into normal boot

`init.rc` runs `/usr/bin/insert_modules.sh` as root. Copy it into the overlay and append the
launcher without removing the original content:

```sh
$BB mkdir -p /pers/.system.overlay/usr/bin
$BB cp /system/usr/bin/insert_modules.sh /pers/.system.overlay/usr/bin/insert_modules.sh
$BB printf '\n# --- persistent ssh ---\nsh /data/ssh/start_ssh.sh &\n' \
  >> /pers/.system.overlay/usr/bin/insert_modules.sh
$BB chmod 0755 /pers/.system.overlay/usr/bin/insert_modules.sh
```

Some builds also call `start_wifi_bt.sh`; if `insert_modules.sh` does not run on your unit, use the
same copy-and-append pattern on `/system/usr/bin/start_wifi_bt.sh`.

### 5.8 Wi-Fi

Pre-write the Wi-Fi configuration:

```sh
$BB mkdir -p /pers/.data/wifi
cat > /pers/.data/wifi/wpa_supplicant.conf <<'EOF'
ctrl_interface=/data/wifi
update_config=1
country=YOUR_COUNTRY
network={
	ssid="YOUR_WIFI"
	psk="YOUR_PASSWORD"
}
EOF
$BB chmod 0644 /pers/.data/wifi/wpa_supplicant.conf
```

Set your country code and credentials. Do not start `wpa_supplicant` in the ramfs shell; the
Android build exits with error 255 there. It starts normally on the next boot.

### 5.9 Sync and reboot

```sh
$BB sync
reboot
```

The normal boot now uses:

- `/pers/.system.overlay/usr/bin/dropbearmulti` — patched binary;
- `/pers/.system.overlay/etc/passwd` — passwordless root;
- `/pers/.system.overlay/usr/bin/insert_modules.sh` — starts `/data/ssh/start_ssh.sh`;
- `/pers/.data/ssh/dropbear_ecdsa_host_key` — host key;
- `/pers/.data/wifi/wpa_supplicant.conf` — Wi-Fi.

---

## 5b. Alternative — override the property (no binary patch)

This is the first workaround: do not touch `dropbearmulti` at all. Instead, make `init` load a
`build.prop` where `ro.product.release.track` is absent or empty. The Dropbear check then compares
an empty value against `stable-channel`, gets a mismatch, and continues normally.

This must be done at **boot time**. `setprop ro.product.release.track ...` from a running system
does **not** work: `ro.*` properties are write-once, and the property was already set from
`build.prop`.

### 5b.1 From the ramfs

```sh
$BB mkdir -p /pers/.system.overlay

# Copy the whole file, then remove only the gate line.
$BB grep -v '^ro\.product\.release\.track=' /system/build.prop \
  > /pers/.system.overlay/build.prop

$BB chmod 0600 /pers/.system.overlay/build.prop
```

Verify that the copy is complete and the line is gone:

```sh
$BB grep -c '^ro\.product\.release\.track=' /pers/.system.overlay/build.prop
# 0

$BB wc -l /system/build.prop /pers/.system.overlay/build.prop
# the overlay copy must be exactly one line shorter
```

Then:

```sh
$BB sync
reboot
```

### 5b.2 Verify after reboot

```sh
getprop ro.product.release.track
# empty

ssh -p 2222 root@SPEAKER_IP
```

If you also want the SSH start script and host key, use section 5.4–5.7; the property override only
removes the gate, it does not start `sshd` by itself.

### 5b.3 Trade-offs and revert

- Copy the **whole** `build.prop`. Writing only a file with the modified line would discard the
  other `ro.build.*` properties.
- The property is gone globally. On this firmware `process_monitor` and `bt_debug_enable` also read
  it, but they only use it for metrics/Bluetooth debug and handle an empty value.
- `init.rc` has `on property:ro.product.release.track=` → `start autostartscript`. That service is
  commented out in the analyzed image; if a build enables it, emptying the property would run
  `/data/autostart.sh`.
- This method is version-independent: it does not depend on the `dropbearmulti` offset.
- Revert on a running system:

```sh
rm -f /cache/.system.overlay/build.prop
sync
reboot
```

or, from the ramfs:

```sh
$BB rm -f /pers/.system.overlay/build.prop
$BB sync
reboot
```

Do not confuse this with `setprop`: runtime `setprop` of an `ro.*` property is rejected by the
property service.

---

## 6. Fallback path — telnet first, then Dropbear

Use this only if section 5 cannot be completed from the ramfs (for example, the ramfs tools are
unavailable or you prefer to patch from the normal system). The ramfs is still needed for the first
step.

### 6.1 From the ramfs: passwordless root + telnet only

```sh
$BB mkdir -p /pers/.system.overlay/etc /pers/.data/root /pers/.data/ssh
$BB echo 'root::0:0:root:/data/root:/bin/sh' > /pers/.system.overlay/etc/passwd
$BB chmod 0644 /pers/.system.overlay/etc/passwd
$BB rm -f /pers/.system.overlay/etc/shadow

cat > /pers/.data/ssh/start_ssh.sh <<'EOS'
#!/bin/sh
/sbin/busybox telnetd -p 2323 -l /bin/sh &
EOS
$BB chmod 0755 /pers/.data/ssh/start_ssh.sh

$BB mkdir -p /pers/.system.overlay/usr/bin
$BB cp /system/usr/bin/insert_modules.sh /pers/.system.overlay/usr/bin/insert_modules.sh
$BB printf '\n# --- telnet ---\nsh /data/ssh/start_ssh.sh &\n' \
  >> /pers/.system.overlay/usr/bin/insert_modules.sh
$BB chmod 0755 /pers/.system.overlay/usr/bin/insert_modules.sh

$BB sync
reboot
```

### 6.2 Connect via telnet

Find the speaker IP in the router, then:

```bash
telnet SPEAKER_IP 2323
# or
nc SPEAKER_IP 2323
```

You get a root shell on the normal system. No password.

### 6.3 Patch Dropbear from telnet

The normal system's `dd` is `toolbox_root`, which rejects `conv=notrunc`. Use the prefix + append
method:

```sh
# Backup
cp /system/usr/bin/dropbearmulti /data/ssh/dropbearmulti.orig
chmod 0755 /data/ssh/dropbearmulti.orig

# Verify the original
md5sum /system/usr/bin/dropbearmulti
# 281e5dd72c70a886de8c2c864b6fe954

dd if=/system/usr/bin/dropbearmulti bs=1 skip=$((0x9290)) count=2 2>/dev/null | od -An -tx1
# 50 b9

# Build the patched copy
dd if=/system/usr/bin/dropbearmulti of=/data/ssh/dropbearmulti \
  bs=1 count=$((0x9290)) 2>/dev/null
printf '\012\340' >> /data/ssh/dropbearmulti
dd if=/system/usr/bin/dropbearmulti bs=1 skip=$((0x9292)) \
  >> /data/ssh/dropbearmulti 2>/dev/null
chmod 0755 /data/ssh/dropbearmulti

# Verify
wc -c /system/usr/bin/dropbearmulti /data/ssh/dropbearmulti
# both 182676
dd if=/data/ssh/dropbearmulti bs=1 skip=$((0x9290)) count=2 2>/dev/null | od -An -tx1
# 0a e0
md5sum /data/ssh/dropbearmulti
# 4ce94db599ae2a4bd41eda171cc5281d
```

### 6.4 Install the patched binary and the host key

Write through the merged `/system` path so overlayfs performs the copy-up. Do not write only to
`/cache/.system.overlay` and expect the merged view to update immediately.

```sh
cp /data/ssh/dropbearmulti /system/usr/bin/dropbearmulti
chmod 0755 /system/usr/bin/dropbearmulti
md5sum /system/usr/bin/dropbearmulti
# 4ce94db599ae2a4bd41eda171cc5281d

/system/usr/bin/sshd_keys -t ecdsa -f /data/ssh/dropbear_ecdsa_host_key
chmod 0600 /data/ssh/dropbear_ecdsa_host_key
```

If `/system` is not writable, use the `/data` copy instead and change the start script to run
`/data/ssh/dropbearmulti sshd ...`.

### 6.5 Replace the telnet-only start script

```sh
cat > /data/ssh/start_ssh.sh <<'EOS'
#!/bin/sh
/system/usr/bin/sshd -R -r /data/ssh/dropbear_ecdsa_host_key -B -p 2222 -E >> /data/ssh/dropbear.log 2>&1
/sbin/busybox telnetd -p 2323 -l /bin/sh &
( while : ; do /system/bin/killall update_engine 2>/dev/null ; /system/bin/sleep 5 ; done ) &
EOS
chmod 0755 /data/ssh/start_ssh.sh
```

Start it immediately from the telnet session, or reboot:

```sh
sh /data/ssh/start_ssh.sh &
```

Then verify from the computer:

```bash
ssh -p 2222 root@SPEAKER_IP
```

---

## 7. Verify after a normal boot

Find the IP in the router, or scan the LAN:

```bash
# Replace <subnet> with your LAN prefix (the first three octets)
for i in $(seq 1 254); do
  nc -z -w1 <subnet>.$i 2222 2>/dev/null && echo "2222 open: <subnet>.$i"
done
```

Check the SSH banner:

```bash
printf '' | nc -w2 SPEAKER_IP 2222
# SSH-2.0-dropbear_...
```

Log in:

```bash
ssh -p 2222 root@SPEAKER_IP
```

On the speaker, confirm the patch is active and the property is still the stock value:

```sh
getprop ro.product.release.track
# stable-channel

md5sum /system/usr/bin/dropbearmulti
# 4ce94db599ae2a4bd41eda171cc5281d

ps | grep sshd
netstat -ltn | grep 2222
cat /data/ssh/dropbear.log
```

The log should show `Running in background` and, after login, `Auth succeeded with blank password`.

Telnet fallback:

```bash
telnet SPEAKER_IP 2323
```

---

## 8. Revert and cleanup

Revert the Dropbear patch (the original squashfs binary becomes visible again):

```sh
rm -f /cache/.system.overlay/usr/bin/dropbearmulti
sync
reboot
```

Close the telnet fallback:

```sh
# edit /data/ssh/start_ssh.sh and remove the telnetd line
# or kill it for the current boot:
killall telnetd
```

Restore the original from the squashfs if needed:

```sh
cp /system.ro/usr/bin/dropbearmulti /cache/.system.overlay/usr/bin/dropbearmulti
chmod 0755 /cache/.system.overlay/usr/bin/dropbearmulti
sync
reboot
```

Remove the SSH hook:

```sh
# remove the "# --- persistent ssh ---" block from
# /cache/.system.overlay/usr/bin/insert_modules.sh
# or delete the overlay copy to restore the original script
rm -f /cache/.system.overlay/usr/bin/insert_modules.sh
sync
reboot
```

---

## 9. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `sshd` exits with `disabled on stable-channel` | The patched binary is not the one being executed. Check `md5sum /system/usr/bin/dropbearmulti`; it must be `4ce94db5…`. |
| `/system/usr/bin/dropbearmulti` still shows the original MD5 after copying to `/cache/.system.overlay` | Overlayfs dentry cache. Write through `/system/usr/bin/dropbearmulti` instead, or reboot. |
| `dd: conv option disabled` | The normal system's `dd` is `toolbox_root`; use the prefix + `printf` + suffix method, not `conv=notrunc`. |
| `head: not integer: c` | The normal system's `head` is the toolbox variant; use `dd bs=1 count=…`, not `head -c`. |
| `ln: link failed No such file or directory` | The toolbox `ln` does not handle combined `-sf`. Use `dropbearmulti sshd …` directly, or `rm -f` then `ln -s`. |
| `sshd_keys -t ed25519` fails | Use `ecdsa`; the firmware has no ed25519 support. |
| Login denied with blank password | `sshd` must run with `-B`; `/etc/passwd` must have an empty password field. |
| SSH port not open | Check `/data/ssh/dropbear.log`, the hook in `/system/usr/bin/insert_modules.sh`, and the Wi-Fi config. |
| `wpa_supplicant` exits 255 in the ramfs | Expected; do not run it there. It starts on normal boot. |
| Serial drops characters | Use `tio -o 5`, paste shorter lines, or split the bootargs. |
| Update engine returns | Keep the `killall` loop and block WAN at the router. |

---

## Appendix — why the patch works

`dropbearmulti`'s server main does:

```c
property_get("ro.product.release.track", buf, "s");
if (bcmp(buf, "stable-channel", 15) == 0) {
    fprintf(stderr, "disabled on stable-channel\n");
    exit(1);
}
```

The check is inside the server-main function at file offset `0x923c` (Ghidra `0x1923c`) in the
1.52.54 binary:

```text
0x9280  blx   property_get@plt
0x9284  ldr   r1, [pc, #0xac]       ; "stable-channel"
0x9288  movs  r2, #0xf
0x928c  blx   bcmp@plt
0x9290  cbnz  r0, #0x92a8           ; different -> continue
0x9292  ...                         ; print + exit(1)
0x92a8  ...                         ; normal startup
```

Changing `0x9290` from `50 b9` to `0a e0` turns the conditional branch into an unconditional
branch to `0x92a8`, so the property value no longer matters.

The original binary remains in `/system.ro/usr/bin/dropbearmulti` and in the `mtd4` squashfs; the
overlay only shadows it. The `data` partition and its overlay are not part of the signed
`boot`/`recovery` images, so this procedure does not touch them.
