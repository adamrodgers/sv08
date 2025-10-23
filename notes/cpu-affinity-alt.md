# CPU Affinity Optimization for Klipper Systems

## Overview

CPU affinity assigns specific processes to dedicated CPU cores, reducing context switching and improving real-time performance for 3D printing. This guide dedicates two cores exclusively to Klipper while letting all other services naturally balance across the remaining cores.

### Core Assignment Strategy
- **Cores 0-1**: System services, Moonraker, Crowsnest (natural scheduling)
- **Cores 2-3**: Klipper only (dedicated, isolated cores)

### Why This Matters
- **Minimizes jitter**: Klipper gets two uninterrupted cores for precise motion control
- **May improve print quality**: Less interference during critical motion control operations
- **Better system stability**: Klipper isolated from system service contention
- **May resolve TTC errors**: Could fix "Timer Too Close" errors caused by CPU scheduling delays
- **Headroom for complex kinematics**: Input shaping and advanced features have dedicated compute resources

**Note: This configuration requires more real-world testing to confirm all benefits, but early results suggest improvements in system performance and reliability.**

---

## Prerequisites

- 4+ core ARM SBC (Raspberry Pi 4/5, embedded CM4 modules like those in printers such as the SV08, or similar)
- SSH access to your printer
- Klipper installed (Moonraker and Crowsnest optional)

---

## Configuration

**Purpose**: Gives Klipper exclusive access to cores 2-3. Real-time motion control requires uninterrupted CPU time to prevent print defects and timing issues. Two dedicated cores provide maximum processing power for input shaping, pressure advance, and complex kinematics calculations. All other services (system, Moonraker, Crowsnest) will naturally schedule on cores 0-1.

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

## Apply Changes

Reload systemd configuration:
```bash
sudo systemctl daemon-reload
```

Restart Klipper service:
```bash
sudo systemctl restart klipper.service
```

Reboot for clean application (optional but recommended):
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

Show CPU affinity for Klipper:
```bash
ps -hU $(whoami) | grep klippy | awk '{print $1}' | xargs -I{} taskset -c -p {}
```

### Expected Output
```
--- Klipper Process ---
  613 Ssl    2:05 /home/biqu/klippy-env/bin/python /home/biqu/klipper/klippy/klippy.py

--- CPU Affinity ---  
pid 613's current affinity list: 2,3      # Klipper on cores 2-3 ✓
```

Check other services (should show all cores 0-3):
```bash
ps -hU $(whoami) | grep -E "moonraker|crowsnest" | awk '{print $1}' | xargs -I{} taskset -c -p {}
```

Expected output:
```
pid 614's current affinity list: 0-3      # Moonraker can use any core ✓
pid 605's current affinity list: 0-3      # Crowsnest can use any core ✓
```

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
Check override file exists:
```bash
ls -la /etc/systemd/system/klipper.service.d/
cat /etc/systemd/system/klipper.service.d/override.conf
```

Check effective configuration:
```bash
systemctl show klipper.service | grep CPUAffinity
```

### Reset Configuration
Remove override file:
```bash
sudo rm /etc/systemd/system/klipper.service.d/override.conf
```

Reload and restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart klipper
```

---

## Expected Benefits

- **Possibly smoother prints**: May reduce layer inconsistencies and improve surface finish
- **Potentially higher reliable speeds**: Could reduce timing-related failures at high speeds with dual-core processing
- **Better input shaping performance**: More CPU headroom for advanced motion algorithms
- **May resolve TTC errors**: Could fix "Timer Too Close" errors from CPU scheduling conflicts
- **Better system responsiveness**: System services not artificially constrained to a single core
