# 📊 Topic 5: Monitoring Tools — Zura Style

No vendor marketing bullshit. Just tools that actually catch fires before users notice.

---

## ⚡ The Monitoring Stack — One Line

```
Zabbix → Enterprise dashboards   |   Nagios → Plugin king
Prometheus → Time-series metrics  |   OpManager → GUI everything
```

Know at least one well. Know all four superficially.

---

## 🔴 Stage 1 — Nagios (The Classic)

**Plugin-based. Alert-first. Simple but powerful.**

```bash
# Install Nagios (RHEL/CentOS)
dnf install epel-release -y
dnf install nrpe nagios-plugins-all -y

# Core config structure
/etc/nagios/                           ← main config
├── nagios.cfg                         ← master config
├── objects/                           ← hosts, services, contacts
│   ├── hosts.cfg
│   ├── services.cfg
│   ├── contacts.cfg
│   └── templates.cfg
└── plugins-config/                    ← plugin configs

# Define a host
vim /etc/nagios/objects/hosts.cfg
# define host {
#     use             linux-server
#     host_name       myserver
#     alias           My Production Server
#     address         192.168.1.10
# }

# Define a service (check_disk)
# define service {
#     use                 generic-service
#     host_name           myserver
#     service_description Disk Usage - /
#     check_command       check_nrpe!check_disk
# }

# Verify config
nagios -v /etc/nagios/nagios.cfg       # validate config
systemctl restart nagios

# NRPE (Nagios Remote Plugin Executor — client side)
# Install on monitored servers
dnf install nrpe nagios-plugins-all
systemctl enable --now nrpe

# NRPE config (what checks to allow)
/etc/nagios/nrpe.cfg
# allowed_hosts=192.168.1.5        ← Nagios server IP
# command[check_disk]=/usr/lib64/nagios/plugins/check_disk -w 20% -c 10%
```

**Nagios Check Commands (know these):**
```
check_ping        ← connectivity
check_disk        ← disk usage
check_load        ← CPU load
check_swap        ← memory
check_http        ← web server
check_ssh         ← SSH connectivity
check_nrpe        ← remote checks via NRPE
```

---

## 🟢 Stage 2 — Zabbix (Enterprise Beast)

**Modern dashboards. Agent-based. Scales to thousands of servers.**

```bash
# Zabbix Agent install
dnf install zabbix-agent zabbix-sender -y
apt install zabbix-agent2             # Ubuntu (agent2 is newer)

# Agent config
vim /etc/zabbix/zabbix_agentd.conf
# Server=192.168.1.10                   ← Zabbix server IP
# ServerActive=192.168.1.10:10051       ← active checks
# Hostname=myserver                     ← register by hostname
# HostMetadata=linux,rhel9              ← tags/metadata

systemctl enable --now zabbix-agent
systemctl status zabbix-agent

# Zabbix Sender (send metrics manually)
zabbix_sender -z 192.168.1.10 -s "myserver" -k "disk.free" -o 123456789

# Common Zabbix Items (data collected)
system.uptime
vfs.fs.size[/,pfree]                  # disk free %
proc.num[cpu.user]
net.if.in[eth0]                       # network in
net.if.out[eth0]                      # network out
vm.memory.size[available]             # free memory
```

**Zabbix Dashboard Navigation:**
```
Configuration → Hosts → myserver → Items → Triggers → Graphs → Screens
Triggers → Actions → Media Types → Users → Notifications
```

---

## 🟡 Stage 3 — Prometheus (Metrics Time-Series)

**Cloud-native. Pull-based. Grafana best friend.**

```bash
# Prometheus is agentless — scrapes HTTP endpoints
# Node Exporter provides system metrics

# Install Node Exporter (system metrics)
dnf install prometheus2-node-exporter  # RHEL repo
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xzf node_exporter-*.tar.gz
cd node_exporter-*/
./node_exporter                        # runs on :9100

systemctl daemon-reload
systemctl enable --now prometheus-node-exporter

# Prometheus config (/etc/prometheus/prometheus.yml)
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['192.168.1.10:9100']
  - job_name: 'myserver'
    static_configs:
      - targets: ['192.168.1.11:9100']

# Common Prometheus Metrics (scrape /metrics endpoint)
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes{mountpoint="/"}
node_disk_io_time_seconds_total
node_network_receive_bytes_total

# Query Prometheus
curl http://prometheus:9090/api/v1/query?query=node_cpu_seconds_total
promtool query instant http://localhost:9090 node_cpu_seconds_total

# Prometheus Query Language (PromQL)
rate(node_cpu_seconds_total[5m])     # CPU usage last 5m
100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)  # memory %
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100  # disk free %
```

