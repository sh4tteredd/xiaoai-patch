# L09G — Hardware guide: UART, USB, and when to solder

This guide covers physical access to the Xiaomi L09G. It is for:

- U-Boot console / recovery;
- first boot and SSH setup;
- NAND dump/flash from U-Boot;
- fastboot.

For persistent SSH and network backup, **USB is not required**: UART once, then the network. See
`ssh-and-backup.en.md`.

> **Safety**
> Use the original 12 V supply. Do not power the board through UART. Use **3.3 V** USB-TTL adapters,
> not 5 V. Disconnect power before soldering. A short or a ripped pad can damage the board. The
> reference photo is `board.jpg`.

---

## 1. Board overview

From `board.jpg` (top side):

- PCB `AS09A`, VER 0104, 2020/04/02;
- Toshiba NAND on the left: `TC58NVG1S3HTA00` (2 Gib);
- Amlogic SoC under the central metal shield;
- UART header `J5` at the top right, with `RX`, `GND`, `TX`;
- `USB1`/ESD area on the right, near the edge;
- power connector at the bottom right (12 V).

The actual USB pads are on the **bottom side** of the board; the thread mentions labelled pads and
ESD pads as an alternative. Always check the silkscreen and continuity before soldering.

---

## 2. UART

### 2.1 What you need

- **3.3 V** USB-TTL adapter: CP2102, PL2303, FT232RL (with 3.3 V jumper), CH340 3.3 V.
- 3-pin 2.54 mm header or thin wires.
- Soldering iron, flux, solder, hot glue/tape for strain relief.
- A computer with `tio`, `picocom` or `screen`.

### 2.2 Wiring

Header `J5`:

```text
J5
┌─────────┐
│  RX     │  ← adapter TX
│  GND    │  ← adapter GND
│  TX     │  ← adapter RX
└─────────┘
```

Rules:

- **adapter TX → board RX**
- **adapter RX → board TX**
- **adapter GND → board GND**
- **do NOT connect adapter VCC/5V**

A common ground is mandatory. If the console is silent, the first suspect is swapped TX/RX.

### 2.3 Settings

```text
115200 8N1
no flow control
```

Examples:

```bash
tio -b 115200 -o 5 /dev/ttyUSB0
# or
picocom -b 115200 /dev/ttyUSB0
# or
screen /dev/ttyUSB0 115200
```

`-o 5` adds a small per-character delay: it prevents dropped characters when pasting long lines
(U-Boot bootargs, scripts). If the serial line still drops characters, shorten the lines or increase
the delay.

Permissions:

```bash
sudo setfacl -m u:$USER:rw /dev/ttyUSB0
# or add the user to the dialout/uucp group
```

### 2.4 When it is needed

- First SSH setup: **yes**.
- Bootloop / U-Boot recovery: **yes**.
- NAND dump from U-Boot: **yes**.
- Normal HA satellite use: **no**, after setup network SSH is enough.
- Dump over SSH: **no**, UART is only needed for the first access.

### 2.5 Soldering

If `J5` does not already have pins:

1. disconnect power;
2. solder a 3-pin header or three thin wires;
3. use little solder; avoid bridges between RX/GND/TX;
4. secure the wires with tape or hot glue;
5. check continuity with a multimeter;
6. verify that GND is really GND and not a voltage.

No soldering is needed if the pins are already present: use female-female jumpers.

---

## 3. USB

There are two different uses; do not confuse them.

### 3.1 USB device / fastboot (micro USB)

- Used to connect the speaker to a PC and run `fastboot` from U-Boot.
- Data: `D+`, `D-`, `GND`.
- Power comes from the PC/host; the board remains powered by its 12 V supply.
- The micro USB through-holes are connected to large ground planes: they may need a powerful iron.
  In the thread the connector was glued in place and still worked.
- U-Boot `fastboot` can **flash** but cannot dump NAND.

### 3.2 USB host / USB stick (USB A)

- Used for `usb start` + `fatload`/`fatwrite` in U-Boot, i.e. dump/flash to a FAT32 stick.
- Data: `D+`, `D-`, `GND` from the bottom-side pads.
- The stick needs **+5 V**: connect the USB A socket VBUS pin to the board’s 5 V rail (test point),
  **not** to the micro USB Vbus.
- Use a small, reliable FAT32 stick; avoid powered hubs.
- Verify with:

```text
usb start
fatls usb 0
```

### 3.3 Bottom-side pads

The thread describes:

- test pads on the bottom side for `D+`, `D-`, `GND`;
- if the micro USB pads rip off, the nearby ESD protection device pads can be used with thin wires;
- the USB A socket was fixed with hot glue and flexible wires;
- USB supply voltage must be taken from the bottom side, not from the micro USB pin.

Do not force the connector: the pads rip easily. Use wires with strain relief.

### 3.4 When it is needed

| Activity | UART | USB host (stick) | USB device (fastboot) | Network/SSH |
|---|---:|---:|---:|---:|
| First SSH setup | yes | no | no | after |
| Partition dump over SSH | first access only | no | no | yes |
| Partition dump from U-Boot | yes | yes | no | no |
| Flash from U-Boot with a stick | yes | yes | no | no |
| Fastboot from a PC | yes | no | yes | no |
| Normal use / HA | no | no | no | yes |
| Brick recovery | yes | useful | useful | no |

**Rule of thumb:** if your goal is SSH + network backup, **you do not need to solder USB**. Solder
only if you want U-Boot dump/flash or fastboot.

---

## 4. Power

- Use the original 12 V supply; do not power the board from UART.
- Do not connect 12 V and USB 5 V improperly at the same time.
- For USB A host, the stick’s +5 V comes from the board rail; do not take it from micro USB.
- Before soldering: disconnect everything, discharge capacitors, use a multimeter.
- After the work: check for shorts between +5 V, GND, D+ and D-.

---

## 5. NAND

- Chip: Toshiba `TC58NVG1S3HTA00` (2 Gib = 256 MiB); the U-Boot log identifies it as
  `TC58NVG1S3HBAI4` (compatible ID).
- No need to desolder the NAND for a dump: read it from U-Boot (`nand read`) or Linux
  (`/dev/mtdblockN`).
- The page-by-page `md.b` dump is very slow (~23 h). Prefer `dd`/`cat` from Linux or
  `nand read` + `fatwrite` from U-Boot.
- The NAND has factory bad blocks; an exact clone needs OOB/bad-block handling (`nanddump` or a full
  raw dump). For a partition restore backup, the `mtd0`–`mtd7` files are enough.

---

## 6. Recommended tools

- 3.3 V USB-TTL adapter (CP2102/PL2303/FT232).
- 3-pin 2.54 mm header, thin silicone wires, flux, solder, fine-tip iron.
- Multimeter for continuity/voltage checks.
- Hot glue and tape for strain relief.
- FAT32 USB stick (only for U-Boot host mode).
- Spare USB A or micro USB socket if needed.
- Computer with `tio`/`picocom`/`screen`, `fastboot`, `adb` (for fastboot).

---

## 7. Decision summary

- **Only SSH and network backup?** UART yes, USB no.
- **U-Boot dump/flash?** UART + USB host (stick) yes.
- **Fastboot?** UART + micro USB device yes.
- **Ripped micro USB pads?** Use the ESD pads or a USB A socket with wires, if experienced.
- **Never soldered on a PCB?** Get help, or use UART only if the pins are already present.

For the full software procedure: `ssh-and-backup.en.md`.
For the technical picture: `technical-conclusions.en.md`.
