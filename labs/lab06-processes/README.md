# Lab 6 — Live processes and memory pressure

Author: Aisulu 
Machine: ASUS ROG Strix G16 (Windows 11, 16 GB RAM), VirtualBox
Guests / Environment: Ubuntu Server 26.04.1 LTS (`2 vCPU`, RAM `3.3 GiB`, swap `2.0 GiB` `/swap.img`) · Windows 11 (`DESKTOP-PN2SJNK`)
Date: 23.11.2026 (times in the Ubuntu logs are guest time, UTC)

## Part 1 — One process, three tools

Process chosen: `ping 127.0.0.1` (`PING.EXE -t 127.0.0.1` on Windows, `ping 127.0.0.1` on Ubuntu) — why: it is a standard built-in utility available out of the box on both systems without extra dependencies, and it runs continuously until stopped, making it easy to inspect across Task Manager, Process Explorer, and `htop` / `ps`.

Windows: `PING.EXE` PID `4796`, parent `cmd.exe` (`8376`). Ubuntu guest: `ping 127.0.0.1` PID `1739`, parent `bash` (`1543`).

Commands:

```text
# Windows (PowerShell)
Get-CimInstance Win32_Process -Filter 'ProcessId=4796' | Select-Object ProcessId, ParentProcessId, CommandLine | Format-List
Get-Process -Id 8376 | Select-Object Id, ProcessName

# Ubuntu
ping 127.0.0.1 > /dev/null &
ps -o pid,ppid,user,rss,vsz,nlwp,cmd -p 1739
ls /proc/1739/fd | wc -l
sudo ls -l /proc/1739/fd | head
ps -p $(ps -o ppid= -p 1739) -o pid,comm
