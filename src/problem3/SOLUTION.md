# 🩺 Problem 3: Diagnose Me Doctor - NGINX VM High Storage (Memory) Usage

You are tasked with troubleshooting a production issue where an Ubuntu 24.04 VM running NGINX is showing **99% storage usage** (but we are told to treat it as a **memory issue**). This VM runs only one service: NGINX (as a load balancer).

---
Table of Contents

0. [Goal](#0-goal)
1. [Step-by-Step Troubleshooting Process](#1-step-by-step-troubleshooting-process)
   - [1.1 Initial Checks](#11-initial-checks)
   - [1.2 Check NGINX Memory Behavior](#12-check-nginx-memory-behavior)
   - [1.3 Check Logs and Error Indicators](#13-check-logs-and-error-indicators)
   - [1.4 Check Configuration for Memory-Sensitive Directives](#14-check-configuration-for-memory-sensitive-directives)
2. [Possible Root Causes & Remediation](#2-possible-root-causes--remediation)
3. [Recovery Steps](#3-recovery-steps)
   - [3.1 Cache Accumulation Issue](#31-cache-accumulation-issue)
   - [3.2 Log Accumulation Issue](#32-log-accumulation-issue)
4. [Conclusion](#4-conclusion)


---
## 0. Goal

Diagnose and troubleshoot **high memory usage** on a production VM hosting **NGINX**, and outline:
- Root causes you expect to find
- Steps to investigate
- Impacts to system
- Recovery and remediation actions

---

## 1. Step-by-Step Troubleshooting Process

### 1.1 **Initial Checks**

#### 🔎 Using `htop` Effectively
After launching `htop`, here are key metrics to observe:

- **CPU Usage (Top Bar)**: Look for cores constantly at 100% — this may indicate a CPU bottleneck, particularly if NGINX processes are at the top.
- **Memory & Swap**: High memory usage with swap activity indicates pressure. If swap is near 100%, the system is likely memory-thrashing.
- **Processes (Bottom Table)**:
  - Sort by `%MEM` or `%CPU` using `F6` or by clicking the column header.
  - Look for `nginx` processes consuming excessive memory (especially worker processes).
  - Check for zombie processes (Z status) or unusually high resident memory (RES column).
- **Uptime & Load Average (Top Left)**: If load average is higher than the number of CPUs, this may indicate sustained resource contention.

```bash
# Check overall memory usage
free -h

# Real-time top processes by memory
htop  # or use 'top'

# List top memory-hungry processes
ps aux --sort=-%mem | head -n 10

# Check disk (in case storage logs are involved)
df -hT /
```

### 1.2 **Check NGINX Memory Behavior**
```bash
# Check NGINX master and worker memory
ps -C nginx -o pid,%mem,%cpu,cmd

# Check number of open connections
sudo netstat -anp | grep nginx | wc -l

# Check number of worker processes
pgrep -fc nginx
```

> **Note:** High memory might be caused by: too many simultaneous open connections, long-lived keep-alives, request buffering, or misconfiguration of workers.

### 1.3 **Check Logs and Error Indicators**
```bash
# NGINX access & error logs
less /var/log/nginx/access.log
less /var/log/nginx/error.log

# System logs
journalctl -u nginx --since "1 hour ago"
dmesg | tail -n 50
```

Look for:
- Repeated client retries
- Slow clients
- Large requests being buffered in memory

### 1.4 **Check Configuration for Memory-Sensitive Directives**

Important NGINX directives that can increase memory pressure:
```nginx
worker_processes auto;
worker_connections 10240;
keepalive_timeout 75s;
client_body_buffer_size 64k;
proxy_buffering on;
proxy_buffers 8 32k;
proxy_buffer_size 64k;
```

- Too many worker processes + high connections can cause high memory usage
- Long keep-alives result in open memory contexts for idle connections
- Large client buffers (client_body_buffer_size) consume RAM per request

---

## 2. Possible Root Causes & Remediation

Here we expand root causes to include storage-heavy factors such as excessive cache and log accumulation, while still respecting the original focus on memory-level insights.

| Cause | Description | Impact | Recovery |
|-------|-------------|--------|----------|
| **Keep-alive connections** | Too many idle open connections | Gradual memory creep | Reduce `keepalive_timeout` to 30s or less |
| **Improper NGINX config** | Too many `worker_processes`, large buffer sizes | Memory leaks, high per-request RAM | Tune `proxy_buffering`, `worker_connections`, etc. |
| **Traffic spike / attack** | DDoS or sudden legitimate surge | Exhausts memory, may cause OOM kill | Add rate limiting, enable caching, autoscale horizontally |
| **Excessive logging** | Verbose access logs in memory (if buffered) | Uses tmpfs/log memory buffers | Rotate logs, use async logger, limit log levels |
| **Memory leak in NGINX module** | Rare but possible (e.g. 3rd party module) | Memory grows over time | Update or remove offending modules, monitor via valgrind |
| **Zombie/Orphaned processes** | Old worker processes not cleaned | Memory not released | Restart NGINX safely, use `--no-daemon` to debug |

---

## 3. Recovery Steps

### 3.1 Cache Accumulation Issue
**Investigation:**
- Check cache usage: `du -sh /var/nginx/cache`
- Validate cache directives in `/etc/nginx/nginx.conf` under `http {}`
- Look for large `max_size`, long `inactive` duration

**Impact:**
- Consumes memory (if on tmpfs) or disk rapidly
- Slows down upstream if cache expires too infrequently

**Recovery:**
```bash
rm -rf /var/nginx/cache/*
```
Then limit via config:
```nginx
proxy_cache_path /var/nginx/cache keys_zone=CACHE:60m levels=1:2 inactive=1h max_size=10g;
proxy_cache CACHE;
```
Reload with:
```bash
nginx -t && systemctl reload nginx
```
Monitor with:
```bash
watch -n 60 "du -sh /var/nginx/cache"
```

### 3.2 Log Accumulation Issue
**Investigation:**
- Check log sizes: `du -sh /var/log/nginx/`
- Truncate massive files:
```bash
> /var/log/nginx/access.log
> /var/log/nginx/error.log
```

**Recovery:**
- Enable proper log rotation:
```bash
nano /etc/logrotate.d/nginx
```
With contents like:
```bash
/var/log/nginx/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        systemctl reload nginx > /dev/null 2>&1
    endscript
}
```

Test it:

```bash
logrotate -f /etc/logrotate.d/nginx
```

**Advanced:**
Send logs to centralized syslog:
```nginx
access_log syslog:server=10.0.1.42,tag=nginx,severity=info;
error_log syslog:server=10.0.1.42 warn;
```


1. **Immediate (Stopgap)**:
   - Restart NGINX: `sudo systemctl restart nginx`
   - Trim logs if ne1. [Goal](#1-goal)
2. [Step-by-Step Troubleshooting Process](#1-step-by-step-troubleshooting-process)
   - [1.1 Initial Checks](#11-initial-checks)
   - [1.2 Check NGINX Memory Behavior](#12-check-nginx-memory-behavior)
   - [1.3 Check Logs and Error Indicators](#13-check-logs-and-error-indicators)
   - [1.4 Check Configuration for Memory-Sensitive Directives](#14-check-configuration-for-memory-sensitive-directives)
3. [Possible Root Causes & Remediation](#2-possible-root-causes--remediation)
4. [Recovery Steps](#3-recovery-steps)
   - [3.1 Cache Accumulation Issue](#31-cache-accumulation-issue)
   - [3.2 Log Accumulation Issue](#32-log-accumulation-issue)
5. [Conclusion](#4-conclusion)
cessary: `sudo truncate -s 0 /var/log/nginx/*`
   - Free up tmp files if mounted to memory: `/tmp`, `/var/tmp`

2. **Config Adjustments**:
   - Lower `keepalive_timeout`, `client_body_buffer_size`
   - Enable caching (e.g. `proxy_cache`) to reduce load on backend
   - Rate limit bad traffic: `limit_req`, `limit_conn`

3. **Monitoring & Alerting**:
   - Set CloudWatch / Prometheus alerts on memory
   - Enable NGINX status module (`stub_status`) and monitor

4. **Long-term**:
   - Consider moving to containerized NGINX + autoscaling group
   - Introduce load test and chaos engineering scenarios to prevent future blind spots

---

## 4. Conclusion

High memory usage on an NGINX VM can have various causes, often rooted in traffic volume, misconfiguration, or bad actors. Proactive monitoring, sensible defaults, and structured logs/configs are key to reliability. Recovery requires both short-term triage and long-term architectural thinking.

