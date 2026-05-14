# RazerBlade 14 (2023) General Fedora (44) Fixes

Fedora (44) Installation Guide (Minimal Setup)
- Update packages
  ```bash
  sudo dnf update
  ```
- Fix critical issues: _BIOS_, _Speaker_, _System Freeze_, _Hibernation_ etc.
- Install nvidia driver
  ```bash
  sudo dnf install kernel-devel-$(uname -r)
  sudo dnf install kernel-headers gcc make
  sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda
  ```
  **Manually fix GRUB config**: add back escape characters -> regen config<br><br>
  (It'd be the best to switch to TTY to compile the driver)<br><br>
  **Switch to TTY**: reboot -> press `e` in the boot menu -> append 
  `system.unit=multi-user.target` to CMD Linux -> `Ctrl x` to boot
  ```bash
  sudo akmods --force
  sudo dracut --force
  reboot
  ```
  Boot the normal graphics environment. Check installation
  ```bash
  nvidia-smi
  ```

## BIOS
- Enable Dedicated GPU Only
- Disable Secure Boot
- Disable Fast Boot

## System Freeze
<details>
<summary>Click to Expand</summary>

- **Issue**: Random system pauses/freezes
- **Fix**: Modify GRUB config to pretend to be Windows (RazerBlade motherboard specializes on Windows)

```
sudo nano /etc/default/grub
```

Append the parameters below to `GRUB_CMDLINE_LINUX="..."`
```
acpi_osi=! acpi_osi=\"Windows 2020\"
```

**Optional**:
- Add `processor.max_cstate=5` (force performance mode)
- Add `pcie_aspm=off`

Finally, run `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` to regenerate config.

> Note:
> `sudo dnf install akmod-nvidia` will modify GRUB config and remove `\` from `\"Windows 2020\"`. Requires a manual fix here.

</details>

## Hibernation
<details>
<summary>Click to Expand</summary>

- **Issue**: Hibernation/Suspend causes system freeze and occasional black screen
- **Fix**: Totally disable Hibernation/Suspend

Goto `Gnome Settings` -> Power -> Disable Automatic Suspend

```bash
gsettings set org.gnome.desktop.session idle-delay 0
```

Disable hibernation related services (use `unmask` instead of `mask` to revert these changes)
```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Disable lid close suspend
```bash
sudo nano /etc/systemd/logind.conf
```
```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
```

Add GRUB confid parameters (regen config afterward!):
- `nvme_core.default_ps_max_latency_us=0` (disable SSD powersafe)

```bash
reboot
```

</details>

## Speaker
<details>
<summary>Click to Expand</summary>

- **Issue**: Built-in speaker not working
- **Fix**: [See `rb14-2023-audio-fix` for details](https://github.com/yadu-tv/rb14-2023-audio-fix/) (licensed under MIT)

```bash
sudo dnf install alsa-tools
```
Download `rb_audio.sh` and grant access
```bash
sudo chmod a+x ~/Downloads/rb_audio.sh
```
```bash
sudo mv ~/Downloads/rb_audio.sh /usr/bin/
```
Create the startup service
```bash
sudo nano /etc/systemd/system/rb_audio.service
```
```
[Unit]
Description=Razer Blade 14 audio fix 

[Service]
ExecStart=/bin/bash /usr/bin/rb_audio.sh

[Install]
WantedBy=multi-user.target 
```
```bash
sudo systemctl enable rb_audio.service
```
```bash
reboot
```

</details>

## Wired Earphones
<details>
<summary>Click to Expand</summary>

- **Issue**: The system occasionally not recognizing the wired earphones
- **Fix**:
  ```bash
  systemctl --user restart wireplumber
  systemctl --user restart pipewire
  ```

</details>

## WiFi
<details>
<summary>Click to Expand (Ignore This Part If WiFi Is Working Properly)</summary>

- **Issue**: Qualcomm WiFi6 not working well with Linux + RX aggregation issue (randomly experiencing low speed WiFi)
- **Fix**: Disable WiFi6 (ax) + Auto WiFi reconnect on startup 

### Service: `fix-wifi.service`

Create `fix-wifi.sh`
```bash
sudo nano /usr/bin/fix-wifi.sh
```

Write `fix-wifi.sh`
```bash
#!/bin/bash

modprobe -r ath11k_pci
modprobe ath11k_pci disable_11ax=1
```

> Note: `Ctrl O` → `Enter` (Save File) → `Ctrl X` (Exit)

Grant access
```bash
sudo chmod a+x /usr/bin/fix-wifi.sh
```

Create `fix-wifi.service`
```bash
sudo nano /etc/systemd/system/fix-wifi.service
```

Write `fix-wifi.service`
```
[Unit]
Description=Fix ath11k WiFi (disable ax)
After=NetworkManager.service
Wants=NetworkManager.service

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/bin/fix-wifi.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable the service
```bash
sudo systemctl enable fix-wifi.service
```

Reboot to check changes
```bash
reboot
```

Run `sudo systemctl status fix-wifi.service` to check info. Expected to see `Active: active (exited)`

Try WiFi speedtest to see if disabling WiFi6 straight up fixes the issue.

> The fix is essentially these two lines
> ```bash
> modprobe -r ath11k_pci
> modprobe ath11k_pci disable_11ax=1
> ```

### Service: `wifi-reconnect.service`

**If The Issue Persists (presumably occasionally getting low speed WiFi after reboot)**: That most likely leads to a RX aggregation issue. The fix is to try auto reconnect once to reset states.

Add a dispatcher script `99-fix-wifi`
```bash
sudo nano /etc/NetworkManager/dispatcher.d/99-fix-wifi
```

Write `99-fix-wifi`
```bash
#!/bin/bash

IFACE="$1"
STATUS="$2"

MARKER="/tmp/fix-wifi-done"

if [ "$IFACE" = "wlp3s0" ] && [ "$STATUS" = "up" ]; then

    if [ -f "$MARKER" ]; then
        exit 0
    fi

    touch "$MARKER"

    /usr/bin/systemctl start wifi-reconnect.service
fi
```

Note that you probably don't want to use `wlp3s0` as well.
Check `iw dev` and find the section below
```
Interface wlp3s0
		ifindex 3
		wdev 0x100000001
```
Interface code similar to **`wlp3s0`** is what you want.

This dispatcher script runs a service `wifi-reconnect.service` when connecting to WiFi during startup. You need to create the service as well.

Add `wifi-reconnect.service`
```bash
sudo nano /etc/systemd/system/wifi-reconnect.service
```

Write `wifi-reconnect.service`
```
[Unit]
Description=Reconnect WiFi

[Service]
Type=oneshot
ExecStart=/usr/bin/bash -c 'sleep 2 && /usr/bin/nmcli dev disconnect wlp3s0 && sleep 1 && /usr/bin/nmcli dev connect wlp3s0'
```

It waits 2 seconds, disconnects `wlp3s0`, waits 1 second, and then connects `wlp3s0` (Note: Use your own interface code)

Grant access
```bash
sudo chmod a+x /etc/NetworkManager/dispatcher.d/99-fix-wifi
```

Reboot to test WiFi
```
reboot
```

### Debug Your Services
Clear journal first
```bash
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s
```

Read service journal
```bash
journalctl -u fix-wifi.service
```
```bash
journalctl -u wifi-reconnect.service
```

***

### Optional: Performance Mode

Enforce performance mode to ensure WiFi speed

```bash
sudo iw dev wlp3s0 set power_save off
```

```bash
sudo nano /etc/NetworkManager/conf.d/wifi-powersave.conf
```

```
[connection]
wifi.powersave = 2
```

### Optional: Timer Based WiFi6 (ax) Disable Trigger

If the startup script didn't last long. Try adding a timer based script to disable WiFi6 periodically, preventing potential firmware reset.

Be sure you run `chmod a+x /usr/bin/fix-wifi.sh` to grant access permission first.

```bash
sudo nano /etc/systemd/system/fix-wifi.timer
```

```
[Unit]
Description=Run fix-wifi periodically

[Timer]
OnBootSec=15
OnUnitActiveSec=30
Unit=fix-wifi.service

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now fix-wifi.timer
```

Check `sudo systemctl status fix-wifi.timer`. Expected to see `Active: active (running)`

</details>

## Touchpad Scroll Sensitivity
<details>
<summary>Click to Expand</summary>

- **Issue**: Touchpad scrolling speed is too high
- **Fix**: [See `touchpad-sensitivity-tweak` for details](https://github.com/shivasai573/touchpad-sensitivity-tweak) (licensed under MIT)

```bash
git clone https://github.com/shivasai573/touchpad-sensitivity-tweak.git
cd touchpad-sensitivity-tweak
```
```bash
chmod +x scripts/install_fix.sh
```
Be sure that `touchegg` is not installed
```bash
sudo dnf remove touchegg
```
```bash
sudo ./scripts/install_fix.sh
```
```bash
reboot
```

</details>
