# Lab 0 Inventory your own machine
Name: Aisulu 
Machine: Laptop Maibenben X16C (AMD Ryzen 7 H 255), Windows 11
Date: 22.11.2026

## What I did
```powershell
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors, VirtualizationFirmwareEnabled
systeminfo | Select-String "Hyper-V"

Get-CimInstance Win32_PhysicalMemory | Select-Object DeviceLocator, Capacity, Speed
Get-CimInstance Win32_ComputerSystem | Select-Object TotalPhysicalMemory

Get-PhysicalDisk | Select-Object FriendlyName, MediaType, BusType, Size
Get-Volume | Select-Object DriveLetter, SizeRemaining, Size

$env:firmware_type
Get-CimInstance Win32_BIOS | Select-Object SMBIOSBIOSVersion, ReleaseDate
```

## Result
| What | Value | Where I got it |
| --- | --- | --- |
| CPU model | AMD Ryzen 7 H 255 W/ Radeon 780M | `Get-CimInstance Win32_Processor` |
| Core / Thread count | 8 cores / 16 threads | `Get-CimInstance Win32_Processor` |
| Total memory | 16 GB installed (15.26 GiB usable) | `Get-CimInstance Win32_ComputerSystem` |
| Memory modules & speed | 1 module (`DIMM 0`, 16 GB), 5600 MT/s | `Get-CimInstance Win32_PhysicalMemory` |
| Disk model & type | YMTC PC42Q-512GB-B, SSD (NVMe) | `Get-PhysicalDisk` |
| Free space on VM volume | 205.4 GiB (220.6 GB) free on `C:\` | `Get-Volume` |
| Firmware type | UEFI | `$env:firmware_type` |
| Firmware version & date | N.1.10MBB10, 10.03.2026 | `Get-CimInstance Win32_BIOS` |
| Hardware virtualization | Present and Enabled (`True`) | `Get-CimInstance Win32_Processor`, `systeminfo` |

## What did not work the first time
When I ran `systeminfo | Select-String "Hyper-V"` to check virtualization support, Windows did not display the four individual Hyper-V requirement lines and instead returned `A hypervisor has been detected. Features required for Hyper-V will not be displayed.` because the Windows hypervisor was already running. To confirm that virtualization is enabled in firmware, I queried `VirtualizationFirmwareEnabled` via `Get-CimInstance Win32_Processor`, which returned `True`.

## Evidence
- [evidence/cpu.txt](evidence/cpu.txt) raw output of CPU specs and virtualization status
- [evidence/memory.txt](evidence/memory.txt) raw output of RAM modules, speed, and total memory
- [evidence/disk.txt](evidence/disk.txt) raw output of physical disk model/type and free space on drive C
- [evidence/firmware.txt](evidence/firmware.txt) raw output of UEFI mode, BIOS version, and release date
