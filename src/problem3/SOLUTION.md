Provide your solution here:

**Troubleshooting workflow (order matters)**

1. **Confirm the signal**  
   Check whether monitoring is measuring the root filesystem, a mount or inodes.

2. **Establish safe access**  
   If usage is critical, avoid heavy writes (large find outputs to disk, copying huge trees). Prefer read-only inspection first. Keep a rollback plan for any config change.

3. **Classify the problem quickly**

```bash
# Check overall disk usage
df -hT

# Identify largest directories
du -h --max-depth=2 / | sort -hr | head -20

# Find large files (>100MB)
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -hr

# Check inode usage (sometimes the issue is too many small files)
df -i
```

4. **Drill into the largest directory** (repeat `du` until you find the folder). Typical hotspots on an NGINX LB: `/var/log`, `/var/lib/nginx`, `/var/cache`, `/var/lib/journal`, `/var/lib/snapd`, `/tmp`.

5. **Correlate with NGINX behavior**  
   Inspect access/error log volume, buffer paths, and rotation configuration.

```bash
# Check NGINX log sizes
du -sh /var/log/nginx/*

# Check NGINX configuration for log paths
grep -r "access_log\|error_log" /etc/nginx/

# Check for core dumps
find / -name "core.*" -o -name "core" -type f 2>/dev/null

# Check NGINX cache if enabled
du -sh /var/cache/nginx/* 2>/dev/null
```

6. **System-wide checks**

```bash
# Check system logs
du -sh /var/log/*

# Check journal logs (systemd)
journalctl --disk-usage

# Check package manager cache
du -sh /var/cache/apt/*
du -sh /var/lib/apt/lists/*

# Check for deleted but open files (zombie files)
lsof +L1
```

7. **Check rotation and journal retention**

```bash
sudo logrotate -d /etc/logrotate.d/nginx
sudo journalctl --disk-usage
```

8. **After mitigation, prevent recurrence**  
   Fix logrotate, ship logs off-box, tune buffers, set retention, add alerts at 70/80/90%, and consider separate `/var/log` volume for LBs.

---

**Common root causes, impacts and recovery**

**Issue 1: Unbounded NGINX (and system) logs**

**What you expect to see**

- `/var/log/nginx/access.log` (sometimes multiple vhost logs) growing continuously, especially under high RPS or verbose logging.
- `/var/log/nginx/error.log` spikes during upstream flapping, TLS issues, or misconfiguration loops.
- `/var/lib/nginx/` not huge unless buffering (see Issue 2), but log dirs dominate.

**Likely causes / scenarios**

- No log rotation or logrotate misconfigured (wrong paths, wrong permissions, postrotate script failing).
- High traffic + large access log lines (many headers, cookies logged, upstream timing fields).
- Health-check endpoints logged at full line rate (classic LB disk killer).
- Journald also filling `/var/log/journal` if persistent logging is enabled and uncapped.

**Impacts**

- Disk full can prevent new writes: NGINX may fail to append logs; depending on setup, reloads can fail; worst case new sessions degrade if temp/buffer paths cannot allocate.
- Operational risk: cannot collect apt security updates; SSH/session issues if `/` cannot write small files; monitoring agents may fail.
- NGINX may crash if unable to write logs.

**Recovery steps**

- Immediate relief (controlled): rotate/truncate the specific huge log after copying tail if needed for incident evidence:

```bash
# OR compress recent logs (only if /tmp has space)
sudo gzip -c /var/log/nginx/access.log > /tmp/access.log.$(date +%F_%H%M).gz

# IMMEDIATE: Truncate logs (preserves log rotation)
sudo truncate -s 0 /var/log/nginx/access.log
sudo truncate -s 0 /var/log/nginx/error.log
sudo nginx -s reopen
```

(Prefer copytruncate via logrotate if signals are tricky; reopen is cleaner when supported.)

- Sustainable fix: ensure `/etc/logrotate.d/nginx` matches actual log paths; verify logrotate runs daily and succeeds; reduce logging (disable access logs for noisy locations, switch to sampled logging, or ship to central logging and keep local retention small).

```bash
# Check log rotation configuration
cat /etc/logrotate.d/nginx

# Fix log rotation (example config)
cat > /etc/logrotate.d/nginx <<'EOF'
/var/log/nginx/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
EOF

# Force log rotation
sudo logrotate -f /etc/logrotate.d/nginx
```

- Consider shipping logs to external system (fluentd, etc).

**Issue 2: On-disk proxy/client buffering (temp files) growing**

**What you expect to see**

- Large usage under `/var/lib/nginx/proxy_temp`, `fastcgi_temp`, `uwsgi_temp`, `client_body_temp` (exact dirs depend on modules and config).
- Often correlates with large uploads, slow upstreams, or buffering settings forcing spool-to-disk.

