# protocol-notes.md

Protocol Notes — ESC/P-R
Project: epson-l3110-driver
Status: Phase 0 — updated with USB traffic capture evidence

---

## 1. Protocol Family

The Epson L3110 uses ESC/P-R (Epson Standard Code for Printers — Raster).

| Protocol | Era | Target hardware |
|----------|-----|-----------------|
| ESC/P | 1980s | Dot matrix printers |
| ESC/P2 | 1990s | Early inkjet; supported by Gutenprint |
| ESC/P-R | 2000s-present | Modern inkjet including EcoTank series |

ESC/P-R is not fully documented publicly. The ESC/P-2 Reference Manual (Epson, 1997)
covers the earlier protocol. Commands below were confirmed by USB traffic capture on
a real L3110 print job (plain text, one page).

---

## 2. IEEE 1284 Device ID

Read via USB class control transfer (GET_DEVICE_ID, bmRequestType=0xA1, bRequest=0x00,
wIndex=0x0001). Confirmed from captured traffic.

```
MFG:EPSON;CMD:ESCPL2,BDC,D4,D4PX,ESCPR1,END4;MDL:L3110 Series;
CLS:PRINTER;DES:EPSON L3110 Series;CID:EpsonRGB;
FID:FXN,DPN,WFN,ETN,AFN,DAN,WRN;RID:30;DDS:022180;ELG:1163;
```

### CMD field breakdown

| Token | Meaning |
|-------|---------|
| ESCPL2 | ESC/P-L2, base raster protocol |
| BDC | Bidirectional communication |
| D4 | IEEE 1284.4 multiplexed transport |
| D4PX | D4 variant |
| ESCPR1 | ESC/P-R version 1 — primary print protocol |
| END4 | ESC/P-R encapsulated over D4 |

The printer uses ESCPR1, not ESCPR7 as some sources suggest for other models.

---

## 3. Transport Layer: EJL over IEEE 1284.4

Before any ESC/P-R command is sent, the driver establishes a session using EJL
(Epson Job Language) over IEEE 1284.4. This was not visible in source code analysis
and was discovered only from the USB capture.

EJL is a session layer that multiplexes channels over USB. The printer expects this
handshake before accepting ESC/P-R data.

---

## 4. ESC/P-R Command Structure

Every ESC/P-R command follows this structure:

```
[ ESC ] [ cmd: 1B ] [ length: 4B LE ] [ name: 4B ASCII ] [ data: length bytes ]
  0x1B     1 byte      uint32_t LE        4 bytes             variable
```

The 4-byte ASCII name field was not documented in source code analysis. It was
confirmed from the USB capture.

Example — dsnd (send raster data):
```
1b  64  87 00 00 00  64 73 6e 64  [payload]
ESC  d   len=135      "dsnd"       135 bytes of compressed raster
```

---

## 5. Confirmed Print Job Sequence

Captured from a real print job on the L3110. Bytes are hex.

### Step 1: EJL session entry
```
1b 01 40 45 4a 4c 20 31 32 38 34 2e 34 0a
```
ASCII: ESC \x01 @EJL 1284.4\n

### Step 2: EJL reset
```
40 45 4a 4c 20 20 20 20 20 20 0a
```
ASCII: @EJL      \n

### Step 3: Printer reset
```
1b 40
```
ASCII: ESC @
Standard ESC/P printer reset to initial state.

