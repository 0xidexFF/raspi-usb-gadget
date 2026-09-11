# Raspberry Pi 4 USB HID Keyboard + Mouse + Camera

## Description

This document configures a Raspberry Pi 4 to operate as a USB HID
(Human Interface Device) containing:

- USB keyboard
- USB mouse
- Raspberry Pi Camera Module 3

The Raspberry Pi can then be connected to another computer through
the Pi's USB-C port.

The USB connection presents the Pi as a keyboard and mouse.

The Camera Module 3 remains connected through CSI and can be used by
a Python application for live video.

---

# System Information

This setup is intended for:

- Raspberry Pi 4
- Raspberry Pi OS / Debian 13 (Trixie)
- Linux kernel 6.x
- Camera Module 3 / IMX708
- USB-C peripheral/gadget mode

---

# Architecture

```text
                       Raspberry Pi 4
                  ┌─────────────────────┐
                  │                     │
 Camera Module 3 ─┤ CSI                │
                  │                     │
                  │ USB Gadget          │
                  │                     │
                  │   HID Keyboard      │
                  │   HID Mouse         │
                  │                     │
                  └──────────┬──────────┘
                             │
                           USB-C
                             │
                             ▼
                       Target Computer
```

The camera does NOT use USB gadget mode.

The camera is connected to the Raspberry Pi CSI connector.

The keyboard and mouse are presented through USB gadget mode.

---

# PART 1 — Check USB Gadget Support

## Description

The Raspberry Pi must operate its USB controller in peripheral mode.

The DWC2 controller provides USB device/gadget functionality.

## Check the USB Device Controller

Run:

```bash
ls -la /sys/class/udc/
```

Expected:

```text
fe980000.usb
```

If `fe980000.usb` exists, the USB controller is available.

---

# PART 2 — Check DWC2 Configuration

## Description

The `dwc2` overlay tells the Raspberry Pi to use the USB controller.

Run:

```bash
grep dwc2 /boot/firmware/config.txt
```

Expected:

```text
dtoverlay=dwc2,dr_mode=peripheral
```

The important part is:

```text
dr_mode=peripheral
```

This tells the controller to behave as a USB device instead of a USB host.

---

# PART 3 — Check Kernel Module

Run:

```bash
grep -o 'modules-load=[^ ]*' /boot/firmware/cmdline.txt
```

Expected:

```text
modules-load=dwc2
```

Load the module manually:

```bash
sudo modprobe dwc2
```

Load the USB gadget framework:

```bash
sudo modprobe libcomposite
```

Check:

```bash
lsmod | grep -E 'dwc2|libcomposite'
```

---

# PART 4 — ConfigFS

## Description

Linux USB gadgets are normally created through ConfigFS.

ConfigFS is mounted at:

```text
/sys/kernel/config
```

Check:

```bash
mount | grep configfs
```

If ConfigFS is not mounted:

```bash
sudo mount -t configfs none /sys/kernel/config
```

Check again:

```bash
mount | grep configfs
```

---

# PART 5 — Create USB Gadget

Create the gadget directory:

```bash
sudo mkdir -p /sys/kernel/config/usb_gadget/pi-hid
```

Enter it:

```bash
cd /sys/kernel/config/usb_gadget/pi-hid
```

---

# PART 6 — USB Device Identity

## Description

These values identify the USB gadget to the host computer.

For development/testing, this configuration uses a generic development
identity.

For a commercial product, use a properly assigned USB Vendor ID and
Product ID.

Set USB version:

```bash
sudo sh -c 'echo 0x0200 > bcdUSB'
```

Set device revision:

```bash
sudo sh -c 'echo 0x0100 > bcdDevice'
```

Create device strings:

```bash
sudo mkdir -p strings/0x409
```

Serial number:

```bash
sudo sh -c 'echo "PI-HID-0001" > strings/0x409/serialnumber'
```

Manufacturer:

