# ViciDial Alma 9 – New machine setup order (DB+Web)

Use this order on a **fresh Alma Linux 9** machine. All as **root**.

---

## Before you start

- **Copy this folder to the new machine** (e.g. `/root/vicidial-alma9-setup/`).  
  The installer only clones the upstream repo; your scripts live here.
- **Edit constants** in the scripts if needed (hostname, FQDN, MySQL root password, SSL email).  
  See top of `prepare-and-install.sh` and `run-installer.sh`.

---

## 1. Prepare

```bash
/root/vicidial-alma9-setup/prepare-and-install.sh
```

- Sets locale, timezone, EPEL, git, kernel, disables SELinux, sets hostname, clones `vicidial-install-scripts`, enables crond.
- **Reboot when it finishes** (script can do `--reboot`, or run `reboot` yourself).  
  Required after disabling SELinux.

---

## 2. Run installer (after first reboot)

```bash
/root/vicidial-alma9-setup/run-installer.sh
```

- Pipes hostname/FQDN and MySQL password into `main-installer.sh`.
- At the end the **main-installer reboots the machine**.

---

## 3. Post-install fixes (after second reboot)

```bash
/root/vicidial-alma9-setup/post-install-fixes.sh
```

- Disables broken `viciportal-ssl.conf` if present so httpd can start, and starts httpd.  
  Run once after the installer’s reboot.

---

## 4. Optional but recommended

| Step | When | Command / action |
|------|------|-------------------|
| **SSL (no “Not secure”)** | After step 3 | `SSL_EMAIL=you@example.com /root/vicidial-alma9-setup/enable-ssl-letsencrypt.sh` (DNS must point to this server; port 80 open). |
| **DB+Web only – disable Asterisk** | After step 3 | Stop Asterisk, edit `/etc/rc.d/rc.local` (comment out Asterisk/DAHDI/start_asterisk_boot.pl), then **reboot** again. |
| **First login** | When httpd is up | https://YOUR_FQDN or https://YOUR_IP — user **6666**, password **1234**; change immediately. |

---

## Summary

| Order | Script / action |
|-------|------------------|
| 1 | `prepare-and-install.sh` |
| 2 | **Reboot** |
| 3 | `run-installer.sh` (machine reboots at end) |
| 4 | **Reboot** (done by installer) |
| 5 | `post-install-fixes.sh` |
| 6 | Optional: `enable-ssl-letsencrypt.sh`, DB+Web Asterisk disable, then follow NEXT-STEPS.txt |

Full details (firewall, MySQL for dialers, add dialers to cluster): **NEXT-STEPS.txt** and **Knowledge-base.md**.
