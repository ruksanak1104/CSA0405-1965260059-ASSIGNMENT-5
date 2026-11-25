
# TASK A2 – Design the System Administration (5 Marks)

## A2.1 Users, Services and Logs (2 Marks)

| **Item** | **Design** |
|---|---|
| Groups | **admins** (sudo, full control), **devops** (deploy, restart services via sudo rules), **portal** (service group, no login), **auditors** (read-only log access) |
| Users | Named personal accounts (no shared accounts, no direct root login); **portal** system user (no shell, home `/opt/portal`) runs the application |
| sudo policy | `%admins ALL=(ALL) ALL`; `%devops` may run only `systemctl restart portal`, `systemctl reload nginx` (via `/etc/sudoers.d/devops`) |
| Passwords / accounts | SSH keys only; `passwd -l` on unused accounts; password ageing with `chage` |
| systemd services | **portal.service** (Gunicorn, `User=portal`, `Restart=on-failure`, `RestartSec=3`, `After=network.target`), **nginx.service**, **database service**, **portal-backup.timer** (nightly) |
| Log monitoring | journald for service logs (`journalctl -u portal -f`); `/var/log/nginx/access.log` and `error.log`; logrotate weekly, 12 weeks retained; hourly log-analysis script (B1.2); alert on 5xx rate and failed SSH logins (`/var/log/auth.log`); fail2ban bans repeated SSH failures |

### Example Unit File (Used in B1.3)

```ini
[Unit]
Description=University Result Portal (Gunicorn)
After=network.target

[Service]
User=portal
Group=portal
WorkingDirectory=/opt/portal
ExecStart=/opt/portal/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 app:app
Restart=on-failure
RestartSec=3
NoNewPrivileges=true
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

## A2.2 Networking Services (2 Marks)

| **Component** | **Design** |
|---|---|
| IP addressing | Private subnet `10.10.0.0/24` in the server room. Web server `10.10.0.10`, application server `10.10.0.11`, database `10.10.0.12` (database on a separate, non-routable VLAN `10.10.1.0/24`); static addresses via netplan; default gateway `10.10.0.1`; public address `203.0.113.10` (documentation range used as example) NAT-ed to the reverse proxy only |
| DNS | `results.university.edu` A record → `203.0.113.10` (TTL lowered to 60 s before result day so traffic can be moved quickly); internal resolver for `*.internal` names; CNAME for the cloud burst endpoint |
| Reverse proxy (Nginx) | Terminates TLS (Let's Encrypt certificate), serves static files and caches them (`proxy_cache`), rate-limits (`limit_req_zone`), forwards dynamic requests with `proxy_pass http://127.0.0.1:8000`; upstream block can list several app servers for load balancing |
| Firewall (ufw) | Default deny incoming, allow outgoing; allow `22/tcp` (SSH, restricted to the admin subnet), `80/tcp`, `443/tcp`; database port `5432` allowed only from `10.10.0.11` |
| SSH with keys | `PasswordAuthentication no`, `PermitRootLogin no`, `PubkeyAuthentication yes`, `AllowGroups admins devops`; Ed25519 keys (`ssh-keygen -t ed25519`); fail2ban |

### UFW Firewall and Nginx Configuration

```bash
# UFW rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

**Nginx configuration:**

```nginx
# /etc/nginx/sites-available/portal

server {
    listen 80;
    server_name results.university.edu;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## A2.3 Automation (1 Mark)

| **Routine task** | **Tool / Schedule** | **What it does** |
|---|---|---|
| Backups | `backup.sh` – cron `0 2 * * *` | `pg_dump`/SQLite copy + tar of `/opt/portal` to `/backup`, keep 7 daily, copy off-site, verify checksum |
| Log rotation | logrotate (weekly) + cron cleanup | Compress logs, delete after 12 weeks, signal Nginx to reopen logs |
| Disk-space alert | `disk_alert.sh` – cron `*/15 * * * *` | Alerts when any filesystem ≥ 80% (script in B1.2) |
| Log analysis | `log_summary.sh` – cron `0 * * * *` | Hourly count of status codes, busiest client, 5xx rate |
| Security updates | unattended-upgrades | Automatic patching of the OS |
| Health check | cron every minute: `curl -fs localhost/health || systemctl restart portal` | Self-healing in addition to systemd `Restart=` |

## Alternative Concept and Complexity Analysis – Ansible Instead of Handwritten Shell Scripts

With Ansible, the desired state of every server is described in YAML playbooks (users, packages, services, ufw rules, Nginx configuration). Ansible connects over SSH, is agentless and **idempotent** (running it twice changes nothing the second time). A single command – `ansible-playbook -i inventory site.yml` – configures any number of servers.

### Ansible Playbook (site.yml – Extract)

```yaml
- hosts: portal_servers
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Allow HTTP, HTTPS, SSH
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: [22, 80, 443]

    - name: Enable portal service
      ansible.builtin.systemd:
        name: portal
        enabled: true
        state: started
```

### Comparison: Shell Scripts vs Ansible

| **Criterion** | **Handwritten shell scripts** | **Ansible** | **Evaluation** |
|---|---|---|---|
| Set-up time, 1 server | ≈ 30–60 min to write and test; reusable later | ≈ 1.5–2 h (install, inventory, playbook, test) | Shell wins for a single server |
| Set-up time, 20 servers | ≈ 20 × 10 min manual runs ≈ 3+ h, plus fixes per host | ≈ 5–10 min (parallel forks, forks = 5–20) | Ansible wins clearly |
| Risk of differences between servers (configuration drift) | High – scripts are not idempotent; manual edits creep in | Low – declared state is re-applied; check mode shows drift | Ansible |
| Effort to learn | Low if you know bash | Medium: YAML, modules, inventories, roles | Shell |
| Effort to maintain | Grows quickly (error handling, per-OS differences) | Modules handle OS differences; roles reusable | Ansible for fleets |

### Pseudocode – Configuring n Servers

```text
ALGORITHM ConfigureServers

INPUT:
    servers[1..n]
    t (time to configure one server)
    f (parallel forks)

OUTPUT:
    total_time

BEGIN
    1. // Shell script runs one server after another
       total_shell ← n × t

    2. // Ansible runs f servers at the same time
       total_ansible ← CEILING(n / f) × t + t_setup

    3. RETURN total_shell, total_ansible
END
```

**Complexity:**

- Shell, run sequentially, is **T(n) = O(n)**.
- Ansible with `f` forks is **O(n/f)** = O(n) for fixed `f`, but with a much smaller constant (for n = 20, f = 10: 2 rounds instead of 20).
- Space: both **O(1)** per server; Ansible stores an inventory of size **O(n)**.
- Configuration drift between servers is **O(n²)** pairs to check by hand, but **O(n)** with Ansible check mode.