**Likely causes / scenarios**

- `proxy_buffering on` (default) + large responses + slow clients → NGINX buffers to disk.
- `client_max_body_size` raised for uploads; clients send big bodies; upstream slow → `client_body_temp` grows.
- Mis-sized `proxy_buffers` / `proxy_busy_buffers_size` causing unexpected buffering behavior under load.

**Impacts**

- Can spike much faster than logs during incidents (upstream latency blowouts).
- Disk full can cause 502/504-class failures if NGINX cannot create temp files.

**Recovery steps**

- Immediate: identify top temp dirs, stop the bleeding by stabilizing upstream/latency if possible; delete stale temp files only if NGINX is not actively using them (safest during a controlled reload/maintenance window).
- Sustainable: tune buffering for an LB role (often: ensure you are not accidentally caching or buffering entire large downloads); cap `client_body_buffer_size` appropriately; consider `proxy_max_temp_file_size 0` only if you understand RAM implications; ensure disk monitoring includes `/var/lib/nginx`.

**Issue 3: Journald / systemd log bloat**

**What you expect to see**

```bash
journalctl --disk-usage
# Output: Archived and active journals take up 40GB
```

**Likely causes / scenarios**

- Systemd journal logs accumulating without limits.
- Service restarts generating excessive logs.
- Upstream connection failures logged repeatedly.

**Impacts**

- Slower boot times.
- Debugging becomes difficult with huge logs.

**Recovery steps**

```bash
# Check current size
journalctl --disk-usage

# Clean old logs (keep last 3 days)
journalctl --vacuum-time=3d

# OR limit by size
journalctl --vacuum-size=500M

# Configure permanent limits — edit /etc/systemd/journald.conf and add/modify:
#   SystemMaxUse=1G
#   SystemKeepFree=2G
#   MaxRetentionSec=7day
vi /etc/systemd/journald.conf

# Restart journald
systemctl restart systemd-journald

# Verify
journalctl --verify
```

**Prevention**

- Set `SystemMaxUse=1G` in `/etc/systemd/journald.conf`.
- Configure appropriate retention policies.
- Forward critical logs to external system.

**Issue 4: Core dumps from NGINX crashes**

**What you expect to see**

```bash
find / -name "core.*" -type f
# /var/crash/core.nginx.12345 (5GB each)
```

**Scenario**

- NGINX segfaults/crashes generating core dumps.
- Multiple restarts creating multiple dumps.
- Default unlimited core dump size.

**Impact**

- Indicates stability issues.
- Disk space consumption.

**Recovery steps**

```bash
# Find core dumps
find / -name "core.*" -o -name "*.core" -type f 2>/dev/null

# Archive important ones for analysis
mkdir /root/coredumps_backup
mv /var/crash/core.* /root/coredumps_backup/ 2>/dev/null

# Delete old dumps
find /var/crash -name "core.*" -mtime +7 -delete
rm -f /core.*

# Disable core dumps (if not needed)
echo "* soft core 0" >> /etc/security/limits.conf
ulimit -c 0

# Or limit size
echo "kernel.core_pattern=/var/crash/core.%e.%p" >> /etc/sysctl.conf
echo "* soft core 100000" >> /etc/security/limits.conf  # 100MB limit

# Investigate WHY NGINX is crashing
journalctl -u nginx -p err --since today
nginx -t  # Check config validity
```

**Prevention**

- Set core dump limits or disable entirely.
- Fix underlying NGINX stability issues.
- Implement automated cleanup of old dumps.
- Consider using `coredumpctl` for managed storage.

**Issue 5: Deleted but open files ("zombie files")**

**What you expect to see**

```bash
lsof +L1
# COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NLINK NODE NAME
# nginx    1234 root  5w   REG  252,0  15G      0     12345 /var/log/nginx/access.log (deleted)
```

**Scenarios**

- Logs rotated/deleted while NGINX still writing to old file handle.
- Process holds reference to deleted file.
- Disk space not reclaimed until process restart.

**Impact**

- Disk space appears used but files invisible.

**Recovery steps**

```bash
# Identify deleted but open files
lsof +L1 | grep nginx

# Gracefully reload NGINX (releases file handles)
nginx -s reload

# OR restart NGINX service
systemctl restart nginx

# Verify space reclaimed
df -h

# Check NGINX is healthy
systemctl status nginx
curl -I http://localhost
```

**Prevention**

- Ensure log rotation sends proper signal to NGINX (`kill -USR1`).
- Use `copytruncate` option in logrotate (less efficient but safer).
- Monitor for zombie files in alerting.
