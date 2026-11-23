# Lab 6 — Live processes and memory pressure

Pair: Aisulu · Date: 22-23.11.2026 · Guests: Ubuntu Server 26.04.1 LTS (3.3 GiB RAM, 2.0 GiB swap) / Windows 11 (16 GB RAM)

## Part 1 — One process, three tools

Process chosen: `ping 127.0.0.1` (`PING.EXE -t 127.0.0.1` on Windows, `ping 127.0.0.1` on Ubuntu) — why: it is a lightweight built-in network utility preinstalled on both systems that runs continuously and provides a clean baseline for comparing process metrics across Windows and Linux tools.

| | Task Manager (Details) | Process Explorer | htop / ps |
|---|---|---|---|
| PID | 4796 | 4796 | 1739 |
| Parent (PID & name) | Not displayed | `8376` (`cmd.exe`) | `1543` (`-bash` / `bash`) |
| Default Memory / CPU column | `532 K` (Memory — active private) | No such default column | `0.1%` (`RES 2952 KiB`) |
| Private working set | `532 K` (Memory — active private) | `460 K` (WS Private; closely related counter, captured at a different moment) | No dedicated column |
| Working set / Resident | `5,280 K` (Working set) | `5,260 K` (Working Set) | `2952 KiB` (`RES`) |
| Virtual / Commit size | `900 K` (Commit size) | `836 K` (Private Bytes — commit equivalent) | `5584 KiB` (`VIRT` / `VSZ`) |
| Shared memory | Not displayed separately | `~4,800 K` (`Working Set - WS Private`) | `2780 KiB` (`SHR`) |
| Handles / File descriptors | `82` | `78` (upper table; lower pane blank) | `4` (in `/proc/1739/fd`) |
| Threads | `3` | `1` | `1` (`NLWP`) |

### Where the tools disagree

1. **Default "Memory" vs. Working Set and RES:** Task Manager's default `Memory` column isolates only the *active private working set* (`532 K`), leaving out shared DLLs, whereas Process Explorer has no single default "Memory" column and instead splits physical usage into `Working Set` (`5,260 K`) and `WS Private` (`460 K` — a closely related private working set counter, captured at a different moment from Task Manager). Both `Working set` (`5,280 K` / `5,260 K`) and Linux `RES` (`2952 KiB` in `htop`) include all physical RAM pages mapped into the process, including shared libraries (`SHR 2780 KiB`).
2. **Private commit vs. total virtual address space (`VIRT`):** Windows reports private virtual memory backed by RAM or the pagefile (`Commit size 900 K` in Task Manager / `Private Bytes 836 K` in Process Explorer), whereas Linux `htop` and `ps` report `VIRT` (`5584 KiB`), which covers the entire virtual address space including shared mappings.
3. **Kernel handles vs. file descriptors:** Windows counts kernel object handles held by the process (`82` in Task Manager, `78` in Process Explorer's upper `Handles` column, though Process Explorer's lower `Handles` pane rendered empty in the captured screenshot and the exact reason was not determined), while Linux only tracks open file descriptors in `/proc/1739/fd` (`4`: `stdin`, `stdout`, `stderr` attached to `/dev/tty1`, plus `1` ICMP socket).
4. **Parent process visibility and thread counts:** Task Manager's Details view hides `PPID` and reports `3` threads for `PID 4796`, whereas Process Explorer shows `PPID 8376` and `1` thread for the exact same PID (matching Linux `PPID 1543`, `1` thread). The exact reason why Task Manager and Process Explorer report different thread counts (`3` vs. `1`) and slightly different handle totals (`82` vs. `78`) for the same process is unclear, likely reflecting differences in how each utility enumerates or refreshes thread and handle states at the moment of capture.

## Part 2 — Memory pressure

Guest RAM: `3.3 GiB` (`3.32 GiB` in `htop`) · swap: `2.0 GiB` (`/swap.img`)

*(Deviation from brief: instead of running 60–90 second stages, short ~6-second bursts with `--vm 2` were used so all four escalating stages completed inside a ~27-second capture window from `00:10:02` to `00:10:29`.)*

Steps:
```bash
# 1. Start periodic sampling of RAM and swap every 3 seconds
(echo "=== BEFORE ==="; while true; do date; free -h; swapon; echo "---"; sleep 3; done) > memory-pressure.txt &

# 2. Run four short escalating stress-ng stages (~6s each, 2 VM workers)
for p in 70% 90% 120% 400%; do
  stress-ng --vm 2 --vm-bytes $p --vm-keep -t 6s
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

What happened (by timestamps and `free -h` readings in `evidence/memory-pressure.txt`):
- `00:10:02`–`00:10:05` (baseline): `252 MiB` of RAM was used, `2.9 GiB` was available, and swap was at `0 B`.
- `00:10:08`–`00:10:11`: used RAM jumped to `2.5 GiB` (`665–670 MiB` available) while swap barely moved (`0 B` → `4.0 MiB`).
- `00:10:14`–`00:10:17`: used RAM reached `3.0 GiB` (`208–215 MiB` available), forcing the kernel to shrink `buff/cache` (from `484 MiB` down to `226 MiB`) and page `348–382 MiB` out to `/swap.img`.
- `00:10:20`–`00:10:23`: used RAM climbed to `3.1 GiB` (`112–118 MiB` available) and swap passed 50% capacity (`1.1–1.2 GiB`).
- `00:10:26`–`00:10:29`: peak exhaustion—both physical RAM (`3.2 GiB` used, `18 MiB` available) and swap (`2.0 GiB` used, `0 B` free) became completely saturated until the OOM killer terminated a worker at `00:10:30`.
- `00:10:32`–`00:10:35` (`AFTER`): available RAM immediately recovered to `2.9 GiB` (`246–248 MiB` used), while `152–156 MiB` remained in swap because inactive swapped pages are not faulted back into RAM until accessed.
- Responsiveness: interactive responsiveness (typing or live `htop` updates) was not observed directly during the runs; the background sampler continued writing every 3 seconds from `00:10:02` to `00:10:35` without missed intervals, while the kernel logged CPU scheduling stalls in `evidence/lockup.txt` during the swap-heavy phase (an earlier attempt at `00:09:44` with PIDs `2312`/`2313` before `BEFORE`, and during the logged run at `00:10:15` and `00:10:24`, including `rcu_preempt` self-detected stall and `soft lockup - CPU#0 stuck for 22s` on `stress-ng-vm:2341`).