```bash
sudo sh -c 'echo "Raspberry Pi" > strings/0x409/manufacturer'
```

Product name:

```bash
sudo sh -c 'echo "USB Keyboard Mouse" > strings/0x409/product'
```

---

# PART 7 — USB Configuration

Create configuration:

```bash
sudo mkdir -p configs/c.1/strings/0x409
```

Set configuration description:

```bash
sudo sh -c 'echo "Keyboard Mouse" > configs/c.1/strings/0x409/configuration'
```

Set maximum bus power:

```bash
sudo sh -c 'echo 250 > configs/c.1/MaxPower'
```

---

# PART 8 — Keyboard HID Function

## Description

This creates a standard USB HID boot keyboard.

The keyboard uses an 8-byte HID report:

```text
Byte 0 = modifier keys
Byte 1 = reserved
Byte 2 = key
Byte 3-7 = additional keys
```

Create keyboard function:

```bash
sudo mkdir -p functions/hid.keyboard
```

Set HID subclass:

```bash
sudo sh -c 'echo 1 > functions/hid.keyboard/subclass'
```

Set keyboard protocol:

```bash
sudo sh -c 'echo 1 > functions/hid.keyboard/protocol'
```

Set report size:

```bash
sudo sh -c 'echo 8 > functions/hid.keyboard/report_length'
```

Create keyboard HID descriptor:

```bash
sudo sh -c 'printf "\x05\x01\x09\x06\xa1\x01\x05\x07\x19\xe0\x29\xe7\x15\x00\x25\x01\x75\x01\x95\x08\x81\x02\x95\x01\x75\x08\x81\x03\x95\x05\x75\x01\x05\x08\x19\x01\x29\x05\x91\x02\x95\x01\x75\x03\x91\x03\x95\x06\x75\x08\x15\x00\x25\x65\x05\x07\x19\x00\x29\x65\x81\x00\xc0" > functions/hid.keyboard/report_desc'
```

---

# PART 9 — Mouse HID Function

## Description

This creates a standard USB HID mouse.

The mouse report contains:

```text
Byte 0 = mouse buttons
Byte 1 = X movement
Byte 2 = Y movement
Byte 3 = wheel movement
```

Create mouse function:

```bash
sudo mkdir -p functions/hid.mouse
```

Set HID subclass:

```bash
sudo sh -c 'echo 1 > functions/hid.mouse/subclass'
```

Set mouse protocol:

```bash
sudo sh -c 'echo 2 > functions/hid.mouse/protocol'
```

Set report size:

```bash
sudo sh -c 'echo 4 > functions/hid.mouse/report_length'
```

Create mouse HID descriptor:

```bash
sudo sh -c 'printf "\x05\x01\x09\x02\xa1\x01\x09\x01\xa1\x00\x05\x09\x19\x01\x29\x03\x15\x00\x25\x01\x95\x03\x75\x01\x81\x02\x95\x01\x75\x05\x81\x01\x05\x01\x09\x30\x09\x31\x09\x38\x15\x81\x25\x7f\x75\x08\x95\x03\x81\x06\xc0\xc0" > functions/hid.mouse/report_desc'
```

---

# PART 10 — Attach Keyboard and Mouse

Create the keyboard link:

```bash
sudo ln -s functions/hid.keyboard configs/c.1/hid.keyboard
```

Create the mouse link:

```bash
sudo ln -s functions/hid.mouse configs/c.1/hid.mouse
```

Check:

```bash
ls -l configs/c.1/
```

The configuration should contain links for:

```text
hid.keyboard
hid.mouse
```

---

# PART 11 — Enable USB Gadget

## Description

This connects the USB gadget configuration to the Raspberry Pi USB
Device Controller.

First check the controller:

```bash
ls /sys/class/udc/
```

Expected:

```text
fe980000.usb
```

Enable:

```bash
sudo sh -c 'echo fe980000.usb > UDC'
```

Check:

```bash
cat UDC
```

