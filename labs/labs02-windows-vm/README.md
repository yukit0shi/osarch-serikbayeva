# Lab 2 — A Windows guest, and a network between them

Pair: Aisulu  Driver first half: Aisulu
Machine: <Maibenben X16C, Windows 11, 16 GB RAM, 8 cores> guests Ubuntu Server + Windows 11 Enterprise eval
Date: <01.11.2026>


## What I did
1. Downloaded Windows 11 Enterprise evaluation. Licence: https://www.microsoft.com/en-us/evalcenter/download-windows-11-enterprise
2. Created lab02-windows: 2 vCPU, 4 GB, 64 GB thin, UEFI + Secure Boot, vTPM.
3. Network mode: NAT Network, both guests. Why: I used NAT Network because both guests must see each other and the internet, and bridged mode is blocked on campus Wi-Fi.
4. Ping Ubuntu → Windows timed out because Windows Defender Firewall blocks incoming ICMP (ping) requests by default. Fixed with: `New-NetFirewallRule -Name "Allow_Ping" -DisplayName "Allow Ping ICMPv4" -Protocol ICMPv4 -IcmpType 8 -Action Allow`
5. Overcommitment: set 2 GB + 4 GB on a 16 GB host.


## Network diagram
Internet
           |
      Host laptop (Wi-Fi)
           |
    [VirtualBox NAT Network]
     /                  \
lab01-ubuntu       lab02-windows
10.0.2.15          10.0.2.3

## Result
| | Before | After |
|---|---|---|
| Host RAM in use |7.8 GB |12.7 GB |
| Available |7.5 GB |2.6 GB |
| Committed / Swap used |13.6/19.5 GB |34.6/36.2 GB |
| What I noticed | — |The host RAM in use increased significantly (from 7.8 GB to 12.7 GB) after starting both virtual machines, reflecting the overcommitted memory setup. |

## What did not work the first time
When pinging the internet (8.8.8.8) from Ubuntu for the first time, I saw a 50% packet loss. It turned out to be the VirtualBox NAT router's initial delay in processing ICMP packets. I fixed it by running the ping command again to Cloudflare's DNS (1.1.1.1), which resulted in a clean 0% packet loss.

## Evidence
- [evidence/ipconfig-windows.txt](evidence/ipconfig-windows.txt) — Windows address
- [evidence/ip-addr-linux.txt](evidence/ip-addr-linux.txt) — Ubuntu address
- [evidence/ping-both-ways.txt](evidence/ping-both-ways.txt) — both directions and the internet
- [evidence/host-memory-before.png](evidence/host-memory-before.png) — host memory, guests off
- [evidence/host-memory-after.png](evidence/host-memory-after.png) — host memory, both guests overcommitted