### What the kernel did

```text
Nov 23 00:10:30 ubuntu kernel: stress-ng-vm invoked oom-killer: gfp_mask=0x140cca(GFP_HIGHUSER_MOVABLE|__GFP_COMP), order=0, oom_score_adj=1000
Nov 23 00:10:30 ubuntu kernel: Out of memory: Killed process 2341 (stress-ng-vm) total-vm:5482104kB, anon-rss:3184640kB, file-rss:512kB, shmem-rss:0kB, UID:1000 pgtables:10420kB oom_score_adj:1000
```

- **Which process:** `stress-ng-vm` (`PID 2341`).
- **Why:** `stress-ng` sets `oom_score_adj=1000` on its memory workers so they are chosen first when memory runs out. Between the two running workers (`PID 2340` and `PID 2341`), `PID 2341` also held the largest resident and swapped footprint (`796,160` pages / `3,184,640 kB` `anon-rss` vs. `397,992` pages in `PID 2340`, plus `508,928` pages in swap), giving it the highest OOM badness score.
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

The second command confirms that the `Microsoft-Windows-Resource-Exhaustion-Detector` provider is registered and the query is valid. The empty result shows only that no Event ID `2004` entries exist in the `System` log, meaning Windows did not record a commit-exhaustion event during testing. Unlike Linux's OOM killer, when Windows approaches or exhausts its commit charge limit (physical RAM + pagefile), the Resource Exhaustion Detector logs Event ID `2004` naming the processes consuming the most memory and fails new allocation requests rather than automatically killing running processes.

## What did not work the first time

- **Empty lower Handles pane in Process Explorer:** Although the upper process table recorded `78` total handles for `PING.EXE` (`PID 4796`), the bottom `Handles` pane stayed blank (the exact reason was not determined), and filtering by `pin` hid the parent `cmd.exe` tree.
- **Percentages below 100% and short stage durations:** Because `stress-ng --vm-bytes` allocates relative to *available* RAM rather than total RAM + swap (`3.3 GiB + 2.0 GiB = 5.3 GiB`), lower percentages barely touched swap (`4.0 MiB` at `00:10:11`). In addition, short ~6-second stages (`-t 6s`) were used instead of the brief's 60-second runs, compressing all four stages into ~27 seconds and requiring `400%` to saturate both RAM and swap.
- **Earlier lockup before log start:** An initial memory-pressure run triggered `soft lockup` messages at `00:09:44` (`PID 2312`/`2313`) before the `=== BEFORE ===` marker (`00:10:02`) was started, so those earlier timestamps remained in `journalctl -k` when `lockup.txt` was exported.
- **Quitting `htop` while search mode was active:** Pressing `F3` opened the bottom `Search:` prompt, capturing the `q` quit key as search text until `Esc` was pressed first to exit search mode.

## Evidence

- [evidence/task-manager.png](evidence/task-manager.png) — Task Manager (Details tab): `PING.EXE` PID `4796` showing Working set, Memory, Commit size, Handles, and Threads
- [evidence/process-explorer.png](evidence/process-explorer.png) — Process Explorer: `PING.EXE` PID `4796`, PPID `8376`, Private Bytes, Working Set, WS Private, Handles (`78` in table; lower pane blank), and Threads
- [evidence/htop.png](evidence/htop.png) — `htop` on Ubuntu: `ping 127.0.0.1` PID `1739`, PPID `1543`, `VIRT`, `RES`, and `SHR`
- [evidence/process-win.txt](evidence/process-win.txt) — PowerShell `Get-CimInstance` and `Get-Process` output confirming PID `4796` and parent `cmd` (`8376`)
- [evidence/process-ps.txt](evidence/process-ps.txt) — `ps` metrics and `/proc/1739/fd` open file descriptors on Ubuntu
- [evidence/memory-pressure.txt](evidence/memory-pressure.txt) — `free -h` and `swapon` sampled every 3 seconds from `BEFORE` through the `stress-ng` stages to `AFTER`
- [evidence/kernel-log-extract.txt](evidence/kernel-log-extract.txt) — kernel log excerpt capturing `oom-killer` invocation and termination of `PID 2341`
- [evidence/lockup.txt](evidence/lockup.txt) — `soft lockup` and `rcu_preempt` stall warnings logged during the swap-heavy phase
- [evidence/eventlog-win.txt](evidence/eventlog-win.txt) — `Get-WinEvent` output confirming zero Event ID `2004` memory exhaustion events in Windows
