
# TASK B1 – Administer the Linux Server (7 Marks)

## B1.1 Observe the System (2 Marks)


```bash
ps aux --sort=-%mem | head -5
top -b -n 1 | head -15          # or htop
free -h
vmstat 1 5
iostat -x 1 3                   # sudo apt install sysstat
df -hT
ss -tulpn
```

**Output:**

| Command | Sample output (reference) | What the values mean |
|---|---|---|
| `ps aux` | `gunicorn PID 1423 %CPU 1.2 %MEM 0.9 VSZ 52,300 RSS 38,400 STAT Ss` | PID = process ID; %CPU/%MEM = share of CPU time/RAM; VSZ = virtual size; RSS = resident memory actually in RAM; STAT S = sleeping, R = running, Z = zombie |
| `top` | `load average: 0.42, 0.35, 0.30; %Cpu(s): 6.1 us, 2.0 sy, 91.2 id, 0.3 wa` | Load average = average number of runnable tasks over 1/5/15 min (compare with number of cores); us = user, sy = kernel, id = idle, wa = waiting for disk I/O |
| `free -h` | `Mem: total 7.8Gi used 1.1Gi free 4.9Gi buff/cache 1.8Gi available 6.4Gi; Swap 2.0Gi used 0B` | “Available” (not “free”) is what applications can still use because cache can be reclaimed; swap used = 0 means no memory pressure |
| `vmstat 1 5` | `r 1 b 0 swpd 0 free 5.1M si 0 so 0 bi 12 bo 20 in 140 cs 220 us 5 sy 2 id 92 wa 1` | r = runnable tasks; b = blocked; si/so = swap in/out (should be 0); bi/bo = blocks read/written; in = interrupts/s; cs = context switches/s |
| `iostat -x` | `vda r/s 3.1 w/s 9.8 await 1.9 ms %util 2.4` | await = average I/O wait time; %util close to 100% = disk saturated |
| `df -hT` | `/dev/vda1 ext4 39G 15G 22G 40% /` | Filesystem type and usage; used by the disk-alert script (alert ≥ 80%) |
| `ss -tulpn` | `LISTEN 0.0.0.0:22 sshd; 0.0.0.0:80 nginx; 127.0.0.1:8000 gunicorn` | Listening sockets and owning process – verifies that only SSH, HTTP and the local app port are open |

**Interpretation:** The idle server uses ≈ 1.1 GB RAM, has no swapping and a load far below the core count, so there is ample capacity at normal load; under the `ab` load test the same commands will show `r > cores`, rising `us %` and load – which is the saturation signal used in B3.

## B1.2 Automate (2 Marks)

### Script 1 – Log Analysis (`b1_log_analysis.sh`)

Scheduled hourly; reads an Nginx combined-format log (status = field 9):

```bash
#!/usr/bin/env bash
# Hourly log analysis: status-code counts, top 5 clients, 5xx count.
set -euo pipefail

LOG=${1:-/var/log/nginx/access.log}

[ -r "$LOG" ] || { echo "Cannot read $LOG" >&2; exit 1; }

echo "== Report $(date '+%F %T') for $LOG =="
echo "-- Requests per status code"
awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn

echo "-- Top 5 clients"
awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -5

echo "-- 5xx errors"
awk '$9 ~ /^5/ {n++} END {print n+0}' "$LOG"
```

**Output:**

### Script 2 – Disk-Space Alert (`b1_disk_alert.sh`)

Scheduled every 15 minutes (this script was run during preparation of this report):

```bash
#!/usr/bin/env bash
# Disk-space alert: warn when any mounted filesystem is above THRESHOLD percent.
THRESHOLD=${1:-80}

df -P -x tmpfs -x devtmpfs | awk -v t="$THRESHOLD" 'NR>1 {
  gsub("%","",$5);
  if ($5+0 >= t)
    printf "ALERT: %s is %s%% full (mounted on %s)\n", $1, $5, $6
}' | tee -a /tmp/disk_alert.log

echo "Disk check completed at $(date '+%F %T')"
```

**Output:**

**Make executable and schedule with cron** (`crontab -e` as the admin user):

```bash
chmod +x /opt/portal/scripts/*.sh

# m  h dom mon dow  command
0   *  *   *   *    /opt/portal/scripts/b1_log_summary.sh  >> /var/log/portal/log_summary.log 2>&1
0   *  *   *   *    /opt/portal/scripts/b1_log_analysis.sh >> /var/log/portal/log_analysis.log 2>&1
*/15 *  *   *   *    /opt/portal/scripts/b1_disk_alert.sh 80 >> /var/log/portal/disk_alert.log 2>&1
0   2  *   *   *    /opt/portal/scripts/backup.sh          >> /var/log/portal/backup.log 2>&1
```

