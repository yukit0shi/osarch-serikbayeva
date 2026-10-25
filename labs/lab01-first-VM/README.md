# Lab 1 — Your first virtual machine

Driver first half: Aisulu Serikbayeva
Machine: Maibenben X16C, Windows 11, guest Ubuntu Server 26.04.1 LTS 
Date: 25.10.2026

## What I did
Installed Oracle VM Virtualbox. Version:7.2.22 r175245 (Qt6.8.0 on windows) . Licence: https://www.virtualbox.org/wiki/VirtualBox_PUEL
Created VM lab01-ubuntu: 2 vCPU, 4 GB RAM, 25 GB dynamic disk, NAT.
Installed Ubuntu Server, took snapshot clean-install.
Broke it: sudo rm /etc/hostname
Rolled back to clean-install: cat /etc/hostname

## Resource justification
I allocated 4 GB of RAM and 2 vCPUs from my host's 16 GB total. Windows uses about 5-6 GB when idle, so leaving the rest for the host ensures my gaming laptop remains fully responsive. 4 GB is plenty for a headless Ubuntu Server and leaves room to build a second Windows guest concurrently in Lab 2.

## Result
The VM boots; the break-and-rollback cycle is shown in the evidence below.

## What did not work the first time
During the initial VM creation, VirtualBox threw a `VERR_FILE_NOT_FOUND` error because it failed to attach the virtual hard disk. I fixed this by deleting the broken VM instance, recreating it from scratch, and ensuring "Skip Unattended Installation" was checked.

## Evidence
- [evidence/vm-settings.png](evidence/vm-settings.png) — CPU, memory and disk as configured
- [evidence/guest-release.png](evidence/guest-release.png) — Ubuntu Server is running in the guest
- [evidence/broken.png](evidence/broken.png) — the guest in its broken state
- [evidence/snapshot-list.png](evidence/snapshot-list.png) — the snapshot, after the rollback
