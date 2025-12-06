---
title: "Updating Samsung SSD Firmware on ARM"
date: 2025-12-07T12:00:00+03:00
slug: "samsung-fw"
draft: false
tags: ["arm", "raspberry-pi", "hardware", "reverse-engineering"]
categories: ["tech"]
---

I have a Raspberry Pi 5 with a Samsung 990 EVO. Samsung released new firmware. Their official utility is x86-64 only. What now?

## When You Need This

Samsung's official method is booting from an ISO and running their utility. That's not always possible:

- **ARM systems** — Raspberry Pi, Ampere, Apple Silicon, AWS Graviton. Samsung's utility simply won't run.
- **Fleet updates** — you have 50 servers and want to update firmware via Ansible, not run around with a USB stick.
- **Headless servers** — no physical access, SSH only. Rebooting into recovery isn't an option.
- **CI/CD automation** — hardware provisioning before deployment, firmware as part of the pipeline.
- **Remote management** — server in a datacenter, iLO/IPMI available, but mounting an ISO and waiting is slow and inconvenient.

In all these cases, you need a way to flash firmware from a running system via `nvme-cli`.

> **TL;DR**
>
> Samsung encrypts firmware with AES-256-ECB, but the key is right there in the binary. Extract the key, decrypt, flash via standard `nvme-cli`. Works on any architecture.

## The Problem

Samsung distributes firmware updates as bootable ISOs. Inside is `fumagician`, a utility that only runs on x86-64. Can't run it on ARM.

You'd think `nvme-cli` — the standard tool for NVMe operations — would work. But Samsung encrypts the firmware files, so direct flashing fails.

## The Solution

The encryption is AES-256-ECB. The key? Sitting right inside the `fumagician` binary. Samsung apparently bet on security through obscurity.

### Step 1: Extract Firmware from ISO

```bash
# Mount the ISO
sudo mount -o loop Samsung_SSD_990_EVO_1B2QKXJ7.iso /mnt

# Extract initrd
cd /tmp
zcat /mnt/boot/x86_64/loader/initrd | cpio -idmv

# Firmware files are in root/fumagician/
ls root/fumagician/*.enc
```

### Step 2: Extract the Key

```bash
strings root/fumagician/fumagician | grep -E '^[A-Za-z0-9+/]{43}=$'
```

This finds a base64 string — the 32-byte AES key.

### Step 3: Decrypt

```python
#!/usr/bin/env python3
from Crypto.Cipher import AES
import base64
import zipfile
import io
import sys

KEY_B64 = "YOUR_KEY_HERE"  # from step 2
key = base64.b64decode(KEY_B64)

def decrypt_aes_ecb(data: bytes) -> bytes:
    cipher = AES.new(key, AES.MODE_ECB)
    return cipher.decrypt(data)

# First layer — decrypt .enc file
with open(sys.argv[1], "rb") as f:
    encrypted = f.read()

decrypted = decrypt_aes_ecb(encrypted)

# Inside is a ZIP archive
with zipfile.ZipFile(io.BytesIO(decrypted)) as z:
    inner_enc = z.read(z.namelist()[0])

# Second layer — decrypt again
firmware = decrypt_aes_ecb(inner_enc)

# Remove PKCS7 padding
padding_len = firmware[-1]
firmware = firmware[:-padding_len]

with open("firmware_payload.bin", "wb") as f:
    f.write(firmware)

print("Done: firmware_payload.bin")
```

### Step 4: Flash the Firmware

```bash
# Upload firmware to SSD
sudo nvme fw-download /dev/nvme0 --fw=firmware_payload.bin --xfer=32768

# Activate in slot 2
sudo nvme fw-commit /dev/nvme0 --slot=2 --action=1

# Reboot
sudo reboot
```

After reboot, verify:

```bash
sudo nvme id-ctrl /dev/nvme0 | grep fr
# fr: 1B2QKXJ7
```

## Why This Is Safe

Samsung uses two-layer protection:

1. **Transport layer** — AES encryption for distribution. This is what we're bypassing.
2. **Hardware layer** — The SSD itself verifies firmware signatures. Can't flash unofficial firmware.

So this isn't a vulnerability — just a way to deliver official firmware to an unsupported architecture.

## Tested On

- Raspberry Pi 5, Ubuntu 25.10
- Samsung 990 EVO 1TB
- Update: 0B2QKXJ7 → 1B2QKXJ7
