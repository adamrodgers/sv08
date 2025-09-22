# CPU Affinity Optimization for Klipper Systems

## Overview

CPU affinity assigns specific processes to dedicated CPU cores, reducing context switching and improving real-time performance for 3D printing. This guide optimizes 4-core systems by dedicating cores to specific services.

### Core Assignment Strategy
- **Cores 0-1**: General system services and background processes
- **Core 2**: Crowsnest (webcam streaming) and Moonraker (API server)
- **Core 3**: Klipper (real-time motion control) - **dedicated core**

### Why This Matters
- **Reduces jitter**: Klipper gets uninterrupted CPU time for precise motion control
- **May improve print quality**: Less interference during critical operations
- **Better system stability**: Prevents resource contention between services
- **May resolve TTC errors**: Could fix "Timer Too Close" errors caused by system overload

**Note: This configuration requires more real-world testing to confirm all benefits, but early results suggest significant improvements in system performance and reliability.**

---

## Prerequisites

- 4+ core ARM SBC (Raspberry Pi 4/5, embedded CM4 modules like those in printers such as the SV08, or similar)
- SSH access to your printer
- Klipper, Moonraker, and Crowsnest installed

---

## Step 1: Set System-Wide CPU Affinity

**Purpose**: Configures all system services to use cores 0-1, reserving cores 2-3 for printer services.

Create the system configuration directory:
```bash
sudo mkdir -p /etc/systemd/system.conf.d
```

Set default CPU affinity for all services to cores 0-1:
```bash
cat << EOF | sudo tee /etc/systemd/system.conf.d/cpuaffinity.conf
[Manager]
CPUAffinity=0-1
EOF
```

---

## Step 2: Configure Crowsnest and Moonraker (Core 2)

**Purpose**: Assigns webcam streaming and API services to core 2. These I/O-bound services can share a core since they rarely peak simultaneously.

Configure Crowsnest CPU affinity:
```bash
sudo systemctl edit crowsnest.service
```

Add and save:
```ini
[Service]
CPUAffinity=2
```

Configure Moonraker CPU affinity:
```bash  
sudo systemctl edit moonraker.service
```

Add and save:
```ini
[Service]
CPUAffinity=2
```

---

## Step 3: Configure Klipper (Core 3)

**Purpose**: Gives Klipper exclusive access to core 3. Real-time motion control requires uninterrupted CPU time to prevent print defects and timing issues.

Configure Klipper CPU affinity:
```bash
sudo systemctl edit klipper.service
```

Add and save:
```ini
[Service]
CPUAffinity=3
```

---

## Step 4: Apply Changes

Reload systemd configuration:
```bash
sudo systemctl daemon-reload
```

Restart Klipper service:
```bash
sudo systemctl restart klipper.service
```

Restart Moonraker service:
```bash
sudo systemctl restart moonraker.service
```

Restart Crowsnest service:
```bash
sudo systemctl restart crowsnest.service
```

Reboot for clean application (recommended):
```bash
sudo reboot
```

---

## Verification

### Check CPU Affinity
View your processes:
```bash
ps -hU $(whoami)
```

Show CPU affinity for each process:
```bash
ps -hU $(whoami) | awk '{print $1}' | xargs -I{} taskset -c -p {}
```

### Expected Output
```
--- Process List ---
  613 Ssl    2:05 /home/biqu/klippy-env/bin/python /home/biqu/klipper/klippy/klippy.py
  614 Ssl    0:48 /home/biqu/moonraker-env/bin/python -m moonraker  
  605 Ss     0:00 /bin/bash /usr/local/bin/crowsnest

--- CPU Affinities ---  
pid 613's current affinity list: 3        # Klipper on core 3 ✓
pid 614's current affinity list: 2        # Moonraker on core 2 ✓  
pid 605's current affinity list: 2        # Crowsnest on core 2 ✓
```

Monitor real-time CPU usage:
```bash
htop
```
Press 'F2' → Display options → Check "Detailed CPU time". Core 3 should show primarily Klipper activity.

---

## Troubleshooting

### Check Service Status
Check service status:
```bash
sudo systemctl status klipper.service
```

View recent logs:
```bash
journalctl -u klipper.service -n 20
```

### Verify Configuration
Check override files exist:
```bash
ls -la /etc/systemd/system/klipper.service.d/
```

Check effective configuration:
```bash
systemctl show klipper.service | grep CPUAffinity
```

### Reset Configuration
Remove override files:
```bash
sudo rm /etc/systemd/system/klipper.service.d/override.conf
sudo rm /etc/systemd/system/moonraker.service.d/override.conf  
sudo rm /etc/systemd/system/crowsnest.service.d/override.conf
sudo rm /etc/systemd/system.conf.d/cpuaffinity.conf
```

Reload and restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart klipper moonraker crowsnest
```

---

## Expected Benefits

- **Possibly smoother prints**: May reduce layer inconsistencies and improve surface finish
- **Potentially higher reliable speeds**: Could reduce timing-related failures at high speeds  
- **Stable webcam streams**: Consistent frame rates during printing
- **Better system responsiveness**: Improved resource utilization across cores
- **May resolve TTC errors**: Could fix "Timer Too Close" errors from system overload
