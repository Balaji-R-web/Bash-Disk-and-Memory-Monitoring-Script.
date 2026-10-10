# Project 4: Bash Disk and Memory Monitoring Script

## 📌 Project Overview

This project demonstrates how to automate Linux disk and memory monitoring using Bash scripting on an Amazon Linux EC2 instance.

The script collects disk usage, memory utilization, CPU load, and system uptime. It generates a monitoring report, records results in a log file, and can be scheduled to run automatically.

## 🎯 Objectives

- Monitor disk utilization.
- Check memory usage.
- Monitor system load and uptime.
- Automate health checks using Bash.
- Generate timestamped monitoring reports.
- Record monitoring results in log files.
- Configure scheduled execution using cron.

## 🛠️ Environment

- Operating System: Amazon Linux
- Platform: AWS EC2
- Shell: Bash
- Tools: `df`, `free`, `uptime`, `awk`, `date`, `cron`

---

## 1. Check the Linux Environment

**Command:**

```bash
whoami
```

**Sample Output:**

```text
ec2-user
```

Check the operating system:

```bash
cat /etc/os-release
```

Check the Bash version:

```bash
bash --version
```

**Explanation:** Confirms the user, operating system, and Bash environment.

## 2. Check Disk Utilization

**Command:**

```bash
df -h
```

**Sample Output:**

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1   20G  6.0G   14G  31% /
tmpfs           950M     0  950M   0% /dev/shm
```

**Explanation:** Displays filesystem capacity, used space, available space, and utilization percentage. The values are illustrative.

## 3. Check Memory Usage

**Command:**

```bash
free -h
```

**Sample Output:**

```text
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       450Mi       700Mi        10Mi       800Mi       1.3Gi
Swap:             0B          0B          0B
```

**Explanation:** Displays RAM and swap usage. The `available` column is generally more useful than the `free` column when assessing memory available for new workloads.

## 4. Check CPU Load and Uptime

**Command:**

```bash
uptime
```

**Sample Output:**

```text
19:30:00 up 5 days, 3:20, 1 user, load average: 0.08, 0.12, 0.10
```

**Explanation:** Displays system uptime, logged-in users, and load averages over 1, 5, and 15 minutes.

## 5. Create the Project Directory

**Command:**

```bash
mkdir -p ~/project4-monitoring/logs
cd ~/project4-monitoring
```

Verify:

```bash
pwd
ls -la
```

**Explanation:** Creates a dedicated directory for the monitoring script and its logs.

## 6. Create the Bash Monitoring Script

Create the file:

```bash
nano monitor.sh
```

If `nano` is unavailable, use `vi monitor.sh` or create the file using a heredoc.

Paste the following script:

```bash
#!/bin/bash

# Linux Disk and Memory Monitoring Script

set -u

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
LOG_DIR="${LOG_DIR:-$SCRIPT_DIR/logs}"
LOG_FILE="$LOG_DIR/system-monitor.log"

DISK_THRESHOLD="${DISK_THRESHOLD:-80}"
MEMORY_THRESHOLD="${MEMORY_THRESHOLD:-80}"

mkdir -p "$LOG_DIR" || exit 1

if ! [[ "$DISK_THRESHOLD" =~ ^[0-9]+$ ]] ||
   ! [[ "$MEMORY_THRESHOLD" =~ ^[0-9]+$ ]] ||
   (( DISK_THRESHOLD > 100 || MEMORY_THRESHOLD > 100 )); then
    echo "ERROR: Thresholds must be integers from 0 to 100." >&2
    exit 1
fi

TIMESTAMP="$(date '+%Y-%m-%d %H:%M:%S')"
HOSTNAME_VALUE="$(hostname)"
DISK_USAGE="$(df -P / | awk 'NR==2 {gsub(/%/, "", $5); print $5}')"
MEMORY_USAGE="$(free | awk '/^Mem:/ {if ($2 > 0) printf "%d", ($2 - $7) * 100 / $2}')"
LOAD_AVERAGE="$(awk '{print $1}' /proc/loadavg)"
UPTIME_VALUE="$(uptime -p)"

if [[ ! "$DISK_USAGE" =~ ^[0-9]+$ ]] ||
   [[ ! "$MEMORY_USAGE" =~ ^[0-9]+$ ]]; then
    echo "ERROR: Unable to collect disk or memory usage." >&2
    exit 1
fi

