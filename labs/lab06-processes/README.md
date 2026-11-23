# Lab 6 — Live processes and memory pressure

Pair: Aisulu (solo) · Date: 23.11.2026 · Guests: Ubuntu Server 26.04.1 LTS (3.3 GiB RAM, 2.0 GiB swap) / Windows 11 (16 GB RAM)

## Part 1 — One process, three tools

Process chosen: `ping 127.0.0.1` (`PING.EXE -t 127.0.0.1` on Windows, `ping 127.0.0.1` on Ubuntu) — why: it is a lightweight built-in network utility preinstalled on both systems that runs continuously and provides a clean baseline for comparing process metrics across Windows and Linux tools.

| | Task Manager (Details) | Process Explorer | htop / ps |
|---|---|---|---|
| PID | 4796 | 4796 | 1739 |
| Parent (PID & name) | Not displayed | `8376` (`cmd.exe`) | `1543` (`-bash` / `bash`) |
| Default Memory / CPU column | `532 K` (Memory — active private) | `836 K` (Private Bytes) | `0.1%` (`RES 2952 KiB`) |
| Working set / Resident | `5,280 K` (Working set) | `5,260 K` (Working Set) | `2952 KiB` (`RES`) |
| Virtual / Commit size | `900 K` (Commit size) | `460 K` (WS Private) | `5584 KiB` (`VIRT` / `VSZ`) |
| Shared memory | Not displayed separately | `~4,800 K` (`Working Set - WS Private`) | `2780 KiB` (`SHR`) |
| Handles / File descriptors | `82` | `78` | `4` (in `/proc/1739/fd`) |
| Threads | `3` | `1` | `1` (`NLWP`) |

### Where the tools disagree

1. **Default "Memory" vs. Working Set and RES:** Task Manager's default `Memory` column isolates only the *active private working set* (`532 K`), leaving out shared DLLs. In contrast, `Working set` (`5,280 K` in Task Manager, `5,260 K` in Process Explorer) and Linux `RES` (`2952 KiB` in `htop`) include all physical RAM pages mapped into the process, including shared libraries (`SHR 2780 KiB`).
2. **Private commit vs. total virtual address space (`VIRT`):** Windows reports private virtual memory backed by RAM or the pagefile (`Commit size 900 K` in Task Manager / `Private Bytes 836 K` in Process Explorer), whereas Linux `htop` and `ps` report `VIRT` (`5584 KiB`), which covers the entire virtual address space including shared mappings.
3. **Kernel handles vs. file descriptors:** Windows counts every kernel object handle held by the process (`82` in Task Manager, `78` in Process Explorer due to minor runtime differences between capture moments), while Linux only tracks open file descriptors in `/proc/1739/fd` (`4`: `stdin`, `stdout`, `stderr` attached to `/dev/tty1`, plus `1` ICMP socket).
4. **Parent process visibility and thread counts:** Task Manager's Details view hides `PPID` and lists `3` threads (accounting for internal Windows runtime/subsystem threads), whereas Process Explorer (`PPID 8376`, `1` thread) and `htop` (`PPID 1543`, `1` thread) clearly show the parent shell and single execution thread.

## Part 2 — Memory pressure

Guest RAM: `3.3 GiB` (`3.32 GiB` in `htop`) · swap: `2.0 GiB` (`/swap.img`)

Steps:
```bash
# 1. Start periodic sampling of RAM and swap every 3 seconds
(echo "=== BEFORE ==="; while true; do date; free -h; swapon; echo "---"; sleep 3; done) > memory-pressure.txt &

# 2. Run four escalating stress-ng memory stages
for p in 70% 90% 120% 400%; do
  stress-ng --vm 1 --vm-bytes $p --vm-keep -t 15s
done

# 3. Stop background sampler and mark completion
kill %1; echo "=== AFTER ===" >> memory-pressure.txt

# 4. Extract kernel OOM and lockup diagnostics
sudo journalctl -k | grep -E "oom-killer|oom-kill|Out of memory|workqueue" | tail -n 35 > kernel-log-extract.txt
sudo journalctl -k | grep -E "watchdog|soft lockup|rcu_preempt|clocksource" | tail -n 35 > lockup.txt
```

| | Before | During (worst) | After |
|---|---|---|---|
| Available | `2.9 GiB` (`00:10:02`) | `18 MiB` (`00:10:29`) | `2.9 GiB` (`00:10:35`) |
| Swap used | `0 B` (`00:10:02`) | `2.0 GiB` (`00:10:29`) | `152 MiB` (`00:10:35`) |