Expected:

```text
fe980000.usb
```

At this point the USB gadget is active.

---

# PART 12 — Check HID Devices

Run:

```bash
ls -l /dev/hidg*
```

Expected:

```text
/dev/hidg0
/dev/hidg1
```

Normally:

```text
/dev/hidg0 = keyboard
/dev/hidg1 = mouse
```

---

# PART 13 — Keyboard Test

## Description

The following sends the letter `A`.

HID key code 4 represents `A`.

Press:

```bash
sudo sh -c 'printf "\x00\x00\x04\x00\x00\x00\x00\x00" > /dev/hidg0'
```

Release:

```bash
sudo sh -c 'printf "\x00\x00\x00\x00\x00\x00\x00\x00" > /dev/hidg0'
```

The target computer should receive:

```text
A
```

---

# PART 14 — Mouse Test

## Move Mouse

The following moves the mouse to the right.

```bash
sudo sh -c 'printf "\x00\x32\x00\x00" > /dev/hidg1'
```

## Left Click

Press:

```bash
sudo sh -c 'printf "\x01\x00\x00\x00" > /dev/hidg1'
```

Release:

```bash
sudo sh -c 'printf "\x00\x00\x00\x00" > /dev/hidg1'
```

---

# PART 15 — Camera Module 3

## Description

Camera Module 3 uses the IMX708 image sensor.

The camera connects directly to the Raspberry Pi CSI connector.

It does not use the USB HID gadget.

Check camera detection:

```bash
rpicam-hello --list-cameras
```

Expected:

```text
0 : imx708
```

---

# PART 16 — Camera 1080p Test

Preview at 1920x1080:

```bash
rpicam-hello \
    --width 1920 \
    --height 1080
```

Press:

```text
Ctrl+C
```

to stop.

---

# PART 17 — Record 1080p Video

Record 10 seconds:

```bash
rpicam-vid \
    -t 10000 \
    --width 1920 \
    --height 1080 \
    --framerate 30 \
    -o video.h264
```

Record continuously:

```bash
rpicam-vid \
    -t 0 \
    --width 1920 \
    --height 1080 \
    --framerate 30 \
    -o video.h264
```

Stop:

```text
Ctrl+C
```

---

# PART 18 — HID Permissions

## Description

By default `/dev/hidg0` and `/dev/hidg1` may only be writable by root.

Create a dedicated group:

```bash
sudo groupadd -f hidusers
```

Add user `puppy`:

```bash
sudo usermod -aG hidusers puppy
```

Create udev rule:

```bash
sudo nano /etc/udev/rules.d/99-hidg.rules
```

Add:

```text
KERNEL=="hidg[0-9]*", GROUP="hidusers", MODE="0660"
```

Save and reload:

```bash
sudo udevadm control --reload-rules
```

Apply:

```bash
sudo udevadm trigger
```

Log out and log back in.

Check:

```bash
groups
```

The output should contain:

```text
hidusers
```

Check device permissions:

```bash
ls -l /dev/hidg*
```

---

# PART 19 — Permanent HID Startup Script

## Description

The following script creates the USB keyboard and mouse automatically.

Create:

```bash
sudo nano /usr/local/bin/pi-usb-hid.sh
```

Paste:

