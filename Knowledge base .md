# AlmaLinux ViciDial Knowledge Base

**Compiled from [dialer.one](https://dialer.one) and [carpenox/vicidial-install-scripts](https://github.com/carpenox/vicidial-install-scripts)**  
**Target: Alma Linux 9 as DB + Web server for ViciDial**

---

## 1. Alma Linux 9 – Initial Installation (DB + Web Server)

**Source:** [How to Install ViciDial on Alma Linux 9 with my new auto installer](https://dialer.one/index.php/how-to-install-vicidial-on-alma-linux-9-with-my-new-auto-installer/)

- Installer fixes DAHDI and PHP7 issues on Alma Linux 9; includes dynamic portal and CyburPhone.
- **Alma 9 ISO:** https://repo.almalinux.org/almalinux/9/isos/x86_64/AlmaLinux-9-latest-x86_64-dvd.iso

### Step 1 – Dependencies and clone (then reboot)

```bash
# Optional: English locale (recommended from GitHub README)
dnf install -y glibc-langpack-en
localectl set-locale en_US.UTF-8

timedatectl set-timezone America/New_York

yum check-update
yum update -y
yum -y install epel-release
yum update -y
yum install git -y
yum install kernel* --exclude=kernel-debug* -y

# Disable SELinux (required for ViciDial web)
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

cd /usr/src
git clone https://github.com/carpenox/vicidial-install-scripts.git
cd vicidial-install-scripts

reboot
```

### Step 2 – Run the main installer

- You must have a **FQDN** (Fully Qualified Domain Name) for SSL and webphones.
- The installer is interactive: confirm prompts and enter the server IP when asked.

```bash
chmod +x main-installer.sh
./main-installer.sh
```

- **Alma/Rocky 9 main installer:** Dynamic portal, WebPhone, SSL, Asterisk 18, Confbridges.  
- **Other scripts:** See [GitHub vicidial-install-scripts](https://github.com/carpenox/vicidial-install-scripts).

---

## 2. Post-Install – Full Functionality (DB + Web)

**Source:** [How to – Use the full functionality of the ViciDial installer by carpenox](https://dialer.one/index.php/how-to-use-the-full-functionality-of-the-vicidial-installer-by-carpenox/)

### Step 1 – Change default password

- After reboot, log in via **https://yourdomain.com** (or https://IP if no SSL).
- **Default login:** user `6666`, password `1234`.
- Change the password immediately, then in **Users** give your admin account full permissions.

**If you skipped SSL during install:**  
Turn firewall off before running certbot later: `service firewalld stop`, then start again after SSL is installed.

**If not using webphones / no SSL:**  
Log in via `https://YOUR_IP`, change password, and add HTTP + dynamic portal to firewall (see Firewalld section and “No domain” below).

### Step 2 – Lock down firewall

- Port 443 is left in the **public** zone so you can change the default password. After changing it, remove HTTPS from public:

```bash
firewall-cmd --permanent --remove-service=https
firewall-cmd --reload
```

- Update the dynamic portal redirect:
  - Edit: `nano /var/www/vhosts/dynportal/inc/defaults.inc.php`
  - Change `https://cyburdial.com/agc/cyburdial.php` to `https://yourdomain.com/agc/vicidial.php`
- After that, use the **dynamic portal** to validate your IP (see Step 2 in source).  
- Dynamic portal URL examples:  
  - With domain: `https://yourdomain.com:446/valid8.php`  
  - IP only: `http://YOUR_IP:81/valid8.php`

### Step 3 – ViciDial configuration

- Configure campaigns, carriers, DIDs, etc. as needed.
- **Standard DB (optional):** If you use `standard-db.sh`, after import add the letter **“a”** to the **phone login** field for each user (see post-install article).

### Step 4 – Domain and WebRTC

- In **Admin → Servers:** set the server’s **websocket** URL to your domain.
- In **Admin → Templates:** edit the “SIP_generic” WebRTC template with your domain.
- In **System Settings:** set the **audio store / sounds web server** URL to your domain.

### Step 5 – Audio store (when using standard-db)

```bash
cd /var/www/html
mkdir hgcjvmrjzqcngw47wf5zf4xjzd9n0k
chmod -R 755 hgcjvmrjzqcngw47wf5zf4xjzd9n0k
chown -R apache:apache hgcjvmrjzqcngw47wf5zf4xjzd9n0k
cp -R /var/lib/asterisk/sounds/* /var/www/html/hgcjvmrjzqcngw47wf5zf4xjzd9n0k
```

### Step 6 – DIDs and carrier

- **Admin → Reports → Admin Utilities → Bulk Tools:** use “Copy this DID” to paste DIDs in bulk.
- **Admin → Carriers:** set your carrier IP/trunk.

---

## 3. Database Scripts (DB Server)

**Source:** [GitHub README](https://github.com/carpenox/vicidial-install-scripts)

### Standard DB (ready-to-dial baseline)

- Pre-configured DB: campaigns, user groups, ingroups, recording/reporting defaults.  
- Password for user 6666 after import: **CyburDial2024**.  
- Still need to add “a” to phone login per user and configure DIDs + carrier.

```bash
cd /usr/src/vicidial-install-scripts
chmod +x standard-db.sh
./standard-db.sh
```

### Cluster DB (7 servers, 150 users)

```bash
cd /usr/src/vicidial-install-scripts
chmod +x cluster-db.sh
./cluster-db.sh
```

### Adding dialer servers to the cluster (DB server side)

- **Requirement:** The DB server must have **astguiclient** trunk (e.g. from main-installer). `add-dialer-to-DB.sh` sources **`/usr/src/astguiclient/trunk/extras/second_server_install.sql`** and inserts **vicidial_confbridges** for the dialer IP you enter. It uses placeholder `10.10.10.16` which it replaces with your input via `ADMIN_update_server_ip.pl`.

1. **On DB server:** add dialers to the database (conferences/confbridges):

```bash
cd /usr/src/vicidial-install-scripts
chmod +x add-dialer-to-DB.sh
./add-dialer-to-DB.sh
```

2. **On each dialer server:** link that server to the DB:

```bash
cd /usr/src/vicidial-install-scripts
chmod +x run-on-dialer-servers-cluster.sh
./run-on-dialer-servers-cluster.sh
```

Repeat steps 1 and 2 for each new dialer.

---

## 4. Building a ViciDial Cluster (step-by-step)

**Architecture:** One **DB+Web** server (MySQL + Apache + ViciDial web UI + dynamic portal) and **multiple Asterisk/dialer** servers (telephony only). All dialers connect to the single DB. Optional: use **cluster-db.sh** for a pre-built cluster (7 servers, 150 users) or build from **main-installer** + **add-dialer** as below.

**Order of operations:** Common prep → DB+Web server → allow MySQL from dialers → add first dialer (DB side, then dialer side) → add more dialers → ConfBridge / Web UI / Admin GUI.

---

### 4.1 Common for all servers

Apply these on **every** server (DB+Web and each Asterisk/dialer) before role-specific steps.

1. **Hostnames:** Use unique hostnames (e.g. `db1.dialerelite.com`, `dial1.dialerelite.com`). Each hostname becomes the server name in **Admin → Servers**. Set with `hostnamectl set-hostname hostname`.
2. **IPs:** Know the DB+Web IP and each dialer IP. Dialers must reach DB+Web on **3306**.
3. **FQDN:** One FQDN for the web UI (typically the DB+Web hostname) for SSL and WebRTC.
4. **Pre-requisites (see §1.1):** Timezone, `yum update`, epel-release, git, kernel packages, **SELinux disabled** in `/etc/selinux/config`, then reboot.
5. **Clone install scripts:**  
   `cd /usr/src && git clone https://github.com/carpenox/vicidial-install-scripts.git`  
   Then reboot if you just disabled SELinux.
6. **Cron:** Ensure **crond** is enabled and running so keepalive and other ViciDial cron jobs run:
   ```bash
   systemctl enable crond
   systemctl start crond
   ```

**Files relevant on all servers:** `/etc/selinux/config` (SELinux disabled), hostname, network.

---

### 4.2 DB+Web server only

**Do this on the one server that will be DB+Web.**

1. **Install:** From §4.1, run the **main installer** (see §1.2):
   ```bash
   cd /usr/src/vicidial-install-scripts
   chmod +x main-installer.sh
   ./main-installer.sh
   ```
   Use this server’s **FQDN** and **IP** when prompted. Reboot when finished.
2. **Disable Asterisk (DB+Web has no telephony):**  
   Stop now: `screen -S asterisk -X quit`  
   Edit **`/etc/rc.d/rc.local`**: comment out Asterisk log roll (`ADMIN_restart_roll_logs.pl`), DAHDI (modprobe/dahdi_cfg), `sleep 20`, and `start_asterisk_boot.pl` (see §11).
3. **Optional DB import (see §3):** `./standard-db.sh` or `./cluster-db.sh`.
4. **Allow MySQL from dialers:**  
   - In **`/etc/my.cnf.d/mariadb-server.cnf`** set `bind-address = 0.0.0.0`; then `systemctl restart mariadb`.  
   - Ensure MySQL users `cron` and `custom` can connect from dialer IPs (e.g. already `@'%'`).  
   - **Firewall:** Allow 3306 from each dialer IP (or subnet):
     ```bash
     firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="DIALER_IP" port port="3306" protocol="tcp" accept'
     firewall-cmd --reload
     ```
5. **Post-install:** Change default password (6666/1234), lock firewall (remove 443 from public after first login), set dynamic portal redirect (see §2).
6. **Cron / keepalive on DB+Web:** Main-installer installs a **root crontab** that includes `ADMIN_keepalive_ALL.pl` every minute. That script keeps AST_* screen processes running; on DB+Web there is no Asterisk, so processes that need Asterisk will not have a channel to talk to—that is expected. Leave cron as installed.
7. **Web UI and domain (see 4.6 below):** Create Apache vhost for FQDN, set dynamic portal and CyburPhone URLs.

**Files created/updated on DB+Web (main-installer):**

| File | Purpose |
|------|--------|
| `/etc/astguiclient.conf` | VARserver_ip, VARDB_*, paths, VARactive_keepalives |
| `/etc/rc.d/rc.local` | MariaDB, httpd, AST_reset_mysql_vars.pl; comment out Asterisk/DAHDI for DB+Web |
| `/etc/httpd/conf.d/viciportal.conf` | Dynamic portal vhost (port 81) |
| `/etc/httpd/conf.httpd.conf` | DocumentRoot, Alias RECORDINGS/MP3 |
| `/etc/my.cnf` or `/etc/my.cnf.d/mariadb-server.cnf` | MySQL; set bind-address for dialer access |
| `/etc/php.ini` | PHP settings |
| `/var/www/html/index.html` | Redirect to /vicidial/welcome.php |
| `/var/www/vhosts/dynportal/inc/defaults.inc.php` | Portal redirect URLs (set to your FQDN) |
| Root crontab | ADMIN_keepalive_ALL.pl and other AST_* cron jobs |

---

### 4.3 Asterisk/dialer servers only

**Do this on each server that will run Asterisk (telephony).** For the **first** dialer, run **4.3a** on DB+Web first, then **4.3b** on the new server. For **additional** dialers, see **4.4**.

#### 4.3a On DB+Web server (before each new dialer)

Run once **per** new dialer, **before** or right after installing that dialer:

```bash
cd /usr/src/vicidial-install-scripts
chmod +x add-dialer-to-DB.sh
./add-dialer-to-DB.sh
```
Enter MySQL root password when prompted, then the **dialer server’s IP**. The script loads `second_server_install.sql`, updates `vicidial_servers`, and inserts **vicidial_confbridges** for that IP.

#### 4.3b On the new Asterisk/dialer server (Alma 9, fresh)

1. **Pre-requisites and reboot** (§4.1).
2. **Install add-on dialer:**
   ```bash
   cd /usr/src/vicidial-install-scripts
   chmod +x addon-dialer-alma9.sh
   ./addon-dialer-alma9.sh
   ```
   When it finishes, **link this server to the DB:**
   ```bash
   chmod +x run-on-dialer-servers-cluster.sh
   ./run-on-dialer-servers-cluster.sh
   ```
   Enter the **DB+Web server’s IP**. This runs `install.pl --DB_server=DB_IP`, restarts Asterisk, and runs **ADMIN_keepalive_ALL.pl** to start the AST_* screen processes.
3. **Cron and keepalive (critical for Asterisk):**
   - **crond** must be enabled and running (see §4.1). The addon-dialer installs a **root crontab** that includes:
     - **`* * * * * /usr/share/astguiclient/ADMIN_keepalive_ALL.pl`** — runs every minute and keeps the AST_* screen sessions (AST_update, AST_send_listen, AST_VDauto_dial, AST_VDremote_agents, FastAGI_log, ip_relay, etc.) running. Without this, Asterisk-related processes can die and not restart.
   - To install or restore crontab: `crontab /root/crontab-file` (if the installer created it) or ensure the line above is in root’s crontab.
   - Manual run to start any missing processes: **`/usr/share/astguiclient/ADMIN_keepalive_ALL.pl --cu3way`**
4. **Boot startup (rc.local):** The addon-dialer appends to **`/etc/rc.d/rc.local`** and enables **rc-local.service**. Do **not** comment out on dialers:
   - Start Asterisk: **`/usr/share/astguiclient/start_asterisk_boot.pl`**
   - DAHDI: `modprobe dahdi`, `modprobe dahdi_dummy`, `dahdi_cfg`
   - Ensure **rc-local** runs at boot: `systemctl enable rc-local` (or `rc-local.service`).
5. **Reboot** the dialer, then verify:
   - **`screen -ls`** — should show `asterisk` and several `AST*` sessions (e.g. ASTupdate, ASTlisten, ASTsend, ASTVDauto, ASTVDremote, ASTfastlog).
   - **`asterisk -rx "core show channels"`** — Asterisk should respond.
   - If screens are missing after reboot, run **`/usr/share/astguiclient/ADMIN_keepalive_ALL.pl --cu3way`** and check again.
6. **Optional (see §8):** If you see “Unable to lookup SERVER_EXTERNAL_IP”, run:  
   `sudo sed -i 's/SERVER_EXTERNAL_IP/0.0.0.0/' /etc/asterisk/pjsip.conf`

**Files on each Asterisk/dialer (set by addon-dialer + run-on-dialer):**

| File | Purpose |
|------|--------|
| `/etc/astguiclient.conf` | VARserver_ip = this dialer; VARDB_server = DB+Web IP; VARactive_keepalives (e.g. 123468) |
| `/etc/rc.d/rc.local` | Start Asterisk (start_asterisk_boot.pl), DAHDI; leave enabled |
| `/etc/systemd/system/rc-local.service` | So rc.local runs at boot; must be enabled |
| Root crontab | ADMIN_keepalive_ALL.pl every minute (keepalive for AST_* processes) |
| `/etc/asterisk/*` | Asterisk config, pjsip, extensions, confbridge |
| `/etc/httpd/conf/httpd.conf` | RECORDINGS/MP3 Alias |

---

### 4.4 Adding each additional Asterisk/dialer server

For **each** extra dialer: (1) On **DB+Web** run **`./add-dialer-to-DB.sh`** and enter **this** dialer’s IP. (2) On the **new** server run **`./addon-dialer-alma9.sh`**, then **`./run-on-dialer-servers-cluster.sh`** with the DB+Web IP. (3) Ensure cron and rc.local are as in 4.3b, reboot, and verify **screen -ls** and Asterisk.

---

### 4.5 ConfBridge / "No one is in your session: 9600000"

Either rely on add-dialer-to-DB.sh (it inserts vicidial_confbridges for each dialer IP), or on DB+Web run `./confbridges.sh`, then fix IPs with:  
`/usr/share/astguiclient/ADMIN_update_server_ip.pl --old-server_ip=OLD_IP --server_ip=NEW_IP`.

---

### 4.6 Web UI and domain (DB+Web server only)

- Create vhost **`/etc/httpd/conf.d/yourdomain.conf`** for your FQDN (HTTP + HTTPS, SSL cert).
- **`/var/www/vhosts/dynportal/inc/defaults.inc.php`**: `$PORTAL_redirecturl` → `https://YOUR_FQDN/agc/vicidial.php`, `$PORTAL_redirectadmin` → `https://YOUR_FQDN/admin.php`.
- **`/etc/httpd/conf.d/viciportal.conf`**: ServerName (and ServerAlias) = your FQDN.
- **`/var/www/html/CyburPhone/cyburphone.php`**: `$referring_url` → `https://YOUR_FQDN`.

---

### 4.7 ViciDial Admin GUI (all servers and WebRTC)

In the web UI: **Admin → Servers:** set web_socket_url (and external_web_socket_url) for each server. **Admin → System Settings:** set sounds web server. **Admin → Templates:** edit SIP_generic/WebRTC template. **Admin → Carriers / DIDs:** configure as needed. (No file edits; all in DB via GUI.)

---

### 4.8 Summary – All files to change by role

**Common (all servers)**

| Item | What |
|------|------|
| Hostname | Unique per server; set with hostnamectl |
| `/etc/selinux/config` | SELINUX=disabled |
| crond | systemctl enable crond; systemctl start crond |

**DB+Web server only**

| File | When / what |
|------|-------------|
| `/etc/astguiclient.conf` | Set by main-installer. |
| `/etc/rc.d/rc.local` | Comment out Asterisk/DAHDI startup for DB+Web only. |
| `/etc/my.cnf.d/mariadb-server.cnf` | `bind-address = 0.0.0.0` for dialer access. |
| `/etc/httpd/conf.d/<fqdn>.conf` | Create: ServerName, HTTP/HTTPS, SSL certs. |
| `/etc/httpd/conf.d/viciportal.conf` | ServerName (ServerAlias) = your FQDN. |
| `/var/www/vhosts/dynportal/inc/defaults.inc.php` | PORTAL_redirecturl, PORTAL_redirectadmin. |
| `/var/www/html/CyburPhone/cyburphone.php` | $referring_url. |
| Root crontab | Installed by main-installer (ADMIN_keepalive_ALL.pl, etc.). |
| Firewall | Allow 80, 443, 81, 446; allow 3306 from dialer IPs. |

**Each Asterisk/dialer server only**

| File | When / what |
|------|-------------|
| `/etc/astguiclient.conf` | Set by addon-dialer + run-on-dialer (VARserver_ip, VARDB_server, VARactive_keepalives). |
| `/etc/rc.d/rc.local` | Leave Asterisk/DAHDI and start_asterisk_boot.pl enabled. |
| `/etc/systemd/system/rc-local.service` | Enabled so rc.local runs at boot. |
| Root crontab | Must include `* * * * * /usr/share/astguiclient/ADMIN_keepalive_ALL.pl` (keepalive). |
| `/etc/asterisk/pjsip.conf` | Optional: fix SERVER_EXTERNAL_IP (see §8). |

**Database (MySQL on DB+Web)**

| What | How |
|------|-----|
| `asterisk.vicidial_servers` | add-dialer-to-DB.sh (second_server_install.sql). |
| `asterisk.vicidial_confbridges` | add-dialer-to-DB.sh or confbridges.sh + ADMIN_update_server_ip.pl. |
| Web/WebRTC URLs | Admin → Servers, System Settings, Templates (GUI). |

---

## 5. ConfBridge / “No one is in your session: 9600000”

**Source:** Article comments and [confbridge.readme](https://github.com/carpenox/vicidial-install-scripts/blob/main/confbridge.readme)

- If agents see **“No one is in your session: 9600000”** (or similar), run the confbridges script on the **DB/web** server (or wherever your install scripts live):

```bash
cd /usr/src/vicidial-install-scripts
chmod +x confbridges.sh
./confbridges.sh
```

- ConfBridge rows in `asterisk.vicidial_confbridges` use a server IP (e.g. `10.10.10.17`). In the repo’s `confbridge.readme` you can insert confbridge records manually and then run:

```bash
/usr/share/astguiclient/ADMIN_update_server_ip.pl –-old-server_ip=10.10.10.17
```

(Replace with your actual server IP as needed.)

---

## 6. Firewalld (DB + Web server)

**Source:** [How to – Use Firewalld via command line](https://dialer.one/index.php/how-to-use-firewalld-via-command-line/)

- Works with the dynamic portal and ViciDial IP whitelist.

**Service:**

```bash
systemctl enable firewalld
systemctl start firewalld
systemctl stop firewalld
systemctl restart firewalld
systemctl status firewalld
```

**Rules (use `--permanent` then `firewall-cmd --reload`):**

- Add/remove port:  
  `firewall-cmd --permanent --add-port=81/tcp`  
  `firewall-cmd --permanent --remove-port=444/tcp`
- Add/remove service:  
  `firewall-cmd --permanent --add-service=http`  
  `firewall-cmd --permanent --remove-service=https`
- Trusted zone (for IP-only / no domain):  
  `firewall-cmd --add-service=http --zone=trusted --permanent`  
  `firewall-cmd --permanent --add-port=81/tcp`  
  `firewall-cmd --reload`
- Whitelist IP:  
  `firewall-cmd --permanent --add-source=192.168.1.100`  
  `firewall-cmd --permanent --add-source=192.168.1.0/24`
- List:  
  `firewall-cmd --list-all`

---

## 7. No Domain (IP-only) – DB + Web

**Source:** Article comments on the Alma 9 install page.

- ViciDial **works without a domain**; webphones may not work without SSL/domain.
- Add the IPs from which agents will connect (and where webphones would register) to the **trusted** zone:

```bash
firewall-cmd --permanent --add-port=81/tcp
firewall-cmd --add-service=http --zone=trusted --permanent
firewall-cmd --reload
```

- Use the dynamic portal at `http://YOUR_IP:81/valid8.php` to validate IPs.

---

## 8. Troubleshooting

### “Unable to open master device ‘/dev/dahdi/ctl’ for Dahdi”

**Source:** [How to – Fix: Unable to open master device '/dev/dahdi/ctl' for Dahdi](https://dialer.one/index.php/how-to-fix-unable-to-open-master-device-dev-dahdi-ctl-for-dahdi/)

- Common after OS updates on Alma/Rocky/CentOS. Recompile DAHDI:

```bash
cd /usr/src/dahdi-linux-complete-3.4.0+3.4.0/
make && make install && modprobe dahdi
```

(Path may vary; use the DAHDI source path from your installer.)

### “Unable to lookup ‘SERVER_EXTERNAL_IP’”

**Source:** [How to – Fix "Unable to lookup 'SERVER_EXTERNAL_IP'"](https://dialer.one/index.php/how-to-fix-unable-to-lookup-server_external_ip/)

- Newer SVN can show this in pjsip; it may not break anything. To clear it:

```bash
sudo sed -i 's/SERVER_EXTERNAL_IP/0.0.0.0/' /etc/asterisk/pjsip.conf
```

### Table ‘asterisk.vicidial_campaigns’ doesn’t exist

- Usually means the ViciDial DB wasn’t installed or was installed on another server. On a **DB-only** or **DB+web** server, ensure you ran the full installer (e.g. `main-installer.sh`) or restored a proper DB so that all ViciDial tables exist. If this is a dialer-only node, it should point to the central DB server where tables exist.

### DAHDI MeetMe / conference issues (alma-rocky9-ast16.sh)

- Author suggests running `dahdi_cfg -v` and sharing the output (e.g. on Discord) for diagnosis.  
- For SIP-only or cloud setups, Alma Linux 10 is an option; for full DAHDI hardware support, Alma Linux 9 is recommended.

### HTTP auth before ViciDial login

- If you get an extra auth prompt (e.g. htaccess) before the ViciDial login, check whether you used the “standard db add-in” and that Apache/httpd isn’t applying an extra .htaccess in front of the ViciDial app.

### “You don’t have permission to access /RECORDINGS/MP3/”

- Fix Apache permissions and ownership for the recordings directory. Ensure the Alias and &lt;Directory&gt; for `/var/spool/asterisk/monitorDONE/MP3/` in httpd config allow access (e.g. `Require all granted`) and that `apache` (or the httpd user) can read the files. See [dialer.one: Fix RECORDINGS/MP3 permission](https://dialer.one/index.php/how-to-fix-the-you-dont-have-permission-to-access-recordings-mp3-error-within-vicidial/).

### Server shows red in Admin Reports

- **Why it happens:** Admin reports show server status (red/green) based on **heartbeat updates** in the database. Those heartbeats are sent by **Asterisk-related processes** (e.g. AST_listen / AMI) that run on **dialer** servers. So:
  - **DB+Web-only server:** Has no Asterisk, so it does **not** send the same dialer heartbeat. It is **expected** for this server to show **red** in the admin server list even when it is healthy (MySQL, Apache, cron all running). The server can be healthy; the indicator is not designed for non-dialer nodes.
  - **Dialer server (Asterisk):** If a **dialer** shows red, then something is wrong: Asterisk not running, cron/keepalive not running, AST_* screen processes down, or DB unreachable from that dialer.
- **Checks if the red server is your DB+Web box:** Confirm MySQL and httpd are up (`systemctl status mariadb httpd`), cron is running (`systemctl status crond`), and the ViciDial web UI and login work. If all that is fine, the red is expected and can be ignored for that server.
- **Checks if the red server is a dialer:** On that dialer run `asterisk -rx "core show version"`, `screen -ls` (AST_* sessions), and ensure root crontab has `ADMIN_keepalive_ALL.pl` every minute; from the dialer test DB with `mysql -h DB_IP -u cron -p`.

---

## 9. SSL and WebRTC (when using a domain)

- If you didn’t install SSL during the initial run, stop firewalld before running certbot, then start it again.
- By default, 443 is in the public zone so you can change the default password; remove it from public after (see “Lock down firewall” above).
- Webphone/WebRTC with public domain:

```bash
cd /usr/src/vicidial-install-scripts
chmod +x vicidial-enable-webrtc.sh
./vicidial-enable-webrtc.sh
```

Only run this if you have a **public domain and public IP**.

**Certbot SSL renewal (when it fails):** The repo includes **`certbot.sh`** to renew Let’s Encrypt certs (stops firewalld, runs `certbot renew`, restores firewalld and viciportal-ssl.conf, reloads httpd). See also dialer.one: [How to – Renew your certbot SSL cert when it fails](https://dialer.one/index.php/how-to-renew-your-certbot-ssl-cert-when-it-fails/).

---

## 10. Alternative Installers (Alma 9)

- **PHP 8 (beta):**  
  `chmod +x main-installer-php8.sh` then `./main-installer-php8.sh`
- **Asterisk 16 (Alma/Rocky 9):**  
  `chmod +x alma-rocky9-ast16.sh` then `./alma-rocky9-ast16.sh`
- **Asterisk 18 (Alma/Rocky 9):**  
  `chmod +x alma-rocky9-ast18.sh` then `./alma-rocky9-ast18.sh`  
  Then update SSL path in `/etc/httpd/conf.d/viciportal-ssl.conf`.
- **Add-on dialer (Alma/Rocky 9):** for adding telephony-only servers to an existing cluster:  
  `chmod +x addon-dialer-alma9.sh` then `./addon-dialer-alma9.sh`
- **Add-on dialer (Alma 8):** telephony-only for Alma 8:  
  `chmod +x Vici-alma-dialer-install.sh` then `./Vici-alma-dialer-install.sh`
- **Contabo-only (Alma/Rocky/CentOS 9, Asterisk 18):**  
  `chmod +x alma-rocky-centos-9-ast18-contabo.sh` then `./alma-rocky-centos-9-ast18-contabo.sh`
- **Alma/Rocky 8 full install – Ast 16:**  
  `./alma-rocky-centos8-ast16` (script name in repo has no `.sh` suffix)

---

## 11. DB+Web only (no Asterisk) – Carpenox vs ViciBox

**There is no separate “DB+web only” installer in the carpenox (dialer.one) scripts.**

- **Carpenox (Alma/Rocky):** Only two roles are scripted:
  - **main-installer.sh** = full server (DB + web + Asterisk + dynamic portal + CyburPhone + SSL). No prompt to skip Asterisk.
  - **addon-dialer-alma9.sh** = **telephony-only** add-on (for additional dialer servers that connect to an existing DB).
- **Intended approach for a DB+web server with carpenox:** Use **main-installer.sh** on the server that will be DB+web, then **disable Asterisk** (and DAHDI) on that host after install:
  - Stop Asterisk: `screen -S asterisk -X quit`
  - In `/etc/rc.d/rc.local`, comment out: Asterisk log roll, DAHDI (modprobe/dahdi_cfg), `sleep 20`, and `start_asterisk_boot.pl`.
- **If you want a true “role-only” install** (DB-only or DB+web with no Asterisk installed at all), use **ViciBox** (OpenSUSE-based): run `vicibox-install` with **expert installation** and choose only “Database” and optionally “Web server”, and “N” for Telephony. See [ViciBox Cluster docs](https://docs.vicibox.com/en/latest/installation/phase2/cluster.html).

---

## 12. Alma Linux 10 (reference only – you are on Alma 9)

**Source:** [How to – Install ViciDial on Alma Linux 10](https://dialer.one/index.php/how-to-install-vicidial-on-alma-linux-10/)

- Alma 10 uses a 6.x kernel; legacy DAHDI needs patches. The AL10 installer is SIP-only friendly; for full DAHDI hardware, Alma 9 is recommended.
- Installer script: `cyburdial-installer-alma10.sh` (same repo).  
- Pre-requisites and post-install flow are similar; post-install article and firewalld apply.

---

## 13. Web UI “addons” (options.php) – Alma 9

**Why addons appear missing:** Many “addon” features (realtime server display, carrier stats, agent stats, etc.) are **not** enabled by default. They are controlled by **options.php**. If you only have **options-example.php** and never created **options.php**, those features will not show.

**Source:** [How to – Modify your options.php file to get the most out of your dialer](https://dialer.one/index.php/how-to-modify-your-options-php-file-to-get-the-most-out-of-your-dialer/)

### Enable addons (realtime and agent screen)

1. **Realtime screen (Admin / realtime):**  
   On Alma/Rocky/CentOS the ViciDial web root is **`/var/www/html/vicidial`**.  
   - Edit **`/var/www/html/vicidial/options-example.php`**.  
   - Set these to **1** (enable): **SERVdisplay**, **CUSTINFOdisplay**, **CARRIERstats**, **AGENTtimeSTATS**, **logoutLINK**, **parkSTATS**, **SLAstats**, **DIDdesc**, **AGENTlatency**.  
   - Optionally: **$RS_RR** from 40 to **4** (faster refresh), **report_default_format** to **'HTML'**, add **'SALE'** to **AGENTstatusTALLY**.  
   - Save and **rename** the file to **`options.php`** (so the app loads it instead of defaults).

2. **Agent login screen:**  
   Edit **`/var/www/html/agc/options-example.php`**. Enable **user login** so agents can log in with username/password only (phone login still stored on user account). Save and rename to **`options.php`**.

3. **Permissions:** Ensure the Apache user can read the new files (e.g. `chown apache:apache /var/www/html/vicidial/options.php /var/www/html/agc/options.php` if needed).

**No separate Alma 9–specific addon installer:** The carpenox Alma 9 installers (main-installer.sh, etc.) do not ship a separate “addons” package. Enabling the options above gives you the standard ViciDial UI addons (server display, carrier stats, agent stats, etc.). For third-party add-ons (e.g. VICIphone, QueueMetrics, white-label theme), see vendor docs and [ViciDial third-party add-ons](https://www.vicidial.com/?page_id=312).

### Campaign dial level – no installer script

**There is no script in the carpenox vicidial-install-scripts repo for setting or changing campaign dial level.** Dial level (e.g. **Auto Dial Level**, **Maximum Adapt Dial Level**, **Dial Level Threshold**) is configured in ViciDial itself:

- **Admin → Campaigns → [campaign] → modify:** Set **Auto Dial Level**, **Dial Method**, **ADAPT OVERRIDE**, **Auto Dial Level Threshold**, **Maximum Adapt Dial Level**, etc.
- **API:** The **non_agent_api.php** `update_campaign` action can set `auto_dial_level` (and related fields) for automation; the System Setting **Auto Dial Limit** caps the maximum value allowed.

The install scripts only mention “campaign” in the context of optional campaign export scripts and dialplan; they do not include a dedicated “set campaign dial level” script.

---

## 14. Useful tools (included in installer)

- **inxi** – hardware info: `inxi -Fxz`
- **htop** – process viewer
- **iftop** – network interface
- **sngrep** – SIP capture/analysis
- **Roundcube / Dovecot / Postfix** – webmail and mail server

**Repo utilities (vicidial-install-scripts):** **package-reinstall.sh** – reinstall packages used by the stack; **php-changer.sh** – switch PHP version (e.g. 8.1/8.2) if needed.

---

## 15. References

| Topic | URL |
|-------|-----|
| Alma 9 install | https://dialer.one/index.php/how-to-install-vicidial-on-alma-linux-9-with-my-new-auto-installer/ |
| Full functionality / post-install | https://dialer.one/index.php/how-to-use-the-full-functionality-of-the-vicidial-installer-by-carpenox/ |
| Fix Dahdi /dev/dahdi/ctl | https://dialer.one/index.php/how-to-fix-unable-to-open-master-device-dev-dahdi-ctl-for-dahdi/ |
| Fix SERVER_EXTERNAL_IP | https://dialer.one/index.php/how-to-fix-unable-to-lookup-server_external_ip/ |
| Firewalld | https://dialer.one/index.php/how-to-use-firewalld-via-command-line/ |
| Alma 10 install | https://dialer.one/index.php/how-to-install-vicidial-on-alma-linux-10/ |
| Install scripts (GitHub) | https://github.com/carpenox/vicidial-install-scripts |
| ViciBox cluster (DB/Web/Tel roles) | https://docs.vicibox.com/en/latest/installation/phase2/cluster.html |
| Discord (help) | https://discord.gg/ymGZJvF6hK (also linktr.ee/CyburDial) |
| Knowledge base / TOC | https://www.dialer.one, https://dialer.one/index.php/table-of-contents/ |
| Certbot SSL renewal | https://dialer.one/index.php/how-to-renew-your-certbot-ssl-cert-when-it-fails/ |
| Debug Webphones | https://dialer.one/index.php/how-to-debug-webphones-for-vicidial/ |
| RECORDINGS/MP3 permission | https://dialer.one/index.php/how-to-fix-the-you-dont-have-permission-to-access-recordings-mp3-error-within-vicidial/ |
| One dynamic portal for cluster | https://dialer.one/index.php/how-to-use-one-dynamic-portal-for-whitelisting-and-have-it-sync-across-an-entire-cluster/ |
| Add conferences for add-on servers | https://dialer.one/index.php/how-to-add-conferences-for-add-on-servers-to-a-cluster/ |
| Setup ViciDial cluster (dialer.one) | https://dialer.one/index.php/how-to-setup-a-vicidial-cluster/ |
| Modify options.php (Web UI addons) | https://dialer.one/index.php/how-to-modify-your-options-php-file-to-get-the-most-out-of-your-dialer/ |

---

*Summary: Alma Linux 9 is used as the DB and web server for ViciDial. Install with `main-installer.sh` after pre-requisites and reboot; use the post-install steps for password, firewall, domain, and optional standard-db. For a cluster (DB+Web + multiple Asterisk servers), follow **§4 Building a ViciDial Cluster**: **§4.1 Common** (all servers), **§4.2 DB+Web only**, **§4.3 Asterisk/dialer only** (including cron, ADMIN_keepalive_ALL.pl, and rc.local), then **§4.8** for the file-change summary by role. Use `confbridges.sh` if agents see “No one is in your session,” and add dialers with `add-dialer-to-DB.sh` plus `run-on-dialer-servers-cluster.sh` on each dialer. If realtime/admin "addons" (server display, carrier stats, etc.) are missing, create **options.php** from **options-example.php** in `/var/www/html/vicidial` and `/var/www/html/agc` (see **§13**).*
