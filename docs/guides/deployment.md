# Deployment Guide

Production deployment of SYNAPSE on a Linux server.

---

## 1. Prerequisites

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| Python | 3.10+ | 3.12+ |
| Redis | 7.0+ | 7.0.15+ |
| RAM | 1 GB | 2 GB |
| Disk | 10 GB | 20 GB |
| Network | Localhost only | Localhost only (Redis) |

---

## 2. Architecture Overview

SYNAPSE requires three running services:

```
┌──────────────────────────────────────────────┐
│                  systemd                      │
│                                              │
│  ┌──────────────┐  ┌──────────────┐          │
│  │ synapse-   │  │ synapse-agent-b │          │
│  │ agent-a      │  │              │          │
│  │ (Agent A API)│  │ (Agent B API)│          │
│  │ port 8000    │  │ port 8002    │          │
│  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │
│         └────────┬─────────┘                  │
│                  │                            │
│         ┌────────v────────┐                   │
│         │  redis-server   │                   │
│         │  port 6379      │                   │
│         └─────────────────┘                   │
└──────────────────────────────────────────────┘
```

The SYNAPSE Bridge (Agent B side) runs as a thread within the `synapse-agent-b` service, listening on Redis pub/sub.

---

## 3. Service Configuration

### 3.1 Agent A Service

```ini
# /etc/systemd/system/synapse-agent-a.service
[Unit]
Description=SYNAPSE Agent A - autonomous instance
After=network.target redis-server.service

[Service]
Type=simple
User=synapse
WorkingDirectory=/opt/synapse/agent_a
EnvironmentFile=/opt/synapse/agent_a/.env
ExecStart=/opt/synapse/agent_a/venv/bin/python api.py
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

### 3.2 Agent B Bridge Service

```ini
# /etc/systemd/system/synapse-agent-b.service
[Unit]
Description=SYNAPSE Agent B Bridge
After=network.target redis-server.service

[Service]
Type=simple
User=synapse
WorkingDirectory=/opt/synapse/agent_b
EnvironmentFile=/opt/synapse/agent_b/.env
ExecStart=/opt/synapse/agent_b/venv/bin/python api.py
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

### 3.3 Redis Service

Redis uses the system default configuration at `/etc/redis/redis.conf`. Key settings:

```conf
# Redis listens only on localhost (no external access)
bind 127.0.0.1 ::1

# Disable persistence (SYNAPSE uses disk for state)
save ""
appendonly no

# Memory limit (recommended for production)
maxmemory 256mb
maxmemory-policy allkeys-lru
```

---

## 4. Service Management

### Start / Stop / Restart

```bash
# Start all SYNAPSE services
sudo systemctl start redis-server
sudo systemctl start synapse-agent-a
sudo systemctl start synapse-agent-b

# Stop all
sudo systemctl stop synapse-agent-b
sudo systemctl stop synapse-agent-a

# Restart a single service
sudo systemctl restart synapse-agent-a

# Enable on boot
sudo systemctl enable redis-server
sudo systemctl enable synapse-agent-a
sudo systemctl enable synapse-agent-b
```

### Check Status

```bash
# Service status
systemctl status synapse-agent-a
systemctl status synapse-agent-b
systemctl status redis-server

# SYNAPSE health check (API endpoint)
curl http://localhost:8000/synapse/health
```

Expected health response:

```json
{
  "health": {
    "redis_connected": true,
    "agent_a_subscribed": true,
    "agent_b_subscribed": true
  },
  "active_count": 0,
  "max_concurrent": 3
}
```

---

## 5. Logging

### journald (systemd logs)

```bash
# Follow Agent A logs in real-time
journalctl -u synapse-agent-a -f

# Follow Agent B Bridge logs
journalctl -u synapse-agent-b -f

# View last 100 lines
journalctl -u synapse-agent-a -n 100

# Logs since specific time
journalctl -u synapse-agent-a --since "2026-02-08 14:00"

# Follow Redis logs
journalctl -u redis-server -f
```

### Session Journal

Each SYNAPSE session maintains an append-only journal at:

```
$SYNAPSE_WORKSPACE/SYNAPSE_SESSION_<id>/02_JOURNAL.md
```

This journal records all messages, transitions, and orchestrator decisions with timestamps.

### Redis Monitoring

```bash
# Real-time Redis command monitoring
redis-cli MONITOR

# Monitor SYNAPSE channels only
redis-cli SUBSCRIBE synapse:agent_a_to_agent_b synapse:agent_b_to_agent_a synapse:supervisor synapse:control

# Redis info
redis-cli INFO stats
redis-cli INFO memory
```

---

## 6. Environment Variables

Create `.env` files for each service. See the [Configuration Guide](configuration.md) for full details.

### Minimum required variables

**Agent A side** (`/opt/synapse/agent_a/.env`):

```bash
# Redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# SYNAPSE workspace
SYNAPSE_WORKSPACE=/opt/synapse/workspace

# Notifications (optional — configure for your notification provider)
# TELEGRAM_BOT_TOKEN=<your-token>
# TELEGRAM_CHAT_ID=<your-chat-id>

# Email (optional, for document delivery)
SYNAPSE_SMTP_FROM=<your-email>
SYNAPSE_SMTP_PASSWORD=<your-app-password>
SUPERVISOR_EMAIL=<supervisor-email>
```

**Agent B side** (`/opt/synapse/agent_b/.env`):

```bash
# Redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# SYNAPSE workspace
SYNAPSE_WORKSPACE=/opt/synapse/workspace

# API key
SYNAPSE_API_KEY=<your-key>
```

---

## 7. Security Hardening

### 7.1 Redis

```conf
# /etc/redis/redis.conf

# Bind to localhost only
bind 127.0.0.1 ::1

# Disable dangerous commands
rename-command CONFIG ""
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command DEBUG ""
```

### 7.2 File Permissions

```bash
# Session directories: owner-only access
chmod 700 $SYNAPSE_WORKSPACE/SYNAPSE_SESSION_*

# Environment files: owner-only read
chmod 600 /opt/synapse/agent_a/.env
chmod 600 /opt/synapse/agent_b/.env

# Service files: root-owned
sudo chown root:root /etc/systemd/system/synapse-*.service
sudo chmod 644 /etc/systemd/system/synapse-*.service
```

### 7.3 Firewall

SYNAPSE operates entirely on localhost. Ensure external access is restricted:

```bash
# Only allow SSH from outside
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable

# Verify no SYNAPSE port is exposed
sudo ss -tlnp | grep -E '8000|8002|6379'
# All should show 127.0.0.1 only
```

---

## 8. Monitoring and Alerting

### 8.1 Health Check Script

```bash
#!/bin/bash
# /opt/synapse/scripts/synapse_health.sh

HEALTH=$(curl -s http://localhost:8000/synapse/health 2>/dev/null)

if [ $? -ne 0 ]; then
    echo "CRITICAL: Agent A API unreachable"
    exit 2
fi

REDIS=$(echo "$HEALTH" | python3 -c "import sys,json; print(json.load(sys.stdin)['health']['redis_connected'])" 2>/dev/null)

if [ "$REDIS" != "True" ]; then
    echo "CRITICAL: Redis disconnected"
    exit 2
fi

echo "OK: SYNAPSE healthy"
exit 0
```

### 8.2 Cron-based Monitoring

```cron
# Check SYNAPSE health every 5 minutes
*/5 * * * * /opt/synapse/scripts/synapse_health.sh >> /var/log/synapse_health.log 2>&1
```

### 8.3 Key Metrics to Watch

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| Redis connection | `/synapse/health` | `redis_connected: false` |
| Active sessions | `/synapse/health` | `active_count > 3` |
| Service uptime | `systemctl status` | Restart count > 3/hour |
| Disk usage | `df -h` | > 80% capacity |
| Session messages | `session.json` | `messages_count > 150` |

---

## 9. Backup and Recovery

### 9.1 What to Back Up

