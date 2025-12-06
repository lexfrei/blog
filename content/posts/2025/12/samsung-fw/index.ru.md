---
title: "Обновление прошивки Samsung SSD на ARM"
date: 2025-12-07T02:00:00+03:00
slug: "samsung-fw"
draft: false
tags: ["arm", "raspberry-pi", "hardware", "reverse-engineering"]
categories: ["tech"]
---

У меня Raspberry Pi 5 с Samsung 990 EVO. Samsung выпустил новую прошивку. Официальная утилита — только для x86-64. Что делать?

## Когда это нужно

Официальный способ Samsung — загрузиться с ISO и запустить их утилиту. Это не всегда возможно:

- **ARM-системы** — Raspberry Pi, Ampere, Apple Silicon, AWS Graviton. Утилита Samsung просто не запустится.
- **Массовое обновление** — у вас 50 серверов и вы хотите обновить прошивку через Ansible, а не бегать с флешкой.
- **Headless серверы** — нет физического доступа, только SSH. Перезагрузка в recovery — не вариант.
- **Автоматизация в CI/CD** — подготовка железа перед деплоем, firmware как часть пайплайна.
- **Remote management** — сервер в дата-центре, iLO/IPMI есть, но монтировать ISO и ждать — долго и неудобно.

Во всех этих случаях нужен способ залить прошивку из работающей системы через `nvme-cli`.

> **TL;DR**
>
> Samsung шифрует прошивку AES-256-ECB, но ключ лежит прямо в бинарнике. Достаём ключ, расшифровываем, заливаем через стандартный `nvme-cli`. Работает на любой архитектуре.

## Проблема

Samsung распространяет обновления прошивки как загрузочные ISO. Внутри — утилита `fumagician`, которая работает только на x86-64. На ARM её не запустить.

Казалось бы, можно использовать `nvme-cli` — стандартный инструмент для работы с NVMe. Но Samsung шифрует файлы прошивки, и напрямую залить их не получится.

## Решение

Шифрование — AES-256-ECB. Ключ? Лежит прямо в бинарнике `fumagician`. Samsung, видимо, рассчитывали на security through obscurity.

### Шаг 1: Достаём прошивку из ISO

```bash
# Монтируем ISO
sudo mount -o loop Samsung_SSD_990_EVO_1B2QKXJ7.iso /mnt

# Распаковываем initrd
cd /tmp
zcat /mnt/boot/x86_64/loader/initrd | cpio -idmv

# Файлы прошивки в root/fumagician/
ls root/fumagician/*.enc
```

### Шаг 2: Извлекаем ключ

```bash
strings root/fumagician/fumagician | grep -E '^[A-Za-z0-9+/]{43}=$'
```

Эта команда найдёт base64-строку — 32-байтный AES-ключ.

### Шаг 3: Расшифровываем

```python
#!/usr/bin/env python3
from Crypto.Cipher import AES
import base64
import zipfile
import io
import sys

KEY_B64 = "YOUR_KEY_HERE"  # из шага 2
key = base64.b64decode(KEY_B64)

def decrypt_aes_ecb(data: bytes) -> bytes:
    cipher = AES.new(key, AES.MODE_ECB)
    return cipher.decrypt(data)

# Первый слой — расшифровываем .enc файл
with open(sys.argv[1], "rb") as f:
    encrypted = f.read()

decrypted = decrypt_aes_ecb(encrypted)

# Внутри — ZIP-архив
with zipfile.ZipFile(io.BytesIO(decrypted)) as z:
    inner_enc = z.read(z.namelist()[0])

# Второй слой — ещё раз расшифровываем
firmware = decrypt_aes_ecb(inner_enc)

# Убираем PKCS7 padding
padding_len = firmware[-1]
firmware = firmware[:-padding_len]

with open("firmware_payload.bin", "wb") as f:
    f.write(firmware)

print("Done: firmware_payload.bin")
```

### Шаг 4: Заливаем прошивку

```bash
# Загружаем прошивку в SSD
sudo nvme fw-download /dev/nvme0 --fw=firmware_payload.bin --xfer=32768

# Активируем во втором слоте
sudo nvme fw-commit /dev/nvme0 --slot=2 --action=1

# Перезагружаемся
sudo reboot
```

После перезагрузки проверяем:

```bash
sudo nvme id-ctrl /dev/nvme0 | grep fr
# fr: 1B2QKXJ7
```

## Почему это безопасно

Samsung использует двухуровневую защиту:

1. **Транспортный слой** — AES-шифрование при распространении. Именно его мы обходим.
2. **Аппаратный слой** — SSD сам проверяет подпись прошивки. Левую прошивку залить не получится.

Так что это не уязвимость — просто способ доставить официальную прошивку на неподдерживаемую архитектуру.

## Проверено на

- Raspberry Pi 5, Ubuntu 25.10
- Samsung 990 EVO 1TB
- Обновление: 0B2QKXJ7 → 1B2QKXJ7
