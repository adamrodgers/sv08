# CPU Affinity Optimization for Klipper Systems

## Overview

CPU affinity assigns specific processes to dedicated CPU cores, reducing context switching and improving real-time performance for 3D printing. This guide optimizes 4-core systems by dedicating two cores exclusively to Klipper for maximum motion control performance.

### Core Assignment Strategy
- **Core 0**: General system services and background processes
- **Core 1**: Crowsnest (webcam streaming) and Moonraker (API server)
- **Cores 2-3**: Klipper (real-time motion control) - **two dedicated cores**

### Why This Matters
- **Minimizes jitter**: Klipper gets two uninterrupted cores for precise motion control
- **May improve print quality**: Less interference during critical operations with dual-core processing power
- **Better system stability**: Prevents resource contention between services
- **May resolve TTC errors**: Could fix "Timer Too Close" errors caused by system overload
- **Headroom for complex kinematics**: Input shaping and advanced features have dedicated compute resources

**Note: This configuration requires more real-world testing to confirm all benefits, but early results suggest significant improvements in system performance and reliability.**

---

## Prerequisites

- 4+ core ARM SBC (Raspberry Pi 4/5, embedded CM4 modules like those in printers such as the SV08, or similar)
- SSH access to your printer
- Klipper, Moonraker, and Crowsnest installed

---

## Step 1: Set System-Wide CPU Affinity

**Purpose**: Configures all system services to use core 0 only, leaving cores 1-3 available for printer services.

Create the system configuration directory:
```bash
sudo mkdir -p /etc/systemd/system.conf.d
```

Set default CPU affinity for all services to core 0:
```bash
cat << EOF | sudo tee /etc/systemd/system.conf.d/cpuaffinity.conf
[Manager]
CPUAffinity=0
EOF
```

---

## Step 2: Configure Crowsnest and Moonraker (Core 1)

**Purpose**: Assigns webcam streaming and API services to core 1. These I/O-bound services can share a core since they rarely peak simultaneously.

Configure Crowsnest CPU affinity:
```bash
sudo systemctl edit crowsnest.service
```

Add and save:
```ini
[Service]
CPUAffinity=1
```

Configure Moonraker CPU affinity:
```bash  
sudo systemctl edit moonraker.service
```

Add and save:
```ini
[Service]
CPUAffinity=1
```

---

## Step 3: Configure Klipper (Cores 2-3)

**Purpose**: Gives Klipper exclusive access to cores 2-3. Real-time motion control requires uninterrupted CPU time to prevent print defects and timing issues. Two dedicated cores provide maximum processing power for input shaping, pressure advance, and complex kinematics calculations.

Configure Klipper CPU affinity:
```bash
sudo systemctl edit klipper.service
```

Add and save:
```ini
[Service]
CPUAffinity=2-3
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
pid 613's current affinity list: 2,3      # Klipper on cores 2-3 ✓
pid 614's current affinity list: 1        # Moonraker on core 1 ✓  
pid 605's current affinity list: 1        # Crowsnest on core 1 ✓
```

Monitor real-time CPU usage:
```bash
htop
```
Press 'F2' → Display options → Check "Detailed CPU time". Cores 2-3 should show primarily Klipper activity, core 1 should show Moonraker/Crowsnest, and core 0 should show system processes.

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
- **Potentially higher reliable speeds**: Could reduce timing-related failures at high speeds with dual-core processing
- **Better input shaping performance**: More CPU headroom for advanced motion algorithms
- **Stable webcam streams**: Consistent frame rates during printing on dedicated core
- **Better system responsiveness**: Improved resource utilization with isolated services
- **May resolve TTC errors**: Could fix "Timer Too Close" errors from system overload