**Prometheus + Grafana Stack:**
```
Prometheus ← collects metrics
Grafana    ← pretty dashboards
Alertmanager ← sends alerts
Node Exporter ← Linux metrics
Blackbox Exporter ← HTTP/Ping/ICMP checks
```

---

## 🔵 Stage 4 — ManageEngine OpManager

**GUI-first. Agent + SNMP. Everything in dashboards.**

```
Core concept: Agentless (SNMP/WMI) + Agent-based monitoring.
Dashboards → Alarms → Inventory → Reports → Workflow Automation.
```

**Key Features:**
- **Network Discovery** — auto-discover switches, routers, servers
- **Server Monitor** — CPU, RAM, disk, process monitoring
- **Flow Monitor** — NetFlow/sFlow bandwidth analysis
- **Event Log Monitor** — Windows event logs
- **URL Monitor** — HTTP/S response time

**OpManager Ports:**
```
8060  → Web console (HTTP)
8061  → Web console (HTTPS)
161   → SNMP
1610  → SNMP Traps
```

---

## 📊 Stage 5 — Tool Comparison (Interview Gold)

| Feature | Nagios | Zabbix | Prometheus | OpManager |
|---|---|---|---|---|
| Agent | NRPE | Agent | None (Node Exporter) | Agent + SNMP |
| Scalability | Small-Medium | Large | Very Large | Medium |
| Dashboards | Basic | Excellent | Grafana | Excellent |
| Alerting | Email | Email/SMS | Alertmanager | Email/SMS |
| Cost | Free | Free | Free | Paid |
| Learning Curve | Medium | Steep | Steep | Easy |
| Best For | Simple | Enterprise | Cloud-native | GUI lovers |

---

## 🚨 Stage 6 — Monitoring Troubleshooting

```bash
# Agent not reporting?
# Nagios NRPE
systemctl status nrpe
netstat -tulnp | grep 5666            # NRPE port
nrpe -t                               # test config

# Zabbix Agent
systemctl status zabbix-agent
netstat -tulnp | grep 10050           # passive port
netstat -tulnp | grep 10051           # active port
zabbix_agentd -t                      # test config
zabbix_get -s 127.0.0.1 -k "system.uptime"  # test metric

# Prometheus Node Exporter
curl http://localhost:9100/metrics    # check if responding
netstat -tulnp | grep 9100
journalctl -u prometheus-node-exporter

# Firewall blocking?
firewall-cmd --list-all | grep 10050  # Zabbix
firewall-cmd --add-port=9100/tcp --permanent  # Prometheus
firewall-cmd --add-service=nagios --permanent
```

---

## 🛡️ Stage 7 — Critical Alerts to Set (Don't Skip)

```
🚨 CPU > 90% for 5 minutes
🚨 RAM < 10% free for 5 minutes
🚨 Disk > 90% full (separate alert per filesystem)
🚨 Disk I/O wait > 50%
🚨 Network errors > 1%
🚨 Service down (httpd, sshd, mysqld)
🚨 Ping loss > 1 minute
🚨 Log file growing > 100MB/hour
```

**Alert Escalation:**
```
Yellow (Warning) → 5 minutes → Email team
Red (Critical)   → 2 minutes → SMS + PagerDuty + Boss
```

---

## 🧩 Key Files Cheatsheet

```
# Nagios
/etc/nagios/nagios.cfg
/etc/nagios/objects/hosts.cfg
/etc/nagios/nrpe.cfg

# Zabbix Agent
/etc/zabbix/zabbix_agentd.conf
/var/log/zabbix/zabbix_agentd.log

# Prometheus
/etc/prometheus/prometheus.yml
/var/log/prometheus/prometheus.log

# Node Exporter
/etc/systemd/system/prometheus-node-exporter.service

# Logs
/var/log/messages                    # system logs
journalctl -u zabbix-agent           # systemd logs
```

---

## ⚡ Full Monitoring Decision Flow

```
Need monitoring?
        │
        ├─ Simple/small setup? → Nagios
        │
        ├─ Enterprise features? → Zabbix
        │
        ├─ Cloud-native/Grafana? → Prometheus
        │
        └─ GUI dashboards? → OpManager (budget permitting)
```

---

**Zura's Monitoring Interview Rules:**
- They WILL ask "Server shows 100% CPU but top shows 10% — explain." → Zombie processes, wait I/O, or monitoring agent double-counting.
- They WILL ask "How do you monitor disk space?" → `vfs.fs.size[/,pfree]` (Zabbix), `check_disk -w 20 -c 10%` (Nagios).
- They WILL ask "Prometheus vs Nagios?" → Prometheus is time-series metrics, Nagios is service status checks.
- They WILL ask "Agent down — what do you check first?" → Agent service status → firewall → config syntax → network connectivity.

---
*Zura's Topic 5 Monitoring Sheet — Know this or get roasted in interviews* 🔥