```bash
#!/bin/bash

# Stop immediately if a command fails.
set -e

# USB gadget directory.
G=/sys/kernel/config/usb_gadget/pi-hid

# Load USB gadget modules.
modprobe dwc2
modprobe libcomposite

# Wait for the Raspberry Pi USB Device Controller.
for i in {1..30}; do

    UDC=$(ls /sys/class/udc 2>/dev/null | head -n1)

    if [ -n "$UDC" ]; then
        break
    fi

    sleep 1

done

# Stop if no USB controller was found.
if [ -z "$UDC" ]; then

    echo "ERROR: USB Device Controller not found"

    exit 1

fi

# Make sure ConfigFS is mounted.
if ! mountpoint -q /sys/kernel/config; then

    mount -t configfs none /sys/kernel/config

fi

# Create gadget.
mkdir -p "$G"

cd "$G"

# USB version.
echo 0x0200 > bcdUSB

# Device revision.
echo 0x0100 > bcdDevice

# Device strings.
mkdir -p strings/0x409

echo "PI-HID-0001" \
    > strings/0x409/serialnumber

echo "Raspberry Pi" \
    > strings/0x409/manufacturer

echo "USB Keyboard Mouse" \
    > strings/0x409/product

# USB configuration.
mkdir -p configs/c.1/strings/0x409

echo "Keyboard Mouse" \
    > configs/c.1/strings/0x409/configuration

echo 250 > configs/c.1/MaxPower

# ------------------------------------------------
# KEYBOARD
# ------------------------------------------------

mkdir -p functions/hid.keyboard

echo 1 > functions/hid.keyboard/subclass
echo 1 > functions/hid.keyboard/protocol
echo 8 > functions/hid.keyboard/report_length

printf '\x05\x01\x09\x06\xa1\x01\x05\x07\x19\xe0\x29\xe7\x15\x00\x25\x01\x75\x01\x95\x08\x81\x02\x95\x01\x75\x08\x81\x03\x95\x05\x75\x01\x05\x08\x19\x01\x29\x05\x91\x02\x95\x01\x75\x03\x91\x03\x95\x06\x75\x08\x15\x00\x25\x65\x05\x07\x19\x00\x29\x65\x81\x00\xc0' \
    > functions/hid.keyboard/report_desc

# ------------------------------------------------
# MOUSE
# ------------------------------------------------

mkdir -p functions/hid.mouse

echo 1 > functions/hid.mouse/subclass
echo 2 > functions/hid.mouse/protocol
echo 4 > functions/hid.mouse/report_length

printf '\x05\x01\x09\x02\xa1\x01\x09\x01\xa1\x00\x05\x09\x19\x01\x29\x03\x15\x00\x25\x01\x95\x03\x75\x01\x81\x02\x95\x01\x75\x05\x81\x01\x05\x01\x09\x30\x09\x31\x09\x38\x15\x81\x25\x7f\x75\x08\x95\x03\x81\x06\xc0\xc0' \
    > functions/hid.mouse/report_desc

# Remove existing links.
rm -f configs/c.1/hid.keyboard
rm -f configs/c.1/hid.mouse

# Attach keyboard.
ln -s functions/hid.keyboard \
    configs/c.1/hid.keyboard

# Attach mouse.
ln -s functions/hid.mouse \
    configs/c.1/hid.mouse

# Enable USB gadget.
echo "$UDC" > UDC

echo
echo "======================================"
echo " USB HID GADGET ENABLED"
echo "======================================"
echo "UDC      : $UDC"
echo "Keyboard : /dev/hidg0"
echo "Mouse    : /dev/hidg1"
echo "======================================"
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/pi-usb-hid.sh
```

---

# PART 20 — Systemd Service

## Description

Systemd starts the HID gadget automatically after boot.

Create:

```bash
sudo nano /etc/systemd/system/pi-usb-hid.service
```

Put:

