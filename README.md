# LicheeRV nano (WiFi and Ethernet) based web radio / led matrix informer / CO2 monitor how to

1. Download Debian for SG2002

   https://github.com/Fishwaldo/sophgo-sg200x-debian

2. Flash it to micro SD card using BalenaEtcher

3. Boot board from SD card and connect to it via SSH as debian/rv
4. Change user to root/rv

sudo su -

5. Switch off RNDIS on Type-C USB

systemctl disable usb-gadget-rndis

6. Update system

apt update
apt upgrade

7. Install packages

apt install mpd mpc samba samba-common-bin htop alsa-utils nano wget

8. Fix MAC address

nano /etc/network/interfaces.d/end0

allow-hotplug end0
iface end0 inet dhcp
    hwaddress ether XX:XX:XX:XX:XX:XX

9. Configure mpd

systemctl enable mpd
systemctl start mpd

nano /etc/mpd.conf

replace 

music_directory         "/var/lib/mpd/music"

to

music_directory         "/srv/share/mpd/music"

replace

playlist_directory              "/var/lib/mpd/playlists"

to

playlist_directory              "/srv/share/mpd/playlists"


replace

bind_to_address                        "localhost"

to

bind_to_address                        "0.0.0.0"


add to the end

audio_output {
        type            "alsa"
        name            "USB Audio Adapter"
        device          "hw:2,0"
        mixer_device    "hw:2"
        mixer_type      "software"
        mixer_control   "PCM"
        mixer_index     "0"
}

10. Restart mpd

systemctl restart mpd

11. Check mpd

mpc add http://fhhalle.streamabc.net/fhhal-brocken90er-mp3-256-1012329
mpc play

12. Configure Samba

useradd samba
mkdir /srv/share
chown samba:samba /srv/share

Edit /etc/samba/smb.conf

nano /etc/samba/smb.conf

comment blocks

[printers]
[print$]


add to the end

[share]

path = /srv/share
guest ok = yes
writeable = yes
force user = samba

13. Restart samba

systemctl reload smb

14. Create folders for mpd

mkdir /srv/share/mpd
mkdir /srv/share/mpd/music
mkdir /srv/share/mpd/playlists

chmod -R 777 /srv/share/mpd

15. Update mpd database
    
mpc update

16. Install PM2

apt install nodejs npm
npm i -g pm2

pm2 -v
5.4.3

17. Install MPD web UI
    
git clone https://github.com/ondras/cyp.git && cd cyp
npm i

nano index.js

replace

let httpServer = require("http").createServer(onRequest).listen(port);

to

let httpServer = require("http").createServer(onRequest).listen(port, '0.0.0.0');

pm2 start index.js

[PM2] Starting /home/debian/cyp/index.js in fork_mode (1 instance)
[PM2] Done.
┌────┬──────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id │ name     │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├────┼──────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0  │ index    │ default     │ 1.0.0   │ fork    │ 3097     │ 1s     │ 0    │ online    │ 0%       │ 38.8mb   │ debian   │ disabled │
└────┴──────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘

pm2 save

[PM2] Saving current process list...
[PM2] Successfully saved in /home/debian/.pm2/dump.pm2

pm2 startup
[PM2] Init System found: systemd
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:/usr/bin /usr/local/lib/node_modules/pm2/bin/pm2 startup systemd -u debian --hp /home/debian


sudo env PATH=$PATH:/usr/bin /usr/local/lib/node_modules/pm2/bin/pm2 startup systemd -u debian --hp /home/debian

pm2 list

┌────┬──────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id │ name     │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├────┼──────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0  │ index    │ default     │ 1.0.0   │ fork    │ 3097     │ 3m     │ 0    │ online    │ 0%       │ 45.1mb   │ debian   │ disabled │
└────┴──────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘
