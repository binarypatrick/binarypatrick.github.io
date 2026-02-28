---
layout: post
title: "Creating QR Codes for WiFi"
pubDatetime: 2024-07-21
category: "General"
tags: ["Linux", "QR"]
description:
  "Here's an easy way to create a QR code image you can print out and display
  so people can quickly join your WiFi network. In Debian, install qrencode."
---

Here's an easy way to create a QR code image you can print out and display so people can quickly join your WiFi network. In Debian, install `qrencode`.

```bash
apt install qrencode
```

Once it's installed you can create a QR code as a PNG image.

```bash
qrencode -s 24 -l H -o "wifi.png" "WIFI:T:WPA;S:<SSID>;P:<Password>;;"
```

In this example, `-s 24` indicates the image size. `-l H` indicates high error correction, `-o wifi.png` is the output file, and the encoded connection details are last.

For more information, here is the qrencode man page
<https://linux.die.net/man/1/qrencode>
