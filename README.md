# AMP web panel dies: findings (2026-09-29)

This covers why the CubeCoders AMP web panel (ADS01, port 8080) on **hp2 (192.168.11.206)** and the
**devserver (192.168.11.220)** "stops and doesn't respond", and what was changed.

**Fix:** [TheDyXer/amp-panel-patch](https://github.com/TheDyXer/amp-panel-patch) (public). It's applied on both
servers.

## Summary

**The panel doesn't hang: it gets killed at boot.**
- AMP's own systemd unit starts the panel.
- Then, for every "Start on Boot" instance, the unit downloads a Docker image from Docker Hub.
- On the 40/8 Mbit line that takes longer than the unit's 180 s start limit. systemd then kills the unit, and the
  panel inside it dies too.
- Nothing restarts it. On .206 the nightly `ampfix.sh` cron job brought it back at 00:15. On .220 nothing did,
  so the panel was dead for its whole 37-day uptime.

## How it happens

`/etc/systemd/system/ampinstmgr.service` (as installed by AMP):

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
User=amp
ExecStart=/opt/cubecoders/amp/ampinstmgr startboot true
TimeoutSec=180
ExecStop=/opt/cubecoders/amp/ampinstmgr stopall
```

1. `ampinstmgr startboot` starts ADS01 in a tmux session **inside the unit's process group** and logs
   `AMP instance ADS01 is now running.`
2. It then starts each "Start on Boot" instance as a Docker container, pulling `cubecoders/ampbase:<tag>` first.
3. At 180 s systemd logs `start operation timed out. Terminating.` and SIGTERMs the whole process group, tmux and
   ADS included. The unit ends up `failed (Result: timeout)`.
4. Port 8080 is closed. Other machines see `Connection refused`, and nothing brings the panel back.

## Evidence

| Box | Boot | ampinstmgr | Result | Last thing before the kill |
|---|---|---|---|---|
| .220 | 2026-08-02 12:53 | not captured | **timeout** at 12:55:37 | only the last lines were captured |
| .220 | 2026-08-18 11:29 | not captured | **timeout** at 11:33:00 | only the last lines were captured |
| .220 | 2026-08-23 21:45 | 2.8.0.4 | **timeout** at 21:48:17 | `java: Pulling from cubecoders/ampbase`; 3 of 8 layers done |
| .220 | 2026-09-29 22:32 | 2.8.0.8 | OK: `Finished` in 8 s | — |
| .206 | 2026-07-16 23:22 | 2.7.2.8 | **timeout** at 23:26:21 | Docker Hub `TLS handshake timeout`, then `Conflict. The container name "/AMP_OneBlock01" is already in use` |

The full unit logs are in [`evidence/`](evidence/).

**How the panel's absence shows up:**
- **.220's own logs:** `AMP_Logs/` has only one file, from tonight. The panel never ran during the 2026-08-23 →
  2026-09-29 boot.
- **.206's panel log:** it polls .220 as the remote target "Original" every minute, and got
  `Connection Refused` from `192.168.11.220:8080` all day on 2026-09-28 (1,156 + 179 times).
- **.206:** the panel ran under `cron.service` (started by `ampfix.sh`), not under `ampinstmgr.service`. That
  unit still shows `failed (Result: timeout) since Thu 2026-07-16`.

## What it isn't (checked on .206, 23 h after the last nightly restart)

- **Not a hang:** `curl http://127.0.0.1:8080/` returns 200 in 1.5 ms.
- **Not a leak:** 23 threads, 212 fds, 254 MB RSS. None of these grow.
- **Not memory:** no out-of-memory kills, and 15 GB RAM with 11 GB available.
- **No crashes:** 58 days of `AMP_Logs` (2026-08-31 → 2026-09-29) show no panel starts other than the nightly
  00:15 restart and the midnight-UTC log rollover.
- **Not the log noise:** the ~2,600 remote-target errors a day (below) finish in ~3 s per cycle and don't pile
  up.

## Fix applied (2026-09-29, no reboots)

This was installed on both servers with the README one-liner:
`curl -fsSL https://raw.githubusercontent.com/TheDyXer/amp-panel-patch/main/install.sh | sudo bash`

- **Boot fix:** the drop-in `/etc/systemd/system/ampinstmgr.service.d/10-boot-timeout.conf` sets
  `TimeoutStartSec=30min`. `systemctl show` now reports `TimeoutStartUSec=30min` on both.
- **Safety net:** `15 0 * * * /usr/local/sbin/amp-panel-restart` in root's crontab. It runs the same
  `ampinstmgr restart ADS` as the old script, and logs to `/var/log/amp-panel-restart.log`.
- **.206:** the old `15 0 * * * /home/colombus/ampfix.sh` cron line was replaced. The file is still there.
- **Test run on .220:** the panel restarted from PID 4310 to 1748544 and answered HTTP 200 about 8 s later.
- **Before touching the servers:** the installer was tested in a throwaway ubuntu:24.04 container with stub AMP:
  install, re-install (idempotent), `curl | bash` mode, `--remove`, and the error paths.
- **Installer v1.1.0** (same night) adds:
  - step-by-step output with a final verdict;
  - a panel restart during install, which waits until the panel answers;
  - a boot-history check ("boots in the journal where the boot timeout killed AMP");
  - a `--status` mode.
- **Results with v1.1.0:**
  - .220: a real remove + fresh install showed `start timeout: 3min → 30min` and
    `panel restarted (pid 2538319 → 2615275) and answers HTTP 200`. It found 3 boots with the timeout.
  - .206: `--status` reported "Patched and working" and flagged the 2026-07-16 boot as timed out.
- **A bug found and fixed along the way:** `--remove` stopped early when the patch's cron line was root's only
  cron line.

**Still to verify:** the next time .220 reboots (planned: the 24.04 → 26.04 upgrade),
`journalctl -b -u ampinstmgr` must end with `Finished`, and port 8080 must be listening. .206 must not be
rebooted: its power-on is unreliable.

## Side findings (not fixed)

1. **.220 can't reach github.com or api.github.com.** The TCP connect times out; the route is via 192.168.11.1.
   - At the same moment, .206 and the main PC connect fine, and .220 reaches `raw.githubusercontent.com` and
     Docker Hub.
   - Effect: AMP's template sync on .220 fails every ~3 min, each time after a 135 s timeout:
     `fatal: unable to access 'https://github.com/CubeCoders/AMPTemplates.git/': Failed to connect to github.com
     port 443 after 135139 ms`.
   - Anything else on .220 that uses github.com (git, release downloads) is affected as well.
2. **.206's panel polls two remote targets every minute:** "LPserver" at 192.168.11.201:8080 (usually Host
   Unreachable) and "Original" at 192.168.11.220:8080.
   - That's ~2,600 error lines a day, about 1.2 MB of log.
   - "Original" should answer again now that .220's panel runs. If .201 is gone for good, remove LPserver from
     the panel.
