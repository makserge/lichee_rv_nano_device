LicheeRV nano (WiFi and Ethernet) based web radio / led matrix informer / CO2 monitor how to

1. Download Debian for SG2002

   https://github.com/Fishwaldo/sophgo-sg200x-debian

2. Flash it to micro SD card using BalenaEtcher

3. Boot board from SD card and connect to it via SSH as debian/rv
4. Change user to root/rv
5. Switch off RNDIS on Type-C USB

systemctl disable usb-gadget-rndis

6. Update system

apt update

7. Install packages

apt install mpd mpc    
