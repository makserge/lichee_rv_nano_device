# LicheeRV Nano Ethernet-Based Web Radio Setup Guide

This guide details how to configure a Debian-based internet radio with a Samba media share and a lightweight web control interface on the LicheeRV Nano (SG2002).

## 1. Preparation & OS Flashing

1. Download the Debian system image (`licheervnano_sd.img.lz4`) from the [Fishwaldo sophgo-sg200x-debian Repository](https://github.com).
2. Flash the downloaded image to a Micro SD card using [BalenaEtcher](https://balena.io).
3. Insert the card into your LicheeRV Nano, boot up, and connect via SSH:
   * **Username:** `debian`
   * **Password:** `rv`

---

## 2. Base System Configuration

Elevate your user shell to root and disable the default USB network sharing to clean up your network interfaces:

```bash
sudo su -
systemctl disable usb-gadget-rndis
apt update && apt upgrade -y
```

Install the essential multimedia, network file system, and package tools:

```bash
apt install mpd mpc samba samba-common-bin htop alsa-utils nano wget -y
```

Fix your Ethernet MAC address to ensure persistent IP assignment across system reboots:

```bash
nano /etc/network/interfaces.d/end0
```

Add the following block to the network definition file:

```text
allow-hotplug end0
iface end0 inet dhcp
    hwaddress ether XX:XX:XX:XX:XX:XX
```

---

## 3. MPD Audio Setup

Enable the Music Player Daemon background service:

```bash
systemctl enable mpd
systemctl start mpd
nano /etc/mpd.conf
```

Within the file, locate and update the following variables to route audio to your custom directories and accept remote connections:

* Find and change `music_directory "/var/lib/mpd/music"` to:
  ```text
  music_directory         "/srv/share/mpd/music"
  ```
* Find and change `playlist_directory "/var/lib/mpd/playlists"` to:
  ```text
  playlist_directory              "/srv/share/mpd/playlists"
  ```
* Find and change `bind_to_address "localhost"` to:
  ```text
  bind_to_address                        "0.0.0.0"
  ```

Append your hardware ALSA audio profile to the very bottom of the document:

```text
audio_output {
        type            "alsa"
        name            "USB Audio Adapter"
        device          "hw:2,0"
        mixer_device    "hw:2"
        mixer_type      "software"
        mixer_control   "PCM"
        mixer_index     "0"
}
```

Apply settings and verify audio streaming functionality:

```bash
systemctl restart mpd
mpc add http://streamabc.net
mpc play
```

---

## 4. Samba Share Integration

Create a system user for file transfers and allocate system directories:

```bash
useradd samba
mkdir /srv/share
chown samba:samba /srv/share
nano /etc/samba/smb.conf
```

Adjust the file share behavior by commenting out configurations and appending global overrides:

* Comment out the blocks starting with `[homes]`, `[printers]` and `[print$]` by adding a `#` character to the front of those lines.
* In the `[global]` block, immediately underneath the line `usershare allow guests = yes`, add:
  ```text
  path = /srv/share
writeable = yes
guest ok = yes
force user = samba
  ```

Append your network drive definition to the bottom of the config:

```text
[share]
path = /srv/share
writeable = yes
force user = samba
```

Apply modifications and build the music folders:

```bash
systemctl reload smb
mkdir -p /srv/share/mpd/music /srv/share/mpd/playlists
chmod -R 777 /srv/share/mpd
mpc update
```

---

## 5. Web UI Deployment (CYP)

Install the Node.js framework and global Process Manager (PM2):

```bash
apt install nodejs npm -y
npm i -g pm2
```

Clone the https://github.com/makserge/cyp and compile the application bindings:

```bash
git clone https://github.com/makserge/cyp && cd cyp
npm i
nano index.js
```
Launch the daemon and bind the instance to system startup hooks:

```bash
pm2 start index.js
pm2 save
pm2 startup

sudo visudo -f /etc/sudoers.d/cyp
debian ALL=(ALL) NOPASSWD: /usr/sbin/shutdown, /usr/bin/mount
```

*Note: Copy and execute the exact environment execution string provided by the `pm2 startup` terminal printout to establish systemd persistence.*

---

## 6. Storage Expansion & Equalizer Control

Create mounting points to connect your external USB storage device and your
second SD card:

```bash
mkdir -p /media/usb /media/sdcard
nano /etc/fstab
```

Append the auto-mount entries at the bottom of the table layout to link your
cards automatically with explicit write permissions for the Samba user:

```text
/dev/sda1       /media/usb      auto    nosuid,nodev,nofail,uid=1001,gid=1001,umask=000  0  0
/dev/mmcblk1p1  /media/sdcard   auto    nosuid,nodev,nofail,uid=1001,gid=1001,umask=000  0  0
```

Apply the new mount settings immediately and adjust the native directory paths
to allow full read, write, and execute permissions:

```bash
mount -a
```

Link your external physical drive and secondary card directories directly into
the target MPD multimedia repository:

```bash
ln -s /media/usb /srv/share/mpd/music/usb
ln -s /media/sdcard /srv/share/mpd/music/sdcard
```

Install the ALSA processing plugins and set up a system-wide software
equalizer plugin layer:

```bash
apt install libasound2-plugin-equal caps -y
nano /etc/asound.conf
```

Add your operational audio mixing routing configurations:

```text
pcm.my_equalizer {
    type equal
    slave.pcm "plughw:2,0" 
}

ctl.equal {
    type equal
}

pcm.equalplug {
    type plug
    slave.pcm "my_equalizer"
}

pcm.!default {
    type plug
    slave.pcm "equalplug"
}
```

Open up your audio service setup to route output through your virtual channel:

```bash
nano /etc/mpd.conf
```

Modify the pre-existing `audio_output` configuration properties:

```text
audio_output {
    type            "alsa"
    name            "PCM2704 USB Equalizer"
    device          "equalplug"
    mixer_type      "software"
}
```

Expose the configuration file binary path permissions directly to the running
audio daemon instance:

```bash
rm -f ~/.alsaequal.bin
touch /var/lib/mpd/.alsaequal.bin
chown mpd:audio /var/lib/mpd/.alsaequal.bin
chmod 664 /var/lib/mpd/.alsaequal.bin
systemctl restart mpd
mpc update
```

You can now adjust your frequency bands live on your terminal at any time:

```bash
alsamixer -D equal
```