| Data | Location | Priority |
|------|----------|----------|
| Session directories | `$SYNAPSE_WORKSPACE/SYNAPSE_SESSION_*` | HIGH |
| SYNAPSE specifications | `$SYNAPSE_WORKSPACE/SYNAPSE_SPEC/` | HIGH |
| Environment files | `*.env` | CRITICAL |
| Service configurations | `/etc/systemd/system/synapse-*.service` | MEDIUM |

### 9.2 Backup Script

```bash
#!/bin/bash
# /opt/synapse/scripts/synapse_backup.sh

BACKUP_DIR="/opt/synapse/backups"
DATE=$(date +%Y%m%d_%H%M%S)
TARGET="${BACKUP_DIR}/synapse_backup_${DATE}"

mkdir -p "$TARGET"

# Copy session data
cp -r "$SYNAPSE_WORKSPACE"/SYNAPSE_SESSION_* "$TARGET/" 2>/dev/null
cp -r "$SYNAPSE_WORKSPACE"/SYNAPSE_SPEC/ "$TARGET/" 2>/dev/null

# Copy configs (without secrets)
cp /etc/systemd/system/synapse-*.service "$TARGET/" 2>/dev/null

echo "Backup completed: $TARGET"

# Cleanup: keep last 7 days
find "$BACKUP_DIR" -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;
```

### 9.3 Recovery

```bash
# 1. Restore services
sudo cp backup/synapse-*.service /etc/systemd/system/
sudo systemctl daemon-reload

# 2. Restore session data
cp -r backup/SYNAPSE_SESSION_* "$SYNAPSE_WORKSPACE"/

# 3. Restore .env files manually (from secure storage)

# 4. Restart all services
sudo systemctl restart redis-server synapse-agent-a synapse-agent-b

# 5. Verify
curl http://localhost:8000/synapse/health
```

---

## 10. Troubleshooting

### Common Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| `redis_connected: false` | Redis not running | `sudo systemctl start redis-server` |
| `agent_b_subscribed: false` | Bridge not connected | `sudo systemctl restart synapse-agent-b` |
| Session stuck in PAUSED | Manual pause or timeout | `/synapse resume` via Telegram |
| Messages not delivered | Redis pub/sub desync | Restart both services |
| Journal locked | Stale lock file | Remove `/tmp/synapse_journal.lock` |
| Agent B timeout | Response > 1800s | Check Agent B CLI, increase timeout |
| Email delivery failed | Gmail auth expired | Regenerate app password |
| Max sessions reached | 3 concurrent limit | Complete or cancel existing sessions |

### Service Recovery Procedure

```bash
# Full restart sequence (correct order)
sudo systemctl restart redis-server
sleep 2
sudo systemctl restart synapse-agent-a
sleep 2
sudo systemctl restart synapse-agent-b

# Verify
systemctl status redis-server synapse-agent-a synapse-agent-b --no-pager
curl -s http://localhost:8000/synapse/health | python3 -m json.tool
```

### Debug Mode

To run a service in foreground with verbose output:

```bash
# Stop the systemd service first
sudo systemctl stop synapse-agent-a

# Run manually with debug
cd /opt/synapse/agent_a
source venv/bin/activate
PYTHONUNBUFFERED=1 python api.py
```

---

## 11. Production Checklist

Before going live:

- [ ] All services configured with `Restart=always`
- [ ] Services enabled on boot (`systemctl enable`)
- [ ] Redis bound to localhost only
- [ ] Dangerous Redis commands disabled
- [ ] `.env` files have `chmod 600`
- [ ] Session directories have `chmod 700`
- [ ] Firewall configured (only SSH exposed)
- [ ] Health check script in place
- [ ] Backup script scheduled (daily cron)
- [ ] Telegram notifications tested
- [ ] Email delivery tested (if enabled)
- [ ] `/synapse/health` returns all green
- [ ] Test session runs end-to-end successfully

---

**Current deployment**: Ubuntu 24.04 LTS, Python 3.12.3, Redis 7.0.15, single VPS (96 GB disk, 42% used)
