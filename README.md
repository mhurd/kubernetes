# kubernetes

These are my notes and relevant yaml files for the setup of my Raspberry Pi cluster (4x Raspberry Pi 3 Model B's).

## Raspberry Pi Cluster shopping list

* Multi-Pi Stackable Raspberry Pi Case (x2)
[link](https://www.modmypi.com/raspberry-pi/cases-183/raspberry-pi-b-plus2-and-3-cases-1122/stacking-cases-1132/multi-pi-stackable-raspberry-pi-case/?search=stackab) 

* Raspberry Pi 3 Model B Quad Core CPU 1.2 GHz 1 GB RAM Motherboard (x4)
[link](https://www.amazon.co.uk/gp/product/B01CD5VC92/ref=od_aui_detailpages00?ie=UTF8&psc=1)

* SanDisk Ultra 32GB micro SDHC Memory Card + SD Adapter with A1 App Performance up to 98MB/s, Class 10, U1 - FFP (x3)
[link](https://www.amazon.co.uk/gp/product/B073S8LQSL/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)

* SanDisk Ultra 64GB micro SDXC Memory Card + SD Adapter with A1 App Performance up to 100MB/s, Class 10, U1 - FFP (x1)
[link](https://www.amazon.co.uk/gp/product/B073SB2L3C/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)

* Adaptare 40545 Charging Cable USB Type A Male to DC Barrel Jack (5.5 x 2.5 mm – 60 cm) (x1)
[link](https://www.amazon.co.uk/gp/product/B01BHEAM9Q/ref=oh_aui_detailpage_o01_s00?ie=UTF8&psc=1)

* VELCRO Brand Stick On Tape - 20 mm x 2.5 m, Black (x1)
[link](https://www.amazon.co.uk/gp/product/B0013D8IUC/ref=oh_aui_detailpage_o02_s00?ie=UTF8&psc=1)

* Gorilla Pi Heat-sink For Raspberry Pi 3 & Pi 2 Model B. 3Pc Set (x1 copper x2 aluminium) With Pre Installed Heat-sink Adhesive (x4)
[link](https://www.amazon.co.uk/gp/product/B01FY9196K/ref=oh_aui_detailpage_o03_s00?ie=UTF8&psc=1)

* NETGEAR GS305-100UKS 5-Port Gigabit Metal Ethernet Desktop/Wall-mount Switch (x1)
[link](https://www.amazon.co.uk/gp/product/B00TV4A96Q/ref=oh_aui_detailpage_o04_s00?ie=UTF8&psc=1)

* Anker USB Charger Power-Port 5 (40W 5-Port USB Charging Hub) Multi-Port Wall Charger (x1)
[link](https://www.amazon.co.uk/gp/product/B00VTI8K9K/ref=oh_aui_detailpage_o04_s00?ie=UTF8&psc=1)

* Micro USB Cable Ulreon 4 Pack Short 12 inch/0.3m High Speed Quick Charging Sync 2.4A Ultra Strong Durable Nylon Braided Cord (x4)
[link](https://www.amazon.co.uk/gp/product/B074DVPLQP/ref=oh_aui_detailpage_o05_s00?ie=UTF8&psc=1)

* Ethernet Cable, Rankie 5-Pack 0.3m RJ45 Cat 6 Ethernet Patch LAN Network Cable (5-Color Combo) - R1300A (x1)
[link](https://www.amazon.co.uk/gp/product/B01J8KFTB2/ref=oh_aui_detailpage_o05_s00?ie=UTF8&psc=1)

## OS
Use the latest Raspian Stretch Lite img [from here](https://downloads.raspberrypi.org/raspbian_lite_latest)

## Docker installation
Don't use apt-get! The commands below assume that your Raspian user is called *pi*
```bash
sudo curl -sSL https://get.docker.com | sh
sudo usermod -aG docker pi
```