```ini
[Unit]
Description=Raspberry Pi USB HID Keyboard and Mouse

# ConfigFS must be available first.
After=sys-kernel-config.mount

# Request ConfigFS.
Wants=sys-kernel-config.mount

[Service]
Type=oneshot

# Create and enable the USB gadget.
ExecStart=/usr/local/bin/pi-usb-hid.sh

# Keep the service marked as active.
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable at boot:

```bash
sudo systemctl enable pi-usb-hid.service
```

Start now:

```bash
sudo systemctl start pi-usb-hid.service
```

Check:

```bash
sudo systemctl status pi-usb-hid.service --no-pager
```

---

# PART 21 — Reboot Test

Reboot:

```bash
sudo reboot
```

After reboot:

```bash
ls /sys/class/udc/
```

Expected:

```text
fe980000.usb
```

Check HID:

```bash
ls -l /dev/hidg*
```

Expected:

```text
/dev/hidg0
/dev/hidg1
```

Check service:

```bash
systemctl status pi-usb-hid.service --no-pager
```

Expected:

```text
Active: active (exited)
```

---

# PART 22 — Python Remote Control

## Description

The final remote-control system can use the following architecture:

```text
                 Network
 PC running Python ───────────────── Raspberry Pi
       │                                  │
       │                                  ├── Camera Module 3
       │                                  │
       │                                  ├── /dev/hidg0
       │                                  │      Keyboard
       │                                  │
       │                                  └── /dev/hidg1
       │                                         Mouse
       │
       └── Camera View
```

The PC sends keyboard and mouse commands over the network.

The Raspberry Pi writes those commands to:

```text
/dev/hidg0
/dev/hidg1
```

The Pi camera provides the video stream.

---

# PART 23 — Important Python File Naming

Do NOT name the Python program:

```text
socket.py
```

Do NOT create files named:

```text
socket.py
threading.py
json.py
time.py
cv2.py
```

These names can override Python's standard modules.

Recommended:

```text
remote_control.py
```

For example:

```text
/home/puppy/remote_control.py
```

Run:

```bash
python3 /home/puppy/remote_control.py
```

---

# PART 24 — Python Socket Check

Before running the remote-control program:

```bash
python3 -c "import socket; print(socket.__file__); print(socket.AF_INET)"
```

The output should point to Python's standard library.

For example:

```text
/usr/lib/python3.13/socket.py
2
```

If it points to your own project directory, rename the conflicting file.

---

# PART 25 — Troubleshooting

## USB controller missing

```bash
ls -la /sys/class/udc/
```

If empty:

```bash
sudo modprobe dwc2
```

Then:

```bash
ls -la /sys/class/udc/
```

---

## HID devices missing

```bash
ls -l /dev/hidg*
```

Check:

```bash
cat /sys/kernel/config/usb_gadget/pi-hid/UDC
```

Check:

```bash
sudo dmesg | tail -100
```

---

## Service failure

```bash
sudo systemctl status pi-usb-hid.service --no-pager
```

View logs:

```bash
sudo journalctl \
    -u pi-usb-hid.service \
    -b \
    --no-pager
```

---

## Permission denied on /dev/hidg0

Check:

```bash
ls -l /dev/hidg*
```

Check groups:

```bash
groups
```

Check:

```bash
getent group hidusers
```

---

## Windows reports Code 10

Disconnect the USB cable.

On the Pi:

```bash
sudo dmesg | tail -100
```

Then reconnect the USB cable and run:

```bash
sudo dmesg | tail -100
```

The new USB messages are important for diagnosing enumeration problems.

---

# PART 26 — Final Result

After everything is working, the Raspberry Pi will provide:

```text
Raspberry Pi 4
│
├── Camera Module 3
│      └── 1080p video
│
└── USB-C
       │
       └── USB HID Gadget
              ├── Keyboard
              └── Mouse
```

The intended remote-control application is:

```text
┌─────────────────────────────────────────────┐
│              CONTROL PC                     │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │       Raspberry Pi Camera             │  │
│  │              1920x1080                │  │
│  │                                       │  │
│  │       Mouse control inside view       │  │
│  └───────────────────────────────────────┘  │
│                                             │
│       Keyboard → Raspberry Pi               │
│       Mouse    → Raspberry Pi               │
└───────────────────────┬─────────────────────┘
                        │ Network
                        ▼
                 ┌───────────────┐
                 │ Raspberry Pi  │
                 │               │
                 │ Camera → PC   │
                 │               │
                 │ HID Keyboard  │
                 │ HID Mouse     │
                 └───────┬───────┘
                         │ USB-C
                         ▼
                  Target Computer
```

# End
