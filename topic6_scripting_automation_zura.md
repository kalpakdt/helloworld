# 🤖 Topic 6: Scripting & Automation — Zura Style

No CS degree circlejerk. Just scripts that save your ass from manual bullshit.

---

## ⚡ The Automation Stack — One Line

```
Bash → Daily grunt work   |   Korn → Unix legacy
Python → Heavy lifting     |   Ansible/Puppet → Config as code
```

Write scripts. Automate the repetitive crap. Or stay a button-click monkey.

---

## 🔴 Stage 1 — Bash (The King)

**90% of Linux automation is Bash. Know it or GTFO.**

### Bash Essentials
```bash
#!/bin/bash                             # shebang ALWAYS first line

# Variables
NAME="Boss"
HOSTNAME=$(hostname)
LOGFILE="/var/log/myscript.log"

# Conditionals
if [ "$HOSTNAME" == "prod-server" ]; then
    echo "Prod mode"
elif [ $DISK_USAGE -gt 90 ]; then
    echo "Disk full"
else
    echo "All good"
fi

# Loops
for server in server1 server2 server3; do
    ssh $server "df -h /"
done

# While loop (disk check example)
while true; do
    DISK=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
    if [ $DISK -gt 90 ]; then
        echo "ALERT: Disk $DISK%" | mail -s "Disk Full" admin@company.com
    fi
    sleep 300  # 5 min
done

# Functions
health_check() {
    uptime
    free -h
    df -hT
}

health_check
```

### Real-World Bash Scripts
```bash
# 1. Disk Alert Script
#!/bin/bash
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 90 ]; then
    echo "CRITICAL: Root disk at ${DISK_USAGE}%" | mail -s "Disk Alert" ops@team.com
fi

# 2. Server Health Check (multi-server)
#!/bin/bash
SERVERS=("192.168.1.10" "192.168.1.11" "192.168.1.12")
for SERVER in "${SERVERS[@]}"; do
    if ping -c1 -W2 $SERVER &>/dev/null; then
        echo "$SERVER UP"
        ssh $SERVER "uptime; free -h"
    else
        echo "$SERVER DOWN - ALERT!"
    fi
done

# 3. Log Rotation Killer (clean old logs)
#!/bin/bash
find /var/log -name "*.log" -mtime +30 -delete
find /var/log -name "*.gz" -mtime +90 -delete
```

