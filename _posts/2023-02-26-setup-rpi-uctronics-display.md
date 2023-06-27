---
layout: post
title: "Autostarting Uctronics Raspberry Pi OLED Display"
date: 2023-02-26 12:00:00 -0500
category: "Service Setup"
tags: ["raspberry pi", "uctronics"]
---

This is a quick setup for both the Uctronics display code, and the startup service. I will also give an example using `rc.local`.

<!--more-->

## Preparing the Pi

You will need to activate I2C on your raspberry pi. This can be done using the following command, and then navigating the menu to Interface Options > I2C and enabling the interface.

```bash
sudo raspi-config
```

## Creating the Binary

First, you will need to pull down the Uctronics code. I feel like they change the repository names so [I have forked the repo](https://github.com/BinaryPatrick/U6143_ssd1306). Use the following command to pull down my fork, or follow the link and github to Uctronics codebase.

```bash
git clone https://github.com/BinaryPatrick/U6143_ssd1306.git
```

Once it's in place, you will need to navigate to the `C` folder inside

```bash
cd ./U6143_ssd1306/C/
```

Then you can use the `make` command to compile to C code for your architecture.

```bash
sudo make clean && sudo make
```

Once the binary is created you should see a new file named `display` in the folder with the execute permission set. Ensure it works by running the following command.

```bash
./display
```

You should see the screen come to life and start displaying status about the raspberry pi. `CTRL+C` will stop the script running. The OLED screen will freeze in whatever state it was in. As far as I can tell this is normal. The screen won't clear until it loses power.

Now that you've compiled and tested the file, it will need to be copied to the user space binary folder.

```bash
sudo mv ./display /usr/bin/uctronics-display
```

> Note that the binary is renamed to `uctronics-display` with the command
> {: .prompt-warning }

## Autostart

Now that the binary is in place, we need to configure a way to start it automatically.

### rc.local

The easiest way to configure the binary to run automatically is to add a line to the `rc.local` file.

```bash
sudo nano /etc/rc.local
```

Add a line at the bottom before `exit 0`.

```text
/usr/bin/uctronics-display &
```

The ampersand at the end ensures the binary starts as a background task, allowing the script to continue and exit with state 0.

When you reboot, you should see the screen start automatically.

### Creating a Service

Navigate to the systemd system folder.

```bash
cd /etc/systemd/system
```

Once there, create a new service file.

```bash
sudo nano uctronics-display.service
```

```text
[Unit]
Description=UCTRONICS OLED Display Service
After=network.target
StartLimitIntervalSec=0
[Service]
Type=simple
Restart=always
RestartSec=1
User=patrick
ExecStart=/usr/bin/uctronics-display

[Install]
WantedBy=multi-user.target
```

Now you can start your service and enable it's run on startup with the following commands.

```bash
sudo systemctl start uctronics-display
sudo systemctl enable uctronics-display
```

Now you can check the status of your service with the following command.

```bash
sudo systemctl status uctronics-display
```

When you reboot, you should see the screen start automatically.