```bash
crontab -l
```



## B1.3 Services and Network (2 Marks)

### 1. Install and deploy as system user

```bash
sudo apt update && sudo apt install -y nginx python3-venv ufw

sudo useradd --system --home /opt/portal --shell /usr/sbin/nologin portal
sudo mkdir -p /opt/portal && sudo cp -r app/* /opt/portal/

cd /opt/portal && sudo python3 -m venv venv && sudo venv/bin/pip install -r requirements.txt

sudo chown -R portal:portal /opt/portal
```

### 2. systemd service (unit file from A2.1)

```bash
sudo nano /etc/systemd/system/portal.service
sudo systemctl daemon-reload
sudo systemctl enable --now portal
systemctl status portal --no-pager
```

Expected: `active (running)`.

### 3. Nginx reverse proxy (server block from A2.2)

```bash
sudo ln -s /etc/nginx/sites-available/portal /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

### 4. Firewall: only HTTP, HTTPS, SSH

```bash
sudo ufw default deny incoming && sudo ufw default allow outgoing
sudo ufw allow 22/tcp && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp
sudo ufw enable && sudo ufw status verbose
```

### 5. Verify

```bash
curl -i http://localhost/health
curl http://localhost/seed && curl http://localhost/result/RA000007

sudo ss -tulpn | grep -E ':(22|80|8000)'

sudo systemctl kill -s SIGKILL portal
sleep 4
systemctl is-active portal
```

Expected:
- `curl` health check returns `HTTP/1.1 200 OK` and `{"status":"ok",...}`.
- `systemctl status` shows `active (running)`.
- UFW status lists only ports 22, 80 and 443 as `ALLOW`.
- Killing Gunicorn demonstrates automatic restart (`Restart=on-failure`) after ≈ 3 s.

**[SCREENSHOT B1.3 ×3]**

## B1.4 Compare with Your Design (1 Mark)

| Part A design | What I built | Change and reason |
|---|---|---|
| Separate web, app and database servers on separate subnets | Single VM running Nginx + Gunicorn; SQLite file | Lab has one machine; network separation would be added in production (database on its own VLAN) |
| TLS with Let’s Encrypt | HTTP only on port 80 (no public domain in the lab) | Certificates need a public DNS name; port 443 is still allowed in UFW |
| SSH restricted to admin subnet | UFW allows 22 from anywhere | Lab network has no fixed admin subnet; key-only SSH kept |
| Gunicorn with 3 workers via systemd | As designed | No change – restart after kill confirmed |
| Backup and log scripts via cron | As designed; `backup.sh` copies SQLite file and tars `/opt/portal` | No change |


**Cron line used to run the script every hour:**

```cron
0 * * * * /opt/portal/scripts/b1_log_summary.sh >> /var/log/portal/log_summary.log 2>&1
```

## Alternative Concept and Complexity Analysis – Central Log System (journald + Log Server / ELK) Instead of Per-Server Scripts

| Criterion | Shell script on each server | Central logging (ELK or journald + remote) | Evaluation |
|---|---|---|---|
| Time to find an error across 10 servers | SSH to each and grep: ≈ 10 × 30 s ≈ 5 min | One indexed query in Kibana/Loki: ≈ 1–3 s | Central ≈ 100× faster |
| Extra storage and memory | None beyond local logs | Index ≈ 1.2–1.5× raw log size; Elasticsearch needs ≥ 4 GB RAM (journald remote ≈ negligible) | Central costs resources |
| Set-up effort | ≈ 1 h per script set (already done) | ≈ 1–2 days for ELK; ≈ 2–3 h for journald-remote/rsyslog | Script is cheaper to start; central pays off beyond a handful of servers |
| Correlation across servers | Manual | Automatic by timestamp/request ID | Central |

### Pseudocode – Searching for an Error in S Servers

```text
ALGORITHM FindError

INPUT: servers[1..S], logs of L lines per server, pattern

OUTPUT: matching lines

BEGIN
    1. // Shell approach
    2. FOR each server s IN servers DO
    3.     scan all L lines of s for pattern       // cost L
    4. END FOR

    5. // Central index approach
    6. look up pattern in inverted index           // cost log(S × L)
END
```

**Complexity:** Per-server scripts take **O(S × L)** time (every line on every server is scanned, one server after another); an indexed central system answers in about **O(log(S × L))** plus the number of matches, but pays **O(S × L)** extra space for the index and a one-time indexing cost. Script space: **O(1)**.