**Bash Traps (Don't Fuck Up):**
- Quote variables: `"$VAR"` not `$VAR`
- Use `[[ ]]` not `[ ]` for tests
- `local var` in functions
- `set -euo pipefail` at top (exit on error, undefined vars, pipe fails)

---

## 🟢 Stage 2 — Korn Shell (ksh)

**Unix legacy. AIX/Solaris love it. Bash superset mostly.**

```bash
#!/bin/ksh

# Korn-specific goodies
typeset -i COUNT=0                     # integer var
typeset -A ASSOCIATIVE[foo]=bar        # assoc array (pre Bash4)
print -n "No newline"                  # printf without newline

# Select menu (Korn killer feature)
PS3="Pick a server: "
select SERVER in prod1 prod2 staging db; do
    if [[ -n $SERVER ]]; then
        echo "You picked $SERVER"
        break
    fi
done

# Job control (better than Bash)
ksh -l                                 # login shell
jobs                                  # background jobs
fg %1                                 # foreground job 1
```

**Korn vs Bash:**
| Feature | Korn | Bash |
|---|---|---|
| Assoc arrays | Native | Bash 4+ |
| Select menu | Native | Native |
| Job control | Better | Good |
| Default on | AIX/Solaris | Linux |

---

## 🟣 Stage 3 — Python (The Beast)

**When Bash chokes on JSON/XML/HTTP — Python eats it.**

### Python Sysadmin Scripts
```python
#!/usr/bin/env python3
import subprocess
import json
import smtplib
from email.mime.text import MIMEText

# Run shell command, get JSON output
def run_cmd(cmd):
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    return json.loads(result.stdout) if result.stdout else {}

# Disk check
disk = run_cmd("df -h --output=pcent / | tail -1")
usage = int(disk[0].strip('%'))
if usage > 90:
    msg = MIMEText(f"Disk usage: {usage}%")
    msg['Subject'] = 'Disk Alert'
    msg['From'] = 'monitor@server'
    msg['To'] = 'admin@company.com'
    with smtplib.SMTP('localhost') as s:
        s.send_message(msg)

# Multi-server ping via paramiko (SSH lib)
import paramiko
servers = ['192.168.1.10', '192.168.1.11']
for server in servers:
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        client.connect(server, username='root', key_filename='/root/.ssh/id_rsa')
        stdin, stdout, stderr = client.exec_command('uptime')
        print(f"{server}: {stdout.read().decode()}")
    except:
        print(f"{server}: DOWN")
    client.close()
```

**Python Libs for Admins:**
```
paramiko      ← SSH
psutil        ← system metrics (CPU/RAM/disk)
requests      ← HTTP APIs
boto3         ← AWS
ansible       ← Ansible from Python
jinja2        ← templates
yaml/json     ← config parsing
```

---

## 🔧 Stage 4 — Ansible/Puppet (Config Management)

**Don't script servers. Declare desired state.**

### Ansible (Boss's jam [file:1])
```yaml
# inventory.ini
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

# playbook.yml
---
- hosts: webservers
  tasks:
    - name: Install nginx
      dnf:
        name: nginx
        state: present
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
    - name: Copy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

```bash
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass
ansible all -m ping                   # test connectivity
ansible webservers -m setup           # gather facts
```

### Puppet (Declaratve)
```puppet
# site.pp
package { 'nginx':
  ensure => installed,
}

service { 'nginx':
  ensure  => running,
  enable  => true,
  require => Package['nginx'],
}

file { '/etc/nginx/nginx.conf':
  ensure  => file,
  content => template('nginx/nginx.conf.erb'),
  notify  => Service['nginx'],
}
```

---

## 📊 Stage 5 — Script Comparison

| Task | Bash | Korn | Python | Ansible |
|---|---|---|---|---|
| Disk alert | ✅ Easy | ✅ Easy | ✅ + Email | ✅ Playbook |
| Multi-server | SSH loop | SSH loop | paramiko | Native |
| JSON parsing | awk/sed | awk/sed | Native | Native |
| Config templates | ❌ No | ❌ No | jinja2 | Native |
| Idempotent | ❌ No | ❌ No | Manual | ✅ Yes |

---

## 🚨 Stage 6 — Debugging Scripts

```bash
# Bash debug
bash -x script.sh                     # trace execution
bash -n script.sh                     # syntax check
set -x                                # debug from here
trap 'echo DEBUG: line $LINENO' DEBUG

# Python debug
python3 -m pdb script.py              # debugger
python3 -u script.py                  # unbuffered output
print(f"DEBUG: {var}")                # everywhere

# Common errors
# Bash: [[ ]] vs [ ], quote vars, command substitution $( )
# Python: Indentation, f-strings (Python 3.6+), import errors
```

---

## 🧩 Key Best Practices

```
- Shebang: #!/bin/bash OR #!/usr/bin/env python3
- Logging: logger -t myscript "msg" OR print to file
- Idempotency: Check if already done → if [ -f /etc/marker ]; then exit; fi
- Error handling: set -euo pipefail (Bash), try/except (Python)
- Args: getopts (Bash), argparse (Python)
- Version control: Git EVERYTHING
```

---

## ⚡ Full Automation Decision Flow

```
Task too repetitive?
        │
        ├─ One server, simple? → Bash
        │
        ├─ Complex logic/JSON? → Python
        │
        ├─ Multi-server config? → Ansible
        │
        └─ Unix legacy? → Korn
```

---

**Zura's Scripting Interview Rules:**
- They WILL ask "Write a Bash script to alert on disk >90%." → df/awk/mail combo.
- They WILL ask "Bash vs Python — when Python?" → APIs, JSON, libraries, multi-language.
- They WILL ask "Make script idempotent." → Check state first, marker files.
- Boss: Your Wipro Ansible playbooks reduced provisioning 90% — drop that bomb [file:1].

---
*Zura's Topic 6 Scripting Sheet — Know this or get roasted in interviews* 🔥
