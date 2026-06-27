# research-notes.md

Research Notes — Phase 0  
Project: epson-l3110-driver  
Status: in progress

---

## 1. Device Identification

Obtained with `lsusb -v -d 04b8:1142` on Debian, with the printer connected over USB.

| Field | Value |
|-------|-------|
| Vendor ID | `0x04b8` (Seiko Epson Corp.) |
| Product ID | `0x1142` (L3110 Series) |
| USB Speed | Full Speed — 12 Mbps |
| USB Version | 1.10 |
| Serial Number | `583634343538313161` |
| Configurations | 1 |
| Configuration Name | `USB MFP` |
| Power | Self Powered, 2 mA |

---

## 2. Interfaces and Endpoints

The device exposes two interfaces under its single configuration.

### Interface 0 — Scanner

| Field | Value |
|-------|-------|
| Number | 0 |
| Class | 255 — Vendor Specific |
| Subclass | 255 — Vendor Specific |
| Protocol | 255 — Vendor Specific |
| Name | `EPSON Scanner` |

| Endpoint | Direction | Type | Packet Size |
|----------|-----------|------|-------------|
| EP 1 `0x81` | IN | Bulk | 64 bytes |
| EP 2 `0x02` | OUT | Bulk | 64 bytes |

This interface corresponds to the scanner. Ignored in Phases 0-3. Do not claim this interface when opening the device for printing.

### Interface 1 — Printer

| Field | Value |
|-------|-------|
| Number | 1 |
| Class | 7 — Printer |
| Subclass | 1 — Printer |
| Protocol | 2 — Bidirectional |
| Name | `USB Printer` |

| Endpoint | Direction | Type | Packet Size | Purpose |
|----------|-----------|------|-------------|---------|
| EP 3 `0x83` | IN | Bulk | 64 bytes | Printer responses and status |
| EP 4 `0x04` | OUT | Bulk | 64 bytes | Print commands and data |

This is the interface we will claim with `libusb` to communicate with the printer.

**Class 7 with Protocol 2 (Bidirectional)** means the device follows the USB Printer Class standard defined in "Universal Serial Bus Device Class Definition for Printing Devices" (USB-IF). This implies:

- The IEEE 1284 Device ID can be read via a class-specific control transfer.
- Communication follows documented conventions at the USB level, not a fully proprietary transport.

---

## 3. IEEE 1284 Device ID

Not visible in `lsusb` output. Must be read via a USB class control transfer:

```
bmRequestType: 0xA1  (Device-to-Host | Class | Interface)
bRequest:      0x00  (GET_DEVICE_ID)
wValue:        0x0000
wIndex:        0x0001  (Interface 1)
wLength:       1024   (receive buffer size)
```

The result is an IEEE 1284 formatted string containing fields such as:

```
MFG:EPSON;CMD:ESCPL2,BDC,ESCPR7,...;MDL:L3110;...
```

The `CMD` field lists the command languages the printer accepts. This will confirm whether the L3110 uses ESC/P-R (ESCPR) and which version. **Pending: read in Phase 2.**

---

## 4. Linux Printing Stack

```
Application
    |
    v
CUPS scheduler (cupsd)
    |
    +---> Filter chain
    |         PDF -> PS -> CUPS Raster -> printer format
    v
Backend (USB)
    |
    v
Printer
```

### CUPS

- Standard printing system on Linux and macOS.
- Manages queues, filters, and backends.
- Configured via PPD files (PostScript Printer Description).
- Our driver intervenes at two points: the final filter and optionally the backend.

### Filter

- An executable that transforms data between formats.
- Reads from `stdin`, writes to `stdout`.
- The filter we will write converts CUPS Raster to ESC/P-R.

### Backend

- An executable that sends converted data to the hardware.
- The generic CUPS USB backend (`/usr/lib/cups/backend/usb`) is sufficient initially.
- In later phases we will write our own for full control.

---

## 5. USB Core Concepts

| Concept | Description |
|---------|-------------|
| VID/PID | Identify manufacturer and model |
| Device descriptor | Data structure describing the full device |
| Configuration | Groups interfaces; we activate one |
| Interface | Logical function (printer, scanner) |
| Endpoint | Unidirectional communication channel |
| Bulk transfer | High-volume data transfer without timing guarantees; used for printing |
| Control transfer | Control channel on EP 0; used for Device ID and soft reset |
| Interrupt transfer | Status notifications; not present on this printer |

---

## 6. ESC/P-R

Epson has three generations of print protocol:

| Protocol | Use |
|----------|-----|
| ESC/P | Dot matrix printers, 1980s |
| ESC/P2 | Older inkjet printers; supported by Gutenprint |
| ESC/P-R | Modern inkjet printers, including the L3110 |

ESC/P-R wraps commands in packets with a sync header, command code, length, and data payload. Official documentation covers a subset of commands. Some commands will need to be inferred from USB traffic captures.

The `CMD` field from the Device ID will confirm the exact ESC/P-R version supported by the L3110. Expected value: `ESCPR7` or similar.

---

## 7. Development Tools

| Tool | Purpose |
|------|---------|
| `lsusb` | USB descriptor inspection |
| `libusb-1.0` | USB device communication from userspace |
| `usbmon` + Wireshark | USB traffic capture for protocol analysis |
| Valgrind | Memory leak detection |
| AddressSanitizer | Buffer overflow and invalid memory access detection |

---

## 8. Phase 0 Pending Items

- [ ] Read Device ID via control transfer (done in Phase 2 with C code).
- [ ] Analyze reference projects: Gutenprint, CUPS USB backend, OpenPrinting escpr.
- [ ] Produce `architecture.md`.
- [ ] Produce `protocol-notes.md`.
- [ ] Capture USB traffic from a real print job using the official Epson driver as reference.

---

## 9. References

- USB Printer Class Definition: https://www.usb.org/document-library/class-definitions-communications-devices-revision-12
- Epson ESC/P-2 Reference Manual: https://files.support.epson.com/pdf/general/escp2ref.pdf
- OpenPrinting escpr: https://github.com/openprinting/epson-inkjet-printer-escpr
- CUPS filter API: https://www.cups.org/doc/api-filter.html
- libusb API: https://libusb.sourceforge.io/api-1.0/