### Step 4: Enter REMOTE mode
```
1b 28 52 08 00 00 52 45 4d 4f 54 45 31
```
ASCII: ESC ( R [len=8] REMOTE1
Configuration mode before job parameters.

### Step 5: TI command (job identification)
```
54 49 08 00 00 07 6c 01 01 01 01 01
```
Command: TI, length=8. Purpose: job timestamp or identification. Parameters not
fully decoded.

### Step 6: JS command (job start)
```
4a 53 04 00 00 00 00 00
```
Command: JS, length=4, data=all zeros. Signals formal job start.

### Step 7: JH command (job header)
```
4a 48 0e 00 00 00 00 00 00 00 45 53 43 50 52 4c 69 62
```
Command: JH, length=14. Contains ASCII string "ESCPRLib" — library identifier.

### Step 8: HD command (header)
```
48 44 03 00 00 03 04 50
```
Command: HD, length=3. Purpose not fully decoded.

### Step 9: PP command (paper configuration)
```
50 50 03 00 00 01 ff
```
Command: PP, length=3. Paper settings. Parameters not fully decoded.

### Step 10: Padding and ESC/P-R mode entry
```
1b 00 00 00
1b 28 52 06 00 00 45 53 43 50 52
```
ESC \0\0\0 followed by: ESC ( R [len=6] ESCPR
Switches from REMOTE mode into ESC/P-R raster mode.

### Step 11: setq command (print quality)
```
1b 71 09 00 00 00 73 65 74 71 00 01 00 00 00 00 00 00 00
```
ESC q [len=9] "setq" [parameters]
Configures print quality. Parameter meanings not fully decoded.

### Step 12: setj command (job dimensions)
```
1b 6a 16 00 00 00 73 65 74 6a 00 00 0b a0 00 00 10 71 00 2a 00 2a 00 00 0b 4c 00 00 10 1d 00 00
```
ESC j [len=22] "setj" [parameters]
Notable values: 0x0ba0 = 2976, 0x1071 = 4209. Likely page dimensions in internal
units. 0x2a = 42 appears twice, possibly resolution or margin parameter.

### Step 13: sttp command (start page)
```
1b 70 00 00 00 00 73 74 74 70
```
ESC p [len=0] "sttp"

### Step 14: setn command
```
1b 70 01 00 00 00 73 65 74 6e 00
```
ESC p [len=1] "setn" \x00
Purpose not fully decoded. Possibly nozzle configuration.

### Step 15: dsnd command (send raster line) — repeats per line
```
1b 64 [len: 4B LE] 64 73 6e 64 [line_number: 4B] [compressed_data]
```
ESC d [length] "dsnd" [line number] [RLE compressed pixel data]

Line numbers observed in capture: 0x09, 0x0a, 0x0b, 0x0c, 0x0d (ascending).
The first lines (0x00-0x08) are likely blank margin lines or sent before the
capture started.

### Step 16: End sequence
Not captured. Will be determined in Phase 3 by capturing a complete job with
the end-of-page and end-of-job packets included.

---

## 6. Raster Data Format (dsnd payload)

Raster data inside dsnd is RLE compressed (Epson variant).

From source code analysis (epson-escpr-api.c):
- A run: repeat byte N followed by data byte D means repeat D for (N+1) times.
- A literal: literal count byte N followed by (N+1) raw bytes.

Maximum packet size: 4096 bytes. Lines longer than 4096 bytes are split across
multiple dsnd packets.

Observed in capture: lines contain groups like:
```
80 ff ff ff  -- repeat: 0x80=128+1 times the value 0xff (white pixels)
00 00 00     -- literal or repeat of 0x00 (black pixels)
```

The value 0x80 as a repeat byte with 0xff data likely means 129 white bytes,
which is consistent with blank margins at the start of each raster line.

---

## 7. Pending / Unknown

- TI command parameter meaning.
- HD command parameter meaning.
- PP command parameter meaning (paper size encoding).
- setn command purpose.
- setj dimension unit (pixels? dots? 1/720 inch?).
- setq quality parameter encoding.
- End-of-page command (not captured).
- End-of-job command (not captured).
- Whether the printer sends status responses on EP 3 IN during a job.

---

## 8. Sources

- USB traffic capture: real L3110 print job, Debian 13, tshark/usbmon3, June 2025.
- mrnuke/epson-printer-escpr-improved: RLE algorithm, packet size limit.
- ezrec/python-epson: command structure cross-reference.
- Epson ESC/P-2 Reference Manual (1997): https://files.support.epson.com/pdf/general/escp2ref.pdf
- USB Printer Class Definition (USB-IF): GET_DEVICE_ID control transfer spec.