{
    echo "========================================"
    echo "Linux System Monitoring Report"
    echo "Timestamp: $TIMESTAMP"
    echo "Hostname: $HOSTNAME_VALUE"
    echo "Disk Usage (/): ${DISK_USAGE}%"
    echo "Memory Usage: ${MEMORY_USAGE}%"
    echo "Load Average (1 min): $LOAD_AVERAGE"
    echo "System Uptime: $UPTIME_VALUE"

    if (( DISK_USAGE >= DISK_THRESHOLD )); then
        echo "WARNING: Disk usage reached the threshold."
    else
        echo "STATUS: Disk usage is normal."
    fi

    if (( MEMORY_USAGE >= MEMORY_THRESHOLD )); then
        echo "WARNING: Memory usage reached the threshold."
    else
        echo "STATUS: Memory usage is normal."
    fi

    echo "========================================"
    echo
} | tee -a "$LOG_FILE"
```

Save the file in Nano using `Ctrl+O`, press `Enter`, then use `Ctrl+X` to exit.

**Explanation:** The script collects system statistics, checks configurable thresholds, prints a report, and appends each report to a log file.

The memory calculation uses the `available` memory value reported by `free`. Disk monitoring checks the root filesystem (`/`).

## 7. Make the Script Executable

**Command:**

```bash
chmod +x monitor.sh
```

Verify:

```bash
ls -l monitor.sh
```

**Sample Output:**

```text
-rwxr-xr-x 1 ec2-user ec2-user 1800 Oct 10 19:30 monitor.sh
```

**Explanation:** Adds execute permission to the script. File size and timestamp will differ.

## 8. Run the Monitoring Script

**Command:**

```bash
./monitor.sh
```

**Illustrative Output:**

```text
========================================
Linux System Monitoring Report
Timestamp: 2026-10-10 19:30:00
Hostname: ip-172-31-13-186
Disk Usage (/): 31%
Memory Usage: 42%
Load Average (1 min): 0.08
System Uptime: up 5 days, 3 hours, 20 minutes
STATUS: Disk usage is normal.
STATUS: Memory usage is normal.
========================================
```

**Explanation:** The script displays a report using the current system metrics. Actual values will depend on your EC2 instance.

## 9. View the Monitoring Log

**Command:**

```bash
cat logs/system-monitor.log
```

To view only the latest entries:

```bash
tail -n 30 logs/system-monitor.log
```

To follow new log entries in real time:

```bash
tail -f logs/system-monitor.log
```

Press `Ctrl+C` to stop following the log.

**Explanation:** The log file stores reports from each execution.

## 10. Test Custom Thresholds

Run the script with a disk threshold of 50% and memory threshold of 60%:

```bash
DISK_THRESHOLD=50 MEMORY_THRESHOLD=60 ./monitor.sh
```

**Explanation:** Environment variables allow you to change the warning thresholds without editing the script.

If current utilization meets or exceeds a configured threshold, the script prints a warning. Otherwise, it prints a normal status.

## 11. Verify the Script Syntax

**Command:**

```bash
bash -n monitor.sh
```

**Expected Output:** No output if the script has valid Bash syntax.

**Explanation:** Checks the script for syntax errors without running the monitoring commands.

## 12. Run the Script in Debug Mode

**Command:**

```bash
bash -x monitor.sh
```

**Explanation:** Prints executed commands and variable expansions, helping you troubleshoot the script.

Avoid debug mode when a script might expose passwords, tokens, or other sensitive information.

## 13. Schedule Automatic Monitoring with Cron

Find the full path to the script:

```bash
readlink -f ~/project4-monitoring/monitor.sh
```

Edit your user's cron configuration:

```bash
crontab -e
```

Add the following line, replacing `/home/ec2-user/project4-monitoring/monitor.sh` with the actual path returned by `readlink`:

```cron
*/5 * * * * /home/ec2-user/project4-monitoring/monitor.sh
```

**Explanation:** Runs the monitoring script every five minutes.

Verify the configuration:

```bash
crontab -l
```

Cron uses a limited environment, so use absolute paths for scripts and other required files. Confirm that the script's directory and log file are writable by the user running the cron job.

## 14. Check Cron Service Status

On Amazon Linux systems using systemd, check the cron service:

```bash
sudo systemctl status crond
```

If the service is not running and you are authorized to enable it:

```bash
sudo systemctl enable --now crond
```

**Explanation:** Checks and starts the cron daemon where applicable. Service configuration may vary by distribution and environment.

## 15. Check the Monitoring Log After Scheduling

After allowing the scheduled job to run, execute:

```bash
tail -n 40 ~/project4-monitoring/logs/system-monitor.log
```

**Expected Result:** New timestamped reports should appear at the configured interval.

If there are no new reports, check the cron service, the absolute script path, file permissions, and cron logs.

## 16. Clean Up the Cron Job

When you no longer need automatic monitoring, open the crontab:

```bash
crontab -e
```

Remove the monitoring line and save the file.

Verify:

```bash
crontab -l
```

**Explanation:** Prevents the lab script from continuing to run periodically.

## 📚 Important Commands Learned

| Command | Purpose |
|---|---|
| `df -h` | Check filesystem usage |
| `free -h` | Check RAM and swap usage |
| `uptime` | Check uptime and system load |
| `awk` | Extract and calculate values |
| `tee -a` | Display output and append to a file |
| `chmod +x` | Make a script executable |
| `bash -n` | Check shell syntax |
| `bash -x` | Debug script execution |
| `tail -f` | Follow log updates |
| `crontab -e` | Schedule recurring tasks |
| `systemctl status` | Inspect service status |

## 🔐 Best Practices

- Set realistic resource thresholds for the server.
- Store logs in a dedicated directory.
- Review log growth and configure log rotation for long-running monitoring.
- Use cron with absolute paths.
- Keep monitoring scripts readable and test changes before scheduling them.
- Avoid running scripts as root unless necessary.
- Do not commit secrets, credentials, or sensitive system information to GitHub.

## 🚀 Possible Improvements

- Monitor multiple filesystems.
- Add CPU utilization sampling over time.
- Send alerts through an approved email or monitoring platform.
- Integrate with CloudWatch metrics and alarms.
- Add log rotation and historical trend analysis.

## ✅ Project Outcome

This project demonstrates Bash scripting, Linux resource monitoring, threshold-based warnings, log generation, and scheduled automation on Amazon Linux.

## 👨‍💻 Technologies Used

- Linux
- Amazon Linux
- Bash Scripting
- AWS EC2
- Cron
- Linux System Utilities
