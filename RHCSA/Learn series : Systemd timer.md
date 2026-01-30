### Table of contents

- [Introduction](#introduction)
- [Concept](#concept)
- [What and how it works](#what-and-how-it-works)
- [Configuring demo service using systemd timer](#configuring-demo-service-using-systemd-timer)
- [Checking the systemd timer is working](#checking-the-systemd-timer-is-working)
- [Comparison with cronjob](#comparison-with-cronjob)
- [Tips and outro](#tips-and-outro)

### Introduction

`systemd.timer` is the modern Linux replacement for many cron use cases.
It integrates scheduling directly with systemd, giving you better logging, dependency handling, and control.

This tutorial walks through a simple but complete example:

A `hello.service` that writes a timestamped message.

A `hello.timer` that runs it periodically.

How to inspect, debug, and compare with cron.

We first go through systemd timer concepts and how it works. 

### Concept

🔑 A **systemd timer **is a systemd unit that schedules and triggers another unit (usually a .service) at specific times or intervals. It serves a role similar to cron, but is fully integrated into the systemd ecosystem, using the same unit model, dependency handling, logging, and state management.

In systemd, timers are **not standalone jobs**. A timer unit only defines when something should run; the actual work is defined in a corresponding **service unit**. This separation makes scheduling declarative, predictable, and easier to inspect and manage.

### What and how it works

A systemd timer consists of two units:

+ `<name>.timer` — defines when an action should run
+ `<name>.service` — defines what action should run

The workflow is:

1. The timer unit is enabled and started.<br>
2. systemd monitors the timer based on its configuration (time-based or event-based).<br>
3. When the timer condition is met, systemd **activates the associated service unit**.<br>
4. The service runs to completion (or stays active, depending on its type).<br>
5. Logs and execution status are handled by systemd and accessible via `journalctl`.<br>

Common timing modes:

- Monotonic timers (relative to system events):
  + `OnBootSec=` – time after system boot
  + `OnStartupSec=` – time after systemd startup
  + `OnUnitActiveSec=` / `OnUnitInactiveSec=` – time relative to last service run

- Calendar timers (wall-clock based):
  + `OnCalendar=` – schedules like `Mon *-*-* 09:00:00`

Key characteristics

- Timers can **catch up** on missed runs using `Persistent=true` (e.g., system was powered off).
- Execution state and failures are visible using standard systemd tools.
- Timers respect dependencies, ordering, and resource controls defined in the service unit.
- systemd replaces cron’s text-based scheduling with **unit-based, inspectable scheduling**.

In short:
**The timer decides when, the service defines what, and systemd coordinates everything**.
     
### Comparison with cronjob

Following table simplified the comparison between cron and Systemd timer at high-level.
Which one to utilize depends on use case. However, Systemd timer is more modern of the two and is now the recommended approach. 

| Feature | cron | systemd.timer|
|---|---|---|
|Logging|Manual|Built-in`journalctl`|
|Dependencies|No|Yes|
|Missed jobs after reboot|No|Yes(`Persistent=true`)|
|Unified management|No|Yes (`systemctl`)|
|User-level timers|Limited|Native|

### Tips and outro

**autofs** is used for automating the mounting of filesystems. It is most useful in multi-user environment where there are many directories mounting tasks to handle, which becoming repetitive over time. It also provides some automation with wildcard mapping, when centralized home directories is necessary to provide stricter control.

Tips:
1. If you are unable to change directory into the mountpoints, check the directory permission on the NFS server. Specifically for wildcard mapping, ensure each user can access its ```/home/``` directory by matching the UID.
2. If solely using NFSv4, both **mountd** and **nfs-server** services share port 2049, so there are only 2 ports to be whitelisted. 




