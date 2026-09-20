# Self-Hosted Frappe Press on Ubuntu 24.04 — Two-Server Setup

A complete, from-scratch guide to deploying Frappe Press on **two servers** instead of the
standard four, on Ubuntu 24.04 (Noble), without AWS Route 53 or cloud provider API access.

Every fix in this guide was hit in practice. They are listed in the order they occur, so you
can apply them pre-emptively instead of discovering them one failed playbook at a time.

---

## Table of contents

- [Why this guide exists](#why-this-guide-exists)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Part 1 — Server preparation](#part-1--server-preparation)
- [Part 2 — Install Press on Server 1](#part-2--install-press-on-server-1)
- [Part 3 — Ubuntu 24.04 compatibility fixes](#part-3--ubuntu-2404-compatibility-fixes)
- [Part 4 — Production setup and dashboard SSL](#part-4--production-setup-and-dashboard-ssl)
- [Part 5 — SSH chain](#part-5--ssh-chain)
- [Part 6 — Root Domain](#part-6--root-domain)
- [Part 7 — Wildcard TLS certificate](#part-7--wildcard-tls-certificate)
- [Part 8 — Proxy Server](#part-8--proxy-server)
- [Part 9 — Unified Server (App + Database)](#part-9--unified-server-app--database)
- [Part 10 — Apps, sources and releases](#part-10--apps-sources-and-releases)
- [Part 11 — Release Group](#part-11--release-group)
- [Part 12 — Docker registry](#part-12--docker-registry)
- [Part 13 — Deploy Candidate](#part-13--deploy-candidate)
- [## Part 14 — Add Site to Proxy (Upstream Registration)]
- [Troubleshooting reference](#troubleshooting-reference)
- [Production checklist](#production-checklist)

---

## Why this guide exists

The official Press guides assume four separate servers, a real AWS account with Route 53, and
IAM credentials that let Press provision machines for you. This guide covers the case where:

- You have **two servers you provisioned yourself** (any provider — Contabo, Hetzner, AWS, bare metal)
- You have **no Route 53** and no cloud provider API access
- You are on **Ubuntu 24.04**, which several parts of the Press codebase do not yet expect

### A note on the official guide's age

The widely-referenced community guide carries this warning at the top:

> Edit 1: this will work for Press before the below commit on develop branch...
> Commit Hash: `7c2069324ea20c40fd54e2d37a42e08ec3179a27` (Aug 12, 2025)

After that commit, server forms no longer take IP and Private IP directly — they are pulled from
a `Virtual Machine` doctype that only exists when Press provisions the machine itself. This is
why **`is_self_hosted` must be checked** on every server doc in this guide, even though the
official guide never mentions it. See [Part 8](#part-8--proxy-server) for the mechanics.

---

## Architecture

Press has four roles. They do **not** require four machines, but they cannot be naively stacked
either — each agent-based role writes to the same fixed paths (`/home/frappe/agent`,
`/etc/nginx/nginx.conf`, `/etc/nginx/conf.d/agent.conf`), so running two of them on one IP via
separate "Setup Server" runs will collide.

Press solves this officially with `unified_server.yml`, which runs the App and Database roles in
a **single coordinated Ansible play** rather than two separate ones.

| Server | Roles | Why it works |
|---|---|---|
| **Server 1** | Press dashboard + Proxy Server | Dashboard runs as user `press` under `/home/press/frappe-bench` (a plain bench, no agent). Proxy runs as user `frappe` under `/home/frappe/agent`. Different users, different paths — no collision. |
| **Server 2** | App + Database (Unified) | Handled by `unified_server.yml` as one play. |

### Recommended sizing

| Setup | Spec | Notes |
|---|---|---|
| Two servers | 6 vCPU / 12 GB each | Recommended. Total 12 cores. |
| One server | 8 vCPU / 24 GB | Possible but not advised — single point of failure, DB competes with app workloads. |

### Provider considerations

If using a budget VPS provider rather than AWS, the factor that matters most is **disk I/O on
the database server**, since it directly affects response time for every hosted site. CPU on
oversold shared hosts is the second concern. Neither blocks the setup; both affect production
performance.

---

## Prerequisites

- **2 servers**, Ubuntu 24.04 LTS, root/sudo access, static IPs
- **A registered domain** with DNS record control. You need two names:
  - `press.example.com` → Server 1 (the dashboard)
  - `*.sites.example.com` → Server 2 (customer sites, wildcard)
- **Open ports** on both servers: `22, 80, 443, 3306, 3022, 8000`
- **An SSH keypair** you control, usable on both servers

> **DuckDNS note:** if testing with DuckDNS, register two separate domains (e.g. `mypress` and
> `mypresssites`). The subdomain field accepts only `A-Z`, `0-9`, `-` — no dots. DuckDNS resolves
> any subdomain under a registered domain to the same IP automatically, so
> `n1.mypresssites.duckdns.org` works without separate registration.

---

## Part 1 — Server preparation

Do this on **both** servers.

### 1.1 Swap (prevents OOM during Docker builds)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 1.2 UID 1000 must be free

Press creates a `frappe` user with UID 1000. On Ubuntu cloud images, the default user often
already holds it.

```bash
getent passwd 1000        # see who holds it
```

If the default user (`ubuntu`) holds UID 1000, move it. **Run this from a session that is not
logged in as that user**, or you will kill your own connection:

```bash
sudo loginctl terminate-user ubuntu
sudo pkill -9 -u ubuntu
sudo usermod -u 1002 ubuntu
sudo find /home/ubuntu -uid 1000 -exec chown ubuntu {} +

getent passwd 1000        # must now be empty
```

> Pick a UID that is actually free. If you create the `press` user first it may take 1001, in
> which case use 1002 for `ubuntu`.

### 1.3 sshd alias

Several Press playbooks reference `sshd.service`. On Ubuntu the unit is `ssh.service`.

```bash
sudo ln -sf /lib/systemd/system/ssh.service /etc/systemd/system/sshd.service
sudo systemctl daemon-reload
```

> Anywhere the official guide says `systemctl restart sshd`, use `systemctl restart ssh` on
> Ubuntu, or it fails with "Unit not found."

---

## Part 2 — Install Press on Server 1

### 2.1 Create the `press` user

```bash
sudo adduser press
sudo usermod -aG sudo press
sudo su - press

whoami    # must print: press
pwd       # must print: /home/press
```

**Everything in Part 2 and Part 3 runs as `press`.**

### 2.2 Base packages

```bash
timedatectl set-timezone "Africa/Cairo"   # your timezone

sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get install -y git python3-dev python3-pip python3-setuptools python3-venv
sudo apt-get install -y software-properties-common
sudo apt-get install -y mariadb-server mariadb-client
sudo apt-get install -y redis-server
sudo apt-get install -y xvfb libfontconfig libmysqlclient-dev pkg-config
```

### 2.3 wkhtmltopdf

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
sudo apt install ./wkhtmltox_0.12.6.1-2.jammy_amd64.deb -y
wkhtmltopdf --version    # must say "(with patched qt)"
```

### 2.4 MariaDB configuration

```bash
sudo nano /etc/mysql/my.cnf
```

```ini
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

```bash
sudo service mysql restart
sudo mysql_secure_installation
```

Answer as follows:

| Prompt | Answer | Reason |
|---|---|---|
| Switch to unix_socket authentication? | **n** | See warning below |
| Change the root password? | **Y** | Save it — `bench new-site` needs it |
| Remove anonymous users? | **Y** | |
| Disallow root login remotely? | **Y** | |
| Remove test database? | **Y** | |
| Reload privilege tables? | **Y** | |

> **This contradicts the official ERPNext guide, deliberately.** Answering `Y` to unix_socket
> runs `ALTER USER 'root'@'localhost' IDENTIFIED VIA unix_socket`, which makes root accept logins
> **only** when the OS user is root. `bench` runs as `press` and connects with a password, so it
> gets `MySQL Access Denied (1698)`. MariaDB 10.4+ already accepts both methods by default;
> saying `Y` explicitly closes the password path.

### 2.5 Node 20 (not 18)

```bash
cd ~
curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
source ~/.profile
nvm install 20
nvm alias default 20
nvm use 20
sudo apt-get install npm -y
sudo npm install -g yarn
node --version    # must be v20.x
```

> The official guides say Node 18. Current Press `develop` pulls `@vitejs/plugin-vue@6.x`, which
> requires `^20.19.0 || >=22.12.0`. With Node 18 `bench get-app press` fails at `yarn install`.

### 2.6 Bench and site

```bash
sudo pip3 install frappe-bench --break-system-packages
bench --version

cd ~
bench init --frappe-branch version-15 frappe-bench
cd frappe-bench/
sudo chmod -R o+rx /home/press/

bench new-site press.example.com
# prompts for MySQL root password, then an Administrator password — save both

bench get-app press
```

**Do not run `install-app` yet.** Apply Part 3 first.

---

## Part 3 — Ubuntu 24.04 compatibility fixes

Apply all of these before `install-app press`. They are ordered as they would otherwise fail.

### 3.1 Dependency pins

```bash
cd ~/frappe-bench/apps/press
sed -i 's/"ansible==3.4.0",/"ansible>=9,<12",/' pyproject.toml
sed -i 's/"stripe~=2.56.0",/"stripe>=7,<8",/' pyproject.toml

cd ~/frappe-bench
./env/bin/pip install -e apps/press
```

### 3.2 TLS libraries

```bash
cd ~/frappe-bench
./env/bin/pip install "cryptography~=50.0.0" "pyOpenSSL~=26.4.0" --break-system-packages
```

### 3.3 Ansible plugin loader — the critical one

Without this, **every** `Setup Server` fails instantly with
`ModuleNotFoundError: No module named 'ansible_collections.ansible.builtin'` — before SSH is even
attempted. Ansible registers `ansible.builtin` as a synthetic collection only when
`init_plugin_loader()` is called before any `Playbook` object is constructed.

Check the import exists:

```bash
grep -n "init_plugin_loader" ~/frappe-bench/apps/press/press/runner.py
```

If only the import line appears, **the call itself is missing**. Add it as the first statement
inside `Ansible.__init__`:

```bash
nano ~/frappe-bench/apps/press/press/runner.py
```

```python
# at the top, with the other imports
from ansible.plugins.loader import init_plugin_loader

...

class Ansible:
    def __init__(self, server, playbook, user="root", variables=None, port=22):
        init_plugin_loader()          # <-- ADD THIS LINE
        self.server = server
        self.playbook = playbook
        ...
```

Verify:

```bash
sed -n '201,206p' ~/frappe-bench/apps/press/press/runner.py
```

The indentation must match `self.server = server` exactly (tabs, not spaces, in this file).

### 3.4 Install the app

```bash
cd ~/frappe-bench
bench --site press.example.com install-app press
```

If it fails, apply the matching fix below and retry.

**`urllib3.contrib.appengine` ImportError:**

```bash
perl -i -pe 's/^(\s+)import telegram\.vendor\.ptb_urllib3\.urllib3\.contrib\.appengine as appengine$/$1try:\n$1    import telegram.vendor.ptb_urllib3.urllib3.contrib.appengine as appengine\n$1except ImportError:\n$1    appengine = None/' ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py

perl -i -pe 's/^(\s+)import urllib3\.contrib\.appengine as appengine(\s+#.*)?$/$1try:\n$1    import urllib3.contrib.appengine as appengine$2\n$1except ImportError:\n$1    appengine = None/' ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py

sed -i 's/appengine\.AppEngineManager,/type(None),/' ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py
sed -i "s/if appengine\.is_appengine_sandbox():/if appengine and appengine.is_appengine_sandbox():/" ~/frappe-bench/env/lib/python3.12/site-packages/telegram/utils/request.py
```

**`sqlparse` `MAX_GROUPING_TOKENS` AttributeError:**

```bash
cd ~/frappe-bench
./env/bin/pip install --force-reinstall "sqlparse~=0.5.4"
```

### 3.5 MariaDB role — remove the dead 10.6 repository

The `mariadb` Ansible role adds a Rackspace repo for MariaDB 10.6, which has no build for Noble.
Ubuntu 24.04 ships 10.11 natively, so drop the repo tasks and let apt use the system version.

```bash
cd ~/frappe-bench/apps/press/press/playbooks/roles/mariadb/tasks
cp main.yml main.yml.bak

sed -i '/- name: Add MariaDB Repository Key/,/- name: Update APT Cache/{/- name: Update APT Cache/!d}' main.yml

head -10 main.yml   # should now start at "Update APT Cache"
```

Validate the YAML still parses:

```bash
python3 -c "import yaml; yaml.safe_load(open('main.yml')); print('YAML OK')"
```

> Also check the package list references `libmariadb3`, not `libmariadbclient18`. Recent Press
> versions already use the correct name.

---

## Part 4 — Production setup and dashboard SSL

```bash
cd ~/frappe-bench
bench --site press.example.com list-apps        # expect frappe + press
bench --site press.example.com enable-scheduler
bench --site press.example.com set-maintenance-mode off

sudo env "PATH=$PATH" bench setup production press
bench setup nginx
```

### 4.1 Supervisor link

```bash
sudo supervisorctl status
```

If empty:

```bash
sudo ln -sf /home/press/frappe-bench/config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart all
sleep 10
sudo supervisorctl status     # expect 7 processes RUNNING
```

### 4.2 Firewall

```bash
sudo ufw allow 22,25,143,80,443,3306,3022,8000/tcp
sudo ufw enable
```

### 4.3 Dashboard certificate — use bench, not `certbot --nginx`

```bash
bench config dnsmultitenant on
sudo bench setup lets-encrypt press.example.com
```

> **Do not use `sudo certbot --nginx`.** It restarts nginx with its own method while the original
> nginx process still holds port 80, producing:
> ```
> nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
> Encountered exception during recovery: MisconfigurationError: nginx restart failed
> ```
> `bench setup lets-encrypt` stops nginx first, obtains the cert, updates `site_config.json`,
> rebuilds the nginx config and starts it again.

Verify:

```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
```

Open `https://press.example.com/dashboard` and log in as `Administrator`.

---

## Part 5 — SSH chain

Press connects to every managed server as `root`, **by IP**. This includes Server 1 connecting to
itself for the Proxy role.

### 5.1 Enable root login on both servers

```bash
sudo nano /etc/ssh/sshd_config
```

```
PermitRootLogin yes
PasswordAuthentication no
```

```bash
sudo systemctl restart ssh
```

### 5.2 Remove the AWS root lock (AWS images only)

AWS Ubuntu AMIs ship `/root/.ssh/authorized_keys` with a forced command that refuses the login:

```
no-port-forwarding,no-agent-forwarding,no-X11-forwarding,
command="echo 'Please login as the user \"ubuntu\" rather than the user \"root\".';..."
ssh-rsa AAAA...
```

This produces `Please login as the user "ubuntu"...` and closes the connection, regardless of
`PermitRootLogin`. Replace it with a clean key:

```bash
sudo cp /home/ubuntu/.ssh/authorized_keys /root/.ssh/authorized_keys
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
sudo systemctl restart ssh
```

### 5.3 Put the private key on Server 1

Press initiates connections *from* Server 1, so the key must live there — not just on your laptop.

```bash
# from your laptop
scp -i ~/.ssh/yourkey.pem ~/.ssh/yourkey.pem ubuntu@SERVER1_IP:~/
```

```bash
# on Server 1
sudo cp ~/yourkey.pem /home/press/.ssh/
sudo chown press:press /home/press/.ssh/yourkey.pem
sudo chmod 400 /home/press/.ssh/yourkey.pem
```

### 5.4 SSH config on Server 1, as user `press`

Entries must be keyed **by IP**, because that is what Press dials — hostname aliases alone are
not enough.

```bash
nano /home/press/.ssh/config
```

```
Host SERVER1_IP
    User root
    IdentityFile ~/.ssh/yourkey.pem
    StrictHostKeyChecking no

Host SERVER2_IP
    User root
    IdentityFile ~/.ssh/yourkey.pem
    StrictHostKeyChecking no
```

### 5.5 Verify both paths

```bash
sudo su - press
ssh root@SERVER1_IP "whoami"    # loopback — must print root
ssh root@SERVER2_IP "whoami"    # must print root
```

Both must succeed with no password prompt before continuing.

---

## Part 6 — Root Domain

Create it in the UI at `https://press.example.com/app/root-domain/new`:

| Field | Value |
|---|---|
| Domain | `sites.example.com` |
| DNS Provider | `Generic` |
| Default Cluster | `Default` |
| Team | your team ID |
| Enabled | ✅ |
| AWS Access Key / Secret / Region | leave blank |

`Generic` skips all Route 53 automation. The AWS fields are visible but not required.

---

## Part 7 — Wildcard TLS certificate

Press will try to obtain this automatically and fail, because the automatic path requires
Route 53. Obtain it manually and insert the doc yourself.

### 7.1 Install the DNS plugin

Use your DNS provider's certbot plugin. For DuckDNS:

```bash
sudo snap install certbot-dns-duckdns
sudo snap set certbot trust-plugin-with-root=ok
sudo snap connect certbot:plugin certbot-dns-duckdns
```

> Install as a **snap**, not pip. The snap-confined certbot cannot see pip-installed plugins.

### 7.2 Request the certificate — force RSA

```bash
sudo certbot certonly \
  --non-interactive \
  --cert-name sites.example.com \
  --key-type rsa \
  --rsa-key-size 2048 \
  --authenticator dns-duckdns \
  --dns-duckdns-token YOUR_TOKEN \
  --dns-duckdns-propagation-seconds 60 \
  -d "*.sites.example.com" \
  --agree-tos \
  --email you@example.com \
  --config-dir /home/press/.certbot \
  --work-dir /home/press/.certbot/work \
  --logs-dir /home/press/.certbot/logs

sudo chown -R press:press /home/press/.certbot
```

Verify the key type:

```bash
openssl rsa -in /home/press/.certbot/live/sites.example.com/privkey.pem -check -noout
# must print: RSA key ok
```

> **`--key-type rsa` is mandatory.** Modern certbot defaults to ECDSA. Press validates the key
> against its `rsa_key_size` field and rejects an ECDSA key with:
> ```
> ValidationError: Private key length does not match the selected RSA key size.
> Expected 2048 bits, got 256 bits.
> ```
> Pass the token inline with `--dns-duckdns-token`; the snap sandbox cannot read a credentials
> file. Also request the wildcard **alone** — DuckDNS supports only one TXT record per domain, so
> combining the wildcard and apex in one request fails with "Incorrect TXT record".

### 7.3 Insert the certificate doc

```bash
cd ~/frappe-bench
bench --site press.example.com console
```

```python
import frappe

base = "/home/press/.certbot/live/sites.example.com"
cert      = open(f"{base}/cert.pem").read()
chain     = open(f"{base}/chain.pem").read()
fullchain = open(f"{base}/fullchain.pem").read()
privkey   = open(f"{base}/privkey.pem").read()

doc = frappe.get_doc({
    "doctype": "TLS Certificate",
    "domain": "sites.example.com",
    "wildcard": 1,
    "provider": "Other",          # critical — see below
    "rsa_key_size": "2048",
    "team": "YOUR_TEAM_ID",
    "certificate": cert,
    "intermediate_chain": chain,
    "full_chain": fullchain,
    "private_key": privkey,
})
doc.insert(ignore_permissions=True)
frappe.db.commit()
print(doc.name, doc.status)       # expect: *.sites.example.com Active
```

> **`provider` must be `"Other"`, not `"Let's Encrypt"`.** The relevant code:
>
> ```python
> def after_insert(self):
>     self.obtain_certificate()       # runs on EVERY insert
>
> def _obtain_certificate(self):
>     if self.provider != "Let's Encrypt":
>         return                      # the safe exit
>     ...
>     ca.obtain(...)                  # needs Route 53 — fails, sets status = Failure
> ```
>
> With `"Let's Encrypt"`, `after_insert` triggers a real certbot run that fails and marks the doc
> `Failure`. Worse, a **scheduled renewal job** filters on `provider = "Let's Encrypt"` and will
> keep retrying on its own. With `"Other"`, `validate()` instead runs
> `validate_key_certificate_association()`, which verifies the key matches the certificate and
> sets `Active` — and the renewal job ignores the doc entirely.
>
> Never click **Obtain Certificate** or **Trigger Site Domain Callback** in the UI, and never
> plain `.save()` this doc — all three re-enter the failing path.

### 7.4 Renewal cron

```bash
sudo crontab -e
```

```
0 3 1 * * certbot certonly --non-interactive --force-renewal --cert-name sites.example.com --key-type rsa --rsa-key-size 2048 --authenticator dns-duckdns --dns-duckdns-token YOUR_TOKEN --dns-duckdns-propagation-seconds 60 -d "*.sites.example.com" --agree-tos --email you@example.com --config-dir /home/press/.certbot --work-dir /home/press/.certbot/work --logs-dir /home/press/.certbot/logs && chown -R press:press /home/press/.certbot
```

> Keep `--key-type rsa --rsa-key-size 2048` in the cron line, or renewal silently reverts to
> ECDSA. After each renewal you must also update the certificate content in the `TLS Certificate`
> doc — the files change but the doc does not follow automatically.

---

## Part 8 — Proxy Server

Create at `https://press.example.com/app/proxy-server/new`.

| Field | Value |
|---|---|
| Hostname | `n1` |
| Domain | leave blank |
| **Is Self Hosted** | ✅ |
| Self Hosted Server Domain | `sites.example.com` |
| Cluster | `Default` |
| Provider | `Generic` |
| Team | your team ID |
| IP | Server 1 public IP |
| Private IP | Server 1 private IP |
| Enabled Default Routing | ✅ |
| Is Primary | ✅ |
| SSH User | `root` |
| SSH Port | `22` |
| Nginx → Domains | `n1.sites.example.com` |

The doc saves as `n1.sites.example.com`.

### Why `is_self_hosted` is mandatory

```python
def autoname(self):
    if not self.domain:
        self.domain = frappe.db.get_single_value("Press Settings", "domain")
    self.name = f"{self.hostname}.{self.domain}"
    if self.doctype in ["Database Server", "Server", "Proxy Server"] and self.is_self_hosted:
        self.name = f"{self.hostname}.{self.self_hosted_server_domain}"

def after_insert(self):
    if self.ip:
        if self.doctype not in ["Database Server", "Server", "Proxy Server"] or not self.is_self_hosted:
            self.create_dns_record()          # needs Route 53
            self.update_virtual_machine_name() # needs a Virtual Machine doc
```

Without the flag, `after_insert` calls Route 53 and Virtual Machine code paths that do not exist
in a self-hosted setup. The flag also switches the playbook to the `self_hosted_*` variants.

> `self_hosted_server_domain` takes the **base domain only** (`sites.example.com`), not the full
> hostname. Press concatenates the hostname itself.

### 8.1 Run Setup Server

Click **Actions → Setup Server**, then work through these in order.

**`useradd: UID 1000 is not unique`** — [Part 1.2](#12-uid-1000-must-be-free) was missed on this
server. Fix and retry.

**`ModuleNotFoundError: No module named 'pkg_resources'`**

```bash
ssh root@SERVER1_IP
/home/frappe/agent/env/bin/pip install "setuptools<81" --break-system-packages
```

**nginx config missing or broken**

```bash
ssh root@SERVER1_IP
rm -f /etc/nginx/nginx.conf
apt-get install --reinstall --yes -o Dpkg::Options::="--force-confmiss" nginx-common
mkdir -p /home/frappe/agent
touch /home/frappe/agent/nginx.conf
chown -R frappe:frappe /home/frappe
ln -sf /home/frappe/agent/nginx.conf /etc/nginx/conf.d/agent.conf
nginx -t
systemctl start nginx
```

**`unknown directive "vhost_traffic_status_display"`**

```
nginx: [emerg] unknown directive "vhost_traffic_status_display"
       in /etc/nginx/conf.d/agent.conf:89
```

The agent nginx template includes a VTS monitoring block:

```nginx
location /status {
    auth_basic "NGINX VTS";
    auth_basic_user_file /home/frappe/agent/nginx/monitoring.htpasswd;
    vhost_traffic_status_display;
    vhost_traffic_status_display_format html;
}
```

`libnginx-mod-http-vhost-traffic-status` is **not packaged for Ubuntu 24.04**, and
`nginx-extras` does not bundle it. This `/status` endpoint is optional traffic monitoring only.

Locate the template on Server 1 and remove the block from its source:

```bash
grep -rln "vhost_traffic_status_display" ~/frappe-bench/apps/press/press/playbooks/
```

Edit the matching `.j2` template, delete the whole `location /status { ... }` block, then re-run
Setup Server. Removing it from `/etc/nginx/conf.d/agent.conf` alone is not enough — the agent
regenerates that file from the template.

> Restart bench after editing any Python or template file so workers pick up the change:
> ```bash
> cd ~/frappe-bench && bench restart
> ```

---

## Part 9 — Unified Server (App + Database)

This is the officially supported way to put the App and Database roles on one machine.

### How it works

```python
@frappe.whitelist()
def setup_unified_server(self):
    """Setup both the application server and its associated
       database server (unified plays on vm)."""
    ...
    database_server = frappe.get_doc("Database Server", self.database_server)
    self.status = "Installing"
    database_server.status = "Installing"
    ...
    ansible = Ansible(playbook="unified_server.yml", server=self, ...)
```

`unified_server.yml` runs App and DB roles in one play:

```yaml
roles:
  - role: nat_iptables      - role: essentials
  - role: user              - role: nginx
  - role: agent             - role: mount
  - role: bench             - role: docker
  - role: node_exporter     - role: cadvisor
  - role: statsd_exporter   - role: mariadb
  - role: mariadb_memory_allocator
  - role: mysqld_exporter   - role: deadlock_logger
  - role: filebeat          - role: clamav
  # ... plus hardening roles
```

Note it contains no proxy role — this merges **App + DB only**, which is why the Proxy stays on
Server 1.

### 9.1 Create the Database Server doc

At `https://press.example.com/app/database-server/new`:

| Field | Value |
|---|---|
| Hostname | `db` |
| **Is Self Hosted** | ✅ |
| Self Hosted Server Domain | `sites.example.com` |
| Cluster / Provider | `Default` / `Generic` |
| Team | your team ID |
| IP | Server 2 public IP |
| Private IP | Server 2 private IP |
| DB Port | `3306` |
| SSH User / Port | `root` / `22` |
| Self Hosted MariaDB Server IP | leave blank — auto-filled from Private IP |

### 9.2 Create the App Server doc

At `https://press.example.com/app/server/new`, using the **same IP as Server 2**:

| Field | Value |
|---|---|
| Hostname | `f1` |
| **Is Self Hosted** | ✅ |
| **Is Unified Server** | ✅ |
| Self Hosted Server Domain | `sites.example.com` |
| IP / Private IP | Server 2 public / private IP |
| Database Server | the `db.sites.example.com` doc |
| Proxy Server | the `n1.sites.example.com` doc |
| SSH User / Port | `root` / `22` |

### 9.3 Run it

On the **App Server** doc: **Actions → Setup → Setup Unified Server**.

The button appears when `is_unified_server` is checked:

```javascript
[ __('Setup Unified Server'), 'setup_unified_server', true, frm.doc.is_unified_server, __('Setup') ]
```

Expect the same fix sequence as Part 8.1 (`pkg_resources`, nginx, VTS) on this server too.

### 9.4 Mark the build server

```bash
bench --site press.example.com console
```

```python
frappe.db.set_value("Server", "f1.sites.example.com", "use_for_build", 1)
frappe.db.commit()
```

Also allow the agent to reach the Docker socket:

```bash
ssh root@SERVER2_IP "chmod 666 /var/run/docker.sock"
```

---

## Part 10 — Apps, sources and releases

Frappe must be the first app in any release group.

### 10.1 Create the App

`https://press.example.com/app/app/new`

| Field | Value |
|---|---|
| App Name | `frappe` |
| Title | `Frappe` |

### 10.2 Create the App Source

`https://press.example.com/app/app-source/new`

| Field | Value |
|---|---|
| App | `frappe` |
| App Title | `Frappe` |
| Repository URL | `https://github.com/frappe/frappe` |
| Branch | `version-15` |
| Team | your team ID |
| Public | ✅ |
| Versions (table) | `Version 15` |

### 10.3 Approve the App Release

An `App Release` is created automatically, but **it must be approved manually** or the deploy
candidate cannot use the code.

```python
releases = frappe.get_all("App Release", filters={"source": "SRC-frappe-001"},
                          fields=["name", "status"])
print(releases)

release = frappe.get_doc("App Release", "RELEASE_NAME")
release.status = "Approved"
release.save()
frappe.db.commit()
```

### 10.4 Repeat for other apps

Same three steps for `erpnext` (`https://github.com/frappe/erpnext`), `hrms`, etc. App names must
be lowercase and match the repo name.

---

## Part 11 — Release Group

`https://press.example.com/app/release-group/new`

Required fields: `title`, `version`, `team`, `apps`.

| Field | Value |
|---|---|
| Title | `Test Bench` |
| Version | `Version 15` |
| Team | your team ID |
| Apps (table) | row 1 `frappe`, row 2 `erpnext` |
| Servers (table) | `f1.sites.example.com` |

> Order matters — `frappe` first, then dependent apps. ERPNext depends on Frappe; it does not
> replace it.

---

## Part 12 — Docker registry

Built images must be pushed somewhere the app server can pull from. Press works with any registry
speaking Docker Registry API V2 — AWS ECR is not required.

`https://press.example.com/app/press-settings` → **Docker** tab:

| Field | Value (Docker Hub example) |
|---|---|
| Docker Registry URL | `docker.io` |
| Docker Registry Namespace | your Docker Hub account name |
| Docker Registry Username | your Docker Hub account name |
| Docker Registry Password | a Docker Hub **Access Token** |
| Build Server | `f1.sites.example.com` |
| Clone Directory | `/home/frappe/benches/clones` |
| Build Directory | `/home/frappe/benches/builds` |

Leave the Docker S3 keys blank.

> The namespace is **case-sensitive** and must match your account exactly, or the push fails with
> `denied: requested access to the resource is denied`.

Two unrelated fields on the same doctype are mandatory before it will save at all:

| Field | Value |
|---|---|
| Certbot Directory | `/home/press/.certbot` |
| EFF Registration Email | a real email |

---

## Part 13 — Deploy Candidate

From the Release Group: **Actions → Create Deploy Candidate**, then **Deploy** /
**Schedule Build and Deploy**.

Monitor from Server 2:

```bash
ssh root@SERVER2_IP "watch -n 3 'free -h; echo ---; docker ps'"
```

### `Run Validations` fails: `/usr/bin/python3.14` not found

```
FileNotFoundError: [Errno 2] No such file or directory: '/usr/bin/python3.14'
```

The agent's syntax check resolves a Python path like this:

```python
def get_python_path(dirpath: str) -> str:
    pyproject_path = os.path.join(dirpath, "pyproject.toml")
    if os.path.isfile(pyproject_path):
        with open(pyproject_path, "rb") as f, contextlib.suppress(Exception):
            pyproject_data = tomli.load(f)
            requires_python = pyproject_data.get("project", {}).get("requires-python")
            if requires_python:
                version_spec = sv.SimpleSpec(requires_python)
                if version_spec.match(sv.Version("3.14.0")):
                    python_path = shutil.which("python3.14")
                    if python_path:
                        return python_path
                    # Temporary hardcoding until python 3.14
                    return "/usr/bin/python3.14"
    return _get_server_python_path()
```

This asks *"could this constraint theoretically match 3.14?"* rather than *"is 3.14 installed?"*.
Frappe's `requires-python` is an open range like `>=3.10`, which does match 3.14 — so the agent
commits to a Python that does not exist and falls through to a hardcoded path.

The step only runs `compileall` for a syntax check — it never executes the code — so pointing it
at 3.12 is safe:

```bash
ssh root@SERVER2_IP
ln -sf /usr/bin/python3.12 /usr/bin/python3.14
python3.14 --version
```

Apply this on the **App Server**, where the agent runs.

### Build stuck on pending

Check `common_site_config.json` contains a `workers.build` block, and that the bench workers are
running:

```bash
sudo supervisorctl status
cd ~/frappe-bench && bench restart
```

### Build killed mid-way

Almost always OOM. Confirm swap is active:

```bash
ssh root@SERVER2_IP "free -h | grep -i swap"
```

---

## Part 14 — Add Site to Proxy (Upstream Registration)

[#part-14--add-site-to-proxy-upstream-registration](#part-14--add-site-to-proxy-upstream-registration)

Not covered by any official guide. Found by grepping the `Server` doctype's client script:

```
grep -n "add_upstream_to_proxy" ~/frappe-bench/apps/press/press/press/doctype/server/server.js
```

```
['Add to Proxy', 'add_upstream_to_proxy', true, frm.doc.is_server_setup && !frm.doc.is_upstream_setup, __('Network')]
```

This registers the App Server as an nginx upstream on the Proxy, so the Proxy's nginx knows where
to route traffic for that server's hostname. Without it, sites deployed on the App Server are
unreachable through the Proxy even if the site itself is `Active`.

### 14.1 Run it

[#141-run-it](#141-run-it)

From the **App Server** doc: **Actions → Add to Proxy**. If the button is not visible, run it
directly:

```
bench --site press.example.com console
```

```
server = frappe.get_doc("Server", "f1.sites.example.com")
server.add_upstream_to_proxy()
frappe.db.commit()
```

> `is_upstream_setup` may remain `0` even after this succeeds. In practice, sites still route
> correctly through the Proxy as long as the nginx upstream files below exist and the last
> "Add Upstream to Proxy" `Agent Job` shows `Success` — treat the flag as informational, not a
> hard gate.

### 14.2 `401 Unauthenticated` despite a correct `agent_password`

[#142-401-unauthenticated-despite-a-correct-agent_password](#142-401-unauthenticated-despite-a-correct-agent_password)

The `Agent` class authenticates with:

```
password = get_decrypted_password(self.server_type, self.server, "agent_password")
headers = {"Authorization": f"bearer {password}"}
```

The agent itself validates against a **pbkdf2_sha256 hash** stored in `access_token` inside its
own `config.json` — not the plaintext:

```
stored_hash = Server().config["access_token"]
if method.lower() == "bearer" and pbkdf2.verify(access_token, stored_hash):
    return None
```

If these two ever fall out of sync — e.g. after regenerating the password — you get a `401` even
though `get_password("agent_password")` and the value you set are identical strings.

**Regenerate both sides correctly:**

```
sudo /home/frappe/agent/env/bin/python3 -c "
from passlib.hash import pbkdf2_sha256
import secrets
p = secrets.token_hex(24)
print('PLAINTEXT:', p)
print('HASH:', pbkdf2_sha256.hash(p))
"
```

Write the hash into the agent's config using a **single-quoted heredoc**, never
`python3 -c "..."` with double quotes — bash expands every `$` in a pbkdf2 hash
(`$pbkdf2-sha256$29000$...`) as a shell variable and silently mangles the string:

```
sudo tee /tmp/fix_token.py > /dev/null << 'PYEOF'
import json
path = '/home/frappe/agent/config.json'
with open(path) as f:
    data = json.load(f)
data['access_token'] = 'PASTE_HASH_HERE'
with open(path, 'w') as f:
    json.dump(data, f, indent=4)
print('done')
PYEOF
sudo python3 /tmp/fix_token.py
sudo rm /tmp/fix_token.py
sudo chown frappe:frappe /home/frappe/agent/config.json
```

Put the **plaintext** in the `Proxy Server` doc:

```
proxy = frappe.get_doc("Proxy Server", "n1.sites.example.com")
proxy.agent_password = "PASTE_PLAINTEXT_HERE"
proxy.save()
frappe.db.commit()
```

**Then kill the agent's gunicorn master, not just its supervisor entry:**

```
sudo pkill -9 -f "gunicorn --bind 127.0.0.1:25052"
sudo supervisorctl start agent:web
```

> `supervisorctl restart agent:web` is not enough. It only cycles the gunicorn *worker*
> processes; the *master* process (which pre-forked from the old config) survives untouched and
> keeps serving the stale `access_token` indefinitely. Confirm the fix took by checking the PIDs
> changed:
>
> ```
> ps aux | grep "agent.web" | grep -v grep
> ```
>
> Every PID's start time should be *now*, not the original setup date.

### 14.3 Verify the nginx upstream was actually written

[#143-verify-the-nginx-upstream-was-actually-written](#143-verify-the-nginx-upstream-was-actually-written)

```
sudo find /home/frappe/agent/nginx/upstreams -type f
sudo grep -n "SITE_NAME\|SERVER2_IP" /etc/nginx/conf.d/proxy.conf
```

You should see an `upstream` block pointing at Server 2, and a `map` entry pairing your site's
hostname to that upstream. `/etc/nginx/conf.d/proxy.conf` is generated and already wired into
`nginx.conf` via `include /etc/nginx/conf.d/*.conf;`.

> **Do not manually add** `include /home/frappe/agent/nginx/proxy.conf;` to `nginx.conf`. That
> path is a separate copy/source file for the same content the agent already writes into
> `/etc/nginx/conf.d/proxy.conf`. Including both produces `nginx: [emerg] "real_ip_header"
> directive is duplicate` and takes nginx down entirely.

---

## Part 15 — DNS: point the sites domain at Server 1, not Server 2

[#part-15--dns-point-the-sites-domain-at-server-1-not-server-2](#part-15--dns-point-the-sites-domain-at-server-1-not-server-2)

**The wildcard `*.sites.example.com` must resolve to Server 1's IP (the Proxy), not Server 2.**
This is easy to get backwards since Server 2 is where the sites' data and files actually live.

### Symptom

[#symptom](#symptom)

Visiting any site under the wildcard shows:

```
Are you lost?
This address does not point to a site on Frappe Cloud.
```

This page is served by whatever received the raw HTTPS connection when the hostname isn't one it
recognises. If the wildcard resolves straight to Server 2, the browser bypasses the Proxy
entirely and lands on Server 2's own agent nginx (or Frappe's own "unknown site" fallback), which
has no reason to know about routing rules that only exist in the Proxy's config.

### Why pointing it at Server 1 doesn't break anything internal

[#why-pointing-it-at-server-1-doesnt-break-anything-internal](#why-pointing-it-at-server-1-doesnt-break-anything-internal)

Every internal connection Press and the agents make to each other already uses **raw IPs**, never
the wildcard hostname:

- Press → Agent: `Agent(self.proxy_server)` resolves via the `Server`/`Proxy Server` doc's `ip` field, not DNS.
- Proxy → App Server: the generated `upstream` block in `proxy.conf` uses Server 2's IP directly (see [14.3](#143-verify-the-nginx-upstream-was-actually-written)).
- SSH: dialed by IP per [Part 5](#part-5--ssh-chain).

The wildcard's DNS record exists **only** so a visitor's browser knows which server to open a
connection to in the first place. That must be the Proxy, since it's the only place holding
per-site routing rules and the wildcard TLS certificate.

### Fix

[#fix](#fix)

In your DNS provider (DuckDNS or otherwise), point the `sites.example.com` record at **Server
1's IP**, not Server 2's. If you're using DuckDNS with a wildcard-style setup where multiple
subdomains share one registered domain, there's only one IP for that domain — make sure it's
Server 1's.

Flush your local resolver cache and re-check before assuming it hasn't propagated:

```
nslookup somesite.sites.example.com
```

Expect Server 1's IP. If you have a stale record baked into `/etc/hosts` on Server 1 itself from
earlier troubleshooting (pointing the hostname at `127.0.0.1` as a workaround), it's safe to
remove once the real DNS record is corrected — but harmless to leave, since it only affects
Server 1's own outbound requests to that hostname.

---

## Part 16 — Static assets return 404 even though the files exist

[#part-16--static-assets-return-404-even-though-the-files-exist](#part-16--static-assets-return-404-even-though-the-files-exist)

### Symptom

[#symptom-1](#symptom-1)

The site's HTML loads (`200`), but every `/assets/...` request — CSS, JS, fonts — comes back
`404`, even though the files are confirmed present on disk with normal `644` permissions:

```
curl -s -o /dev/null -w "%{http_code}\n" https://mysite.sites.example.com/assets/frappe/dist/css/website.bundle.XXXXXXXX.css
# 404
```

### Cause

[#cause](#cause)

The site's nginx `location /assets { try_files $uri =404; }` block has `root
/home/frappe/benches/BENCH_NAME/sites;` — correct. But `/home/frappe` itself is created `750`
(`drwxr-x--- frappe frappe`), so only the `frappe` user and the `frappe` group can traverse into
it. nginx's worker processes run as `www-data`, which is in neither, so `www-data` cannot even
**enter** the directory tree — regardless of the target file's own permissions. `try_files`
reports this as `404`, not `403`.

Confirm with:

```
namei -l /home/frappe/benches/BENCH_NAME/sites/assets/frappe/dist/css/website.bundle.XXXXXXXX.css
```

Look for the `frappe` directory's mode in the output — `drwxr-x---` with `www-data` absent from
both owner and group is the tell.

### Fix

[#fix-1](#fix-1)

```
sudo usermod -aG frappe www-data
sudo systemctl restart nginx
```

> Use `restart`, not `reload` — the nginx worker processes need to be respawned to pick up the
> new group membership; a config reload alone does not re-evaluate the OS-level group a running
> worker belongs to.

This is a one-time fix per App Server; it isn't tied to any individual site or bench, so it will
not need repeating for future sites deployed on the same server.

---

## Troubleshooting reference — additions

[#troubleshooting-reference--additions](#troubleshooting-reference--additions)

| Symptom | Cause | Fix |
| --- | --- | --- |
| Site is `Active` but "Are you lost? This address does not point to a site on Frappe Cloud." | Wildcard DNS resolves to the App/DB server instead of the Proxy | Point the sites wildcard at Server 1 — [Part 15](#part-15--dns-point-the-sites-domain-at-server-1-not-server-2) |
| `Agent Job` fails with `401 Unauthenticated` even though the stored `agent_password` matches | Stale gunicorn **master** process still serving an old `access_token`; or the hash was corrupted by unquoted `$` in a `python3 -c "..."` call | `pkill -9` the agent gunicorn master, not `supervisorctl restart`; always write secrets via single-quoted heredocs — [14.2](#142-401-unauthenticated-despite-a-correct-agent_password) |
| `nginx: [emerg] "real_ip_header" directive is duplicate` | Manually added an `include` for `/home/frappe/agent/nginx/proxy.conf`, which is already included via `/etc/nginx/conf.d/proxy.conf` | Remove the manual include — [14.3](#143-verify-the-nginx-upstream-was-actually-written) |
| Site HTML loads but all CSS/JS return `404` despite files existing on disk | `/home/frappe` is `750`; `www-data` (nginx) can't traverse into it | `usermod -aG frappe www-data` + `systemctl restart nginx` — [Part 16](#part-16--static-assets-return-404-even-though-the-files-exist) |
| `Add to Proxy` / `add_upstream_to_proxy()` returns no exception but `is_upstream_setup` stays `0` | The flag is set asynchronously and isn't a reliable success signal | Verify via [14.3](#143-verify-the-nginx-upstream-was-actually-written) instead of the flag |

---

## Known open issues — additions

[#known-open-issues--additions](#known-open-issues--additions)

- **`is_upstream_setup` flag** — does not reliably reflect whether the upstream registration
  succeeded; verify against the actual generated nginx config instead.
- **Agent gunicorn master caching** — any change to `/home/frappe/agent/config.json` requires
  killing the master process outright (`pkill -9 -f "gunicorn --bind 127.0.0.1:25052"`), not a
  supervisor-level restart, or the change is silently ignored indefinitely.
- **`www-data` / `frappe` group separation** — a fresh unified server will need the
  `usermod -aG frappe www-data` fix applied once before the *first* site's assets will load.

## Troubleshooting reference

| Symptom | Cause | Fix |
|---|---|---|
| `yarn install` fails, engine incompatible | Node 18 | Node 20 — [2.5](#25-node-20-not-18) |
| `MySQL Access Denied (1698)` | unix_socket auth enabled | Answer `n` — [2.4](#24-mariadb-configuration) |
| `ModuleNotFoundError: ansible_collections.ansible.builtin` | `init_plugin_loader()` never called | [3.3](#33-ansible-plugin-loader--the-critical-one) |
| `urllib3.contrib.appengine` ImportError | telegram + urllib3 2.x | [3.4](#34-install-the-app) |
| `MAX_GROUPING_TOKENS` AttributeError | sqlparse version | [3.4](#34-install-the-app) |
| MariaDB repo 404 during setup | Rackspace 10.6 repo, no Noble build | [3.5](#35-mariadb-role--remove-the-dead-106-repository) |
| `bind() to 0.0.0.0:80 failed` during certbot | `certbot --nginx` restart collision | `bench setup lets-encrypt` — [4.3](#43-dashboard-certificate--use-bench-not-certbot---nginx) |
| `Unit sshd.service not found` | Ubuntu names it `ssh` | [1.3](#13-sshd-alias) |
| `Please login as the user "ubuntu"` | AWS forced-command root lock | [5.2](#52-remove-the-aws-root-lock-aws-images-only) |
| TLS cert status flips to `Failure` | `provider = "Let's Encrypt"` triggers Route 53 path | Use `"Other"` — [7.3](#73-insert-the-certificate-doc) |
| `Private key length does not match` | certbot issued ECDSA | `--key-type rsa` — [7.2](#72-request-the-certificate--force-rsa) |
| `useradd: UID 1000 is not unique` | default user holds UID 1000 | [1.2](#12-uid-1000-must-be-free) |
| `No module named 'pkg_resources'` | setuptools 81+ | [8.1](#81-run-setup-server) |
| `unknown directive "vhost_traffic_status_display"` | VTS module unavailable on Noble | [8.1](#81-run-setup-server) |
| `/usr/bin/python3.14` not found | agent version-resolution bug | [Part 13](#part-13--deploy-candidate) |
| `denied: requested access to the resource` | registry namespace case mismatch | [Part 12](#part-12--docker-registry) |

---

## Production checklist

Beyond the servers themselves:

- [ ] **Domain** with DNS record control — needed for A records, the DNS-01 challenge every ~90 days, and per-customer subdomains
- [ ] **DNS API token** (scoped to DNS edit only) if you want automatic subdomain provisioning per customer
- [ ] **SMTP credentials** — Press sends signup, backup and failure notifications
- [ ] **S3-compatible backup storage** — access key, secret, bucket, region/endpoint
- [ ] **Stripe account** if billing customers through Press
- [ ] **Git access** (deploy keys or tokens) for any private app repositories
- [ ] **Container registry** credentials

### On DNS access

You do not need full registrar access. You need either someone who can add records on request,
or a scoped API token. The three things that require it:

1. A records pointing subdomains at your server IPs
2. TXT records for the wildcard certificate's DNS-01 challenge, repeated at every renewal
3. Automatic subdomain creation as customer sites are provisioned

---

## Known open issues

- **nginx VTS block** — removing it from the Ansible template is a local patch; it will return on
  a Press update until upstream makes the block conditional.
- **`python3.14` symlink** — a workaround for an upstream resolution bug, not a fix. Re-check
  after agent updates.
- **Certificate renewal** — the cron renews the files, but the `TLS Certificate` doc content must
  be refreshed separately. Worth scripting before the first renewal comes due.

---

## Credits

Built on top of the official Frappe Press installation guide and the Frappe Cloud local setup
documentation, with fixes added for Ubuntu 24.04 and self-hosted, non-Route-53 deployments.