What happened: Before load (`00:10:02`–`00:10:05`), the guest used `252 MiB` of RAM with `2.9 GiB` available and `0 B` in swap. During the `70%` stage (`00:10:08`–`00:10:11`), RAM usage rose to `2.5 GiB` while swap barely moved (`4.0 MiB`). At `90%` (`00:10:14`–`00:10:17`), available RAM dropped to `208 MiB`, forcing the kernel to reclaim `buff/cache` (shrinking it from `484 MiB` to `226 MiB`) and page `348–382 MiB` into `/swap.img`. At `120%` (`00:10:20`–`00:10:23`), swap usage exceeded half of its capacity (`1.1–1.2 GiB`), and heavy swap thrashing triggered CPU scheduling stalls (`soft lockup` and `rcu_preempt` warnings recorded in `evidence/lockup.txt`). During the `400%` spike (`00:10:26`–`00:10:29`), both physical RAM (`3.2 GiB` used, `18 MiB` available) and swap (`2.0 GiB` used, `0 B` free) reached complete exhaustion, causing the OOM killer to terminate the worker at `00:10:30`. By `00:10:32`–`00:10:35`, available RAM immediately bounced back to `2.9 GiB`, while `152–156 MiB` remained in swap because inactive swapped pages are not faulted back into RAM until accessed.

### What the kernel did

```text
Nov 23 00:10:30 ubuntu kernel: stress-ng-vm invoked oom-killer: gfp_mask=0x140cca(GFP_HIGHUSER_MOVABLE|__GFP_COMP), order=0, oom_score_adj=1000
Nov 23 00:10:30 ubuntu kernel: Out of memory: Killed process 2341 (stress-ng-vm) total-vm:5482104kB, anon-rss:3184640kB, file-rss:512kB, shmem-rss:0kB, UID:1000 pgtables:10420kB oom_score_adj:1000
```

- **Which process:** `stress-ng-vm` (`PID 2341`).
- **Why:** `stress-ng` assigns `oom_score_adj=1000` to its memory workers so the kernel targets them first under memory exhaustion. At the moment of failure, `PID 2341` held the largest resident and swapped footprint (`anon-rss: 3,184,640 kB` ≈ `3.04 GiB` in RAM plus `508,928` pages in swap, `total-vm: 5.48 GB`), giving it the highest OOM score.
- **When:** `00:10:30` UTC—one second after `memory-pressure.txt` logged `2.0 GiB` full swap (`00:10:29`), freeing RAM back to `2.9 GiB` available by the next sample at `00:10:32`.

### Windows

```text
PS> Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-Resource-Exhaustion-Detector'} -MaxEvents 5 | Format-List TimeCreated, Id, Message
No events were found that match the specified selection criteria.
PS> Get-WinEvent -ListProvider Microsoft-Windows-Resource-Exhaustion-Detector | Select-Object Name

Name                                          
----                                          
Microsoft-Windows-Resource-Exhaustion-Detector
```

The second command verifies that the `Microsoft-Windows-Resource-Exhaustion-Detector` provider is registered and the filter query is valid, proving that Windows never approached commit exhaustion during the lab (no low-memory events such as Event ID 2004 were generated). Unlike Linux's OOM killer, Windows enforces a hard virtual commit limit (physical RAM + pagefile), warning the user and failing new allocation requests rather than automatically killing processes.

## What did not work the first time

- **Clipboard paste in the VirtualBox console:** Multi-line commands could not be pasted directly into the black VirtualBox terminal window, so concise one-line loops (or SSH from Windows PowerShell) had to be used instead.
- **Percentages below 100% did not exhaust memory:** Because `stress-ng --vm-bytes` allocates relative to *available* RAM rather than total RAM + swap (`3.3 GiB + 2.0 GiB = 5.3 GiB`), the `70%` run barely touched swap (`4.0 MiB`) and only `400%` saturated both RAM and swap to trigger the OOM killer.
- **Quitting `htop` while search mode was active:** Pressing `F3` opened the bottom `Search:` prompt, capturing the `q` quit key as search text until `Esc` was pressed first to exit search mode.

## Evidence

- [evidence/task-manager.png](evidence/task-manager.png) — Task Manager (Details tab): `PING.EXE` PID `4796` showing Working set, Memory, Commit size, Handles, and Threads
- [evidence/process-explorer.png](evidence/process-explorer.png) — Process Explorer: `PING.EXE` PID `4796`, PPID `8376`, Private Bytes, Working Set, WS Private, Handles, and Threads
- [evidence/htop.png](evidence/htop.png) — `htop` on Ubuntu: `ping 127.0.0.1` PID `1739`, PPID `1543`, `VIRT`, `RES`, and `SHR`
- [evidence/process-win.txt](evidence/process-win.txt) — PowerShell `Get-CimInstance` and `Get-Process` output confirming PID `4796` and parent `cmd` (`8376`)
- [evidence/process-ps.txt](evidence/process-ps.txt) — `ps` metrics and `/proc/1739/fd` open file descriptors on Ubuntu
- [evidence/memory-pressure.txt](evidence/memory-pressure.txt) — `free -h` and `swapon` sampled every 3 seconds from `BEFORE` through all four `stress-ng` stages to `AFTER`
- [evidence/kernel-log-extract.txt](evidence/kernel-log-extract.txt) — kernel log excerpt capturing `oom-killer` invocation and termination of `PID 2341`
- [evidence/lockup.txt](evidence/lockup.txt) — `soft lockup` and `rcu_preempt` stall warnings recorded during swap thrashing
- [evidence/eventlog-win.txt](evidence/eventlog-win.txt) — `Get-WinEvent` output confirming zero Windows memory exhaustion events
