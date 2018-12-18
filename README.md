linux-on-huawei-matebook-d-14

# GNU/Linux on MateBook D 14" (AMD Ryzen 5 2500U model KPL-W0X in 2018)

> Brain dump: MateBook D running Manjaro Linux

![](https://consumer-img.huawei.com/content/dam/huawei-cbg-site/common/mkt/pdp/tablets/matebook-d-14-k/img/huawei_matebook_d_kv-original.jpg)

## Background

First Huawei MateBook D 14 AMD was released in 2018.
It came with proprietary Microsoft Windows 10 and there was very little information available on its Linux support. 

I am running Manjaro on it. This repository documents what works and what does not.

## Linux Support Matrix

| Device | Model |  Works | Notes |
| --- | --- |  :---: | --- |
| Processor | AMD Ryzen 5 2500U | 💚 Yes | (TODO: document)  |
| Graphics | AMD Raven Ridge [Radeon Vega Series / Radeon Vega Mobile Series] |  💚 Yes | via AMDGPU (TODO: document) |
| Memory | 7098 MB |  💚 Yes | Listed usable, hardware is 8GB |
| Display | 14 inch 16:9, 1920x1080 (FHD) | 💚 Yes | resolution is correctly detected by `xrandr`, backlight control does not work via native function keys, but works via additional scripts (see [Display Backlight](#display-backlight)) |
| Storage | SanDisk SD9SN8W256G1027, 256 GB | 💚 Yes | via standard kernel driver (TODO: document) |
| Soundcard  | Advanced Micro Devices, Inc. [AMD/ATI] | 💚 Yes  | via standard kernel driver, only tested with `pulseaudio` (TODO: document) |
| Speakers  | "Dolby ATMOS" | 💚 Yes | Seems fine, but levels may need adjusting (TODO: document) | 
| Ports | 1 USB 3.0/3.1 Type-C, 1 USB 3.0/3.1, 1 USB 2.0, HDMI |  💚 Yes | USB-C tested only for charging | 
| Wifi | Intel Dual Band Wireless-AC 8265 (a/b/g/n/ac) | 💚 Yes | (TODO: document) | 
| Bluetooth | Intel (idVendor:0x8087, idProduct:0x0a2b) | 💚 Yes | (TODO: document) |
| Airplane Mode | Wifi+Bluetooth | 💚 Yes | ([see details below](#airplane-mode)) |
| Battery | 40 Wh Lithium-Polymer | 💚 Yes | Everything works: current status, chargin/discharging rate and remaining battery time estimates |
| Lid | ACPI-compliant |  💚 Yes | Works as expected: I can just close the lid and it sleeps  |
| Keyboard |  | 💚 Yes | Everything works out of the box, including function keys and backlight | 
| Touchpad | | 💚 Yes | Tap-to-click can be enabled via `libinput` ([see details below](#touchpad)) |
| Touchscreen | | 💚 Yes | (TODO: document) |

## GRUB

`/etc/default/grub`
```
GRUB_CMDLINE_LINUX="vga=current ivrs_ioapic[4]=00:14.0 ivrs_ioapic[5]=00:00.2 idle=nomwait acpi_osi=! acpi_osi='Windows 2015' acpi_enforce_resources=lax scsi_mod.use_blk_mq=1
```

## Touchpad

Because of compatability with some applications (including 'gravity' scrolling in browsers) I have been using `xf86-input-synaptics`.

**There is a bug** that appears intermittently. Touchpad can become lost and unresponsive after wake from suspend.

A workaround is to find the device name and restart it. You can do this manually or automatically. Solutions can be combined. 

#### Find device and create script

```
$ xinput
⎡ Virtual core pointer                          id=2    [master pointer  (3)]
⎜   ↳ Virtual core XTEST pointer                id=4    [slave  pointer  (2)]
⎜   ↳ Wacom HID 48CF Finger                     id=11   [slave  pointer  (2)]
⎜   ↳ ETPS/2 Elantech Touchpad                  id=14   [slave  pointer  (2)]
⎜   ↳ ELAN2203:00 04F3:309A Touchpad            id=12   [slave  pointer  (2)]
```
We want the `ELANXXX:XX XXXX:XXXX` device. Copy that value.
Now create file at `~/.local/bin/restart-touchpad.sh` with the following content (replacing device id and %USER with your username):
```
#!/bin/bash
echo "Restarting Touchpad just in case"
declare -x DISPLAY=":0.0"
declare -x XAUTHORITY="/home/%USER/.Xauthority"
xinput disable 'ELAN2203:00 04F3:309A Touchpad'
xinput enable 'ELAN2203:00 04F3:309A Touchpad'
```

##### Manual

Asign a keyboard shortcut to the script or similar

##### Automatic

Make a systemd service by creating file `/etc/systemd/system/restart-touchpad.service` with the following content (replacing %USER with your username):
```
[Unit]
Description=Restart Touchpad
After=basic.target suspend.target hibernate.target

[Service]
Type=oneshot
Environment=DISPLAY=:0
ExecStart=/home/%USER/.local/bin/restart-touchpad.sh

[Install]
WantedBy=basic.target suspend.target hibernate.target
```
Then enable it:
```
systemctl enable restart-touchpad.service --now
```

## Power Management

Been testing it with Manjaro Unstable + kernel 4.19.8-2. I'm using TLP and KDE.  With this setup and my workflow (mostly browser + media) the battery lasts for around 6-7 hours. Switching to more intensive things like code compilation or video encoding for prolonged time cuts battery to around 4-5 hours. 

#### TLP

> I use `tlp` for power management.
```
sudo pacman -S --needed tlp tlp-rdw iw smartmontools ethtool x86_energy_perf_policy
systemctl mask systemd-rfkill.service
systemctl mask systemd-rfkill.socket
systemctl enable tlp.service
systemctl enable tlp-sleep.service
```

#### lm_sensors

```
pacman -S lm_sensors && sudo sensors-detect
```

## Airplane Mode

Works as expected.

`rfkill list` shows current status of radio interfaces:

```
 rfkill list                                                                      ~
0: phy0: Wireless LAN
	Soft blocked: no
	Hard blocked: no
11: hci0: Bluetooth
	Soft blocked: no
	Hard blocked: no
```

Toggling "airplane mode": `rfkill block all` / `rfkill unblock all`
