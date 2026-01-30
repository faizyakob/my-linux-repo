### Table of contents

- [Introduction](#introduction)
- [Concept](#concept)
- [How it works](#how-it-works)
- [Configuring demo service using systemd timer](#configuring-demo-service-using-systemd-timer)
- [Common mistakes and debugging tips](#common-mistakes-and-debugging-tips)
- [Cleanup and key takeaways](#cleanup-and-key-takeaways)

### Introduction

`systemd.timer` is the modern Linux replacement for many cron use cases.
It integrates scheduling directly with systemd, giving you better logging, dependency handling, and control.

This tutorial walks through a simple but complete example:

A `hello.service` that writes a timestamped message.

A `hello.timer` that runs it periodically.

How to inspect, debug, and compare with cron.

We first go through systemd timer concepts and how it works. 
Then we briefly compares it with cron, before ending it with common mistakes (and how to avoid them) & key takeaways.

### Concept

🔑 A **systemd timer** is a systemd unit that schedules and triggers another unit (usually a .service) at specific times or intervals. It serves a role similar to cron, but is fully integrated into the systemd ecosystem, using the same unit model, dependency handling, logging, and state management.

In systemd, timers are **not standalone jobs**. A timer unit only defines when something should run; the actual work is defined in a corresponding **service unit**. This separation makes scheduling declarative, predictable, and easier to inspect and manage.

<img width="497" height="310" alt="image" src="https://github.com/user-attachments/assets/bb259266-4e3f-42d3-b04b-de5da4035a2f" />

Important idea:
> Timers do nothing by themselves — they only trigger services.

### How it works

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

### Configuring demo service using systemd timer

In this section we cover creating a simple `hello.service` that output a log, and then enable this service via its timer unit, `hello.timer`. As with everything Linux, we can run a service as root, or as normal user. 
The choice will affect the directory in which we place both the timer and service unit files. 

<details>
  <summary> System-level example (runs as root)</summary><br>
  
1. Create the service
   
   This service writes a friendly message with a timestamp.

   **File:** `/etc/systemd/system/hello.service`

   ```
   [Unit]
   Description=Hello World faiz service

   [Service]
   Type=oneshot
   ExecStart=/usr/bin/bash -c 'echo "Hello from systemd at $(date)" >> /var/log/hello.log'
   ```
   **This file contains the following to keep it simple:**
   - `Type=oneshot` → perfect for timers
   - No background process
   - Visible output
   - Piping the output lo hello.log file.

2. Create the timer

   **File:** `/etc/systemd/system/hello.timer`

   ⚠️  Note: It is important that the name of timer MATCH EXACTLY the name of its service. This is how a timer know which unit to activate.

   ```
   [Unit]
   Description=Run hello.service every minute

   [Timer]
   OnBootSec=30s
   OnUnitActiveSec=1min
   Persistent=true
   Unit=hello.service

   [Install]
   WantedBy=timers.target
   ```

   **Explanation**
   |Setting|Meaning|
   |---|---|
   |`OnBootSec=30s`|First run 30s after boot|
   |`OnUnitActiveSec=1min`|Run every minute|
   |`Persistent=true`|Catch up if system was off|
   |`Unit=`|Explicit service mapping|
   
   🔑 Note the unit file contains [Install] section. This represents a unit that wants to have this timer activated. A **.target** is another systemd unit file that represents a group of files that must be running before satisfying the condition of the target. In this case, our timer unit will be invoked by the `timers.target` unit file, which usually used to bring up along other timer units as well when the system first boot up.

   🔑 

4. Enable the timer

   ```
   sudo systemctl daemon-reload
   sudo systemctl enable --now hello.timer
   ```

   🔑 Enabling the timer actually creates a symbolic link from the timers.target to the location of the timer unit file: _Created symlink /etc/systemd/system/timers.target.wants/hello.timer → /etc/systemd/system/hello.timer_.
   
6. Verify it’s scheduled
   
   ```
   systemctl list-timers --all | grep hello
   ```

   <img width="1215" height="37" alt="image" src="https://github.com/user-attachments/assets/17f08044-d2c6-476d-a967-bf428bb4db6c" />

7. Verify execution

   **Check logs via journal**
   ```
   journalctl -u hello.service
   ```

   <img width="798" height="136" alt="image" src="https://github.com/user-attachments/assets/06ca5003-c3c3-4e2f-b133-e82e3f2acd25" />

   **Check the output file**
   
   ```
   cat /var/log/hello.log
   ```

   <img width="542" height="97" alt="image" src="https://github.com/user-attachments/assets/4b20085c-e475-48e7-8bea-c8986bf882cf" />


</details>

<details>
  <summary> User-level timer (no root required)</summary><br>

  Systemd timers also work **per user**, which is something cron handles poorly.

  1. Create directory
     ```
     mkdir -p ~/.config/systemd/user
     ```
     
  2. User service

     **File:** `~/.config/systemd/user/hello-user.service`
     
     ```
     [Unit]
     Description=User hello service

     [Service]
     Type=oneshot
     ExecStart=/usr/bin/echo "Hello from user timer at $(date)"
     ```
     
  4. User timer

     **File:** `~/.config/systemd/user/hello-user.timer`

     ```
     [Unit]
     Description=User hello timer

     [Timer]
     OnBootSec=1min
     OnUnitActiveSec=5min

     [Install]
     WantedBy=timers.target
     ```

  4. Enable it

     ```
     systemctl --user daemon-reload
     systemctl --user enable --now hello-user.timer
     ```
     
  5. View logs

     ```
     journalctl --user -u hello-user.service
     ```
  
</details>

### Comparison with cronjob

We can of course achieve the preceding with **cron**: 

```
*/1 * * * * echo "Hello from systemd at $(date)" >> /var/log/hello.log
```
However, systemd timer provides the following: 
- Separate schedule and execution
- Built-in logging
- Better error handling
- Can depend on:
  + network
  + mounts
  + other services

Below table simplified the comparison between cron and Systemd timer at high-level.
Which one to utilize depends on use case. However, Systemd timer is more modern of the two and is now the recommended approach. 

| Feature | cron | systemd.timer|
|---|---|---|
|Logging|Manual|Built-in`journalctl`|
|Dependencies|No|Yes|
|Missed jobs after reboot|No|Yes(`Persistent=true`)|
|Unified management|No|Yes (`systemctl`)|
|User-level timers|Limited|Native|

### Common mistakes and debugging tips

❌ **Using** `Type=simple`<br>
Timers expect the service to finish.<br>

Use: 
```
Type=oneshot
```

❌ **Forgetting** `daemon-reload`<br>

Any time you change unit files:
```
systemctl daemon-reload
```

❌ **Enabling the service instead of the timer**<br>

You usually **enable only the timer**:
```
systemctl enable --now hello.timer
```

❌ **Expecting output on stdout**<br>

Use:
- files
- logger
- journalctl

Example:
```
ExecStart=/usr/bin/logger "Hello from systemd"
```

🐛 Debugging tips:

|Task|Command|
|---|---|
|See timer state|`systemctl status hello.timer`|
|See service logs|`journalctl -u hello.service`|
|See all timers|`systemctl list-timers --all`|
|Test service manually|`systemctl start hello.service`|

### Cleanup and key takeaways

```
sudo systemctl disable --now hello.timer
sudo rm /etc/systemd/system/hello.{service,timer}
sudo systemctl daemon-reload
```

🐫 Key takeaways

- Timers **trigger services**
- Services do the actual work
- systemd timers are more reliable than cron
- User-level timers are first-class citizens
- `journalctl` is your best friend