3. **Version drift:** ampinstmgr is 2.8.0.8 on both boxes, but the panels are older (ADS 2.7.2.0 on .206 and
   2.6.5.2 on .220).
4. **.206 at boot, 2026-07-16:** `docker run` failed on a leftover `AMP_OneBlock01` container name from an
   unclean shutdown. AMP normally removes its containers when they stop.
5. **.220's instances** (OneBlockCracked01, LobbyCracked01, Main01) started at the 22:32 boot. At 23:02 they
   were stopped through AMP (`Stopping instance … AMP_SYSTEM`), not by this work.

## Environment

| | hp2 (.206) | devserver (.220) |
|---|---|---|
| **OS** | Ubuntu 24.04.5, kernel 6.8.0-117 | Ubuntu 24.04.5, kernel 6.8.0-142 |
| **Hardware** | HP Compaq Elite 8300 USDT, 15 GB RAM, 2.5" HDD | 15 GB RAM |
| **AMP** | ampinstmgr 2.8.0.8, ADS 2.7.2.0 | ampinstmgr 2.8.0.8, ADS 2.6.5.2 |
| **Instances** | Sokobot01, ScanBOT01, OneBlock01, Proxy01, ProxyCracked01 | OneBlockCracked01, LobbyCracked01, Main01 |
| **Internet** | 40/8 Mbit | 40/8 Mbit |
