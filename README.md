# Moving a WordPress Multisite from Managed Cloud Hosting to an On-Premise Archive Server

A practical, step-by-step account of pulling a long-lived, multilingual WordPress multisite off a managed hosting provider and turning it into a read-only archive running on our own VM. Docker, WP-CLI, a lot of `lftp`, and a few surprises along the way.

This isn't a "click here, then here" tutorial written in a vacuum. It's what actually happened, including the parts that didn't work the first time. If you're about to do something similar, the gotchas are probably more useful than the happy path.

---

## Table of contents

1. [The backstory](#the-backstory)
2. [What we ended up with](#what-we-ended-up-with)
3. [Before you start](#before-you-start)
4. [Step 1: Prepare the VM](#step-1-prepare-the-vm)
5. [Step 2: Pull the files over FTP](#step-2-pull-the-files-over-ftp)
6. [Step 3: The filename encoding rabbit hole](#step-3-the-filename-encoding-rabbit-hole)
7. [Step 4: Get the database out](#step-4-get-the-database-out)
8. [Step 5: The Docker stack](#step-5-the-docker-stack)
9. [Step 6: Adjust wp-config.php and .htaccess](#step-6-adjust-wp-configphp-and-htaccess)
10. [Step 7: Import and rewrite the domains](#step-7-import-and-rewrite-the-domains)
11. [Step 8: Post-import cleanup](#step-8-post-import-cleanup)
12. [Step 9: Start it up and test](#step-9-start-it-up-and-test)
13. [Step 10: Make it repeatable](#step-10-make-it-repeatable)
14. [Step 11: Real TLS certificates (optional)](#step-11-real-tls-certificates-optional)
15. [Things we learned the hard way](#things-we-learned-the-hard-way)
16. [Troubleshooting](#troubleshooting)
17. [Security notes](#security-notes)

---

## The backstory

The site had lived on a managed WordPress host for many years. It started small and grew into a multisite network with one subsite per language: English as the main site, plus a handful of others. Each language had its own national domain (`site.example.de`, `site.example.fr` and so on), glued together with the old *WordPress MU Domain Mapping* plugin.

The site was being replaced by a new platform, and the hosting contract was going to be cancelled. The people who owned the content had one reasonable request: *"Before you switch it off, keep a copy we can still open. If something didn't make it across, we want to be able to find the original."*

So the goal wasn't a migration in the usual sense. We didn't need to keep the old site live on the internet. We needed a faithful, browsable archive, running somewhere we control, reachable only from the internal network.

A few constraints shaped everything that followed:

- **No production data on laptops.** Everything should go straight from the hosting provider to our server.
- **No SSH or remote database access** on the hosting side, as it turned out. More on that later.
- **Tens of thousands of files** and a multi-gigabyte database.
- **It had to be repeatable.** A first copy now, then a final refresh on the day content was frozen.

---

## What we ended up with

```
          internal network only
                   │
           ┌───────┴────────┐
           │  Ubuntu VM     │
           │  (on-premise)  │
           └───────┬────────┘
                   │ :80 / :443
              ┌────┴────┐
              │  Caddy  │  TLS termination
              └────┬────┘
              ┌────┴──────┐
              │ WordPress │  PHP 7.4 + Apache
              └────┬──────┘
              ┌────┴──────┐
              │ MySQL 8.0 │
              └───────────┘

   + a WP-CLI container for maintenance (run on demand)
```

All languages run as subdirectories under one hostname:

- `https://site.archive.example.com/` (main site)
- `https://site.archive.example.com/de/`, `/fr/`, `/ja/`, and so on

One DNS record, one certificate, no domain mapping. For an archive, that's all you need.

---

## Before you start

You'll need:

- A Linux VM. We used Ubuntu Server 24.04 with 2 vCPU, 4 GB RAM and a 100 GB disk. Rule of thumb for the disk: roughly three times the size of the site (downloaded files, SQL dumps, the imported database, and a compressed archive).
- FTP access to the old hosting (credentials from your password manager).
- A way to export the database. In our case, the hosting provider's phpMyAdmin.
- Admin access to the old WordPress (you'll want to look at *Tools → Site Health → Info*).
- Someone who can create a DNS record for the archive hostname.

Before touching anything, grab a few facts from **Site Health → Info** on the old site:

- PHP version
- MySQL version
- Database host, name and table prefix
- `ABSPATH` (tells you where WordPress lives on the server)

Match PHP and MySQL versions exactly in Docker. Upgrading PHP on an old site you just want to *preserve* is asking for trouble.

---

## Step 1: Prepare the VM

Nothing exotic here. Two admin users, a shared group so both can work on the same files, Docker, and a firewall.

```bash
# second admin user
sudo adduser ops
sudo usermod -aG sudo ops

# packages
sudo apt update && sudo apt upgrade -y
sudo apt install -y lftp mysql-client ufw jq

# Docker (official convenience script)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker deploy
sudo usermod -aG docker ops

# firewall - allow SSH before enabling!
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Log out and back in so the `docker` group membership takes effect.

A shared working directory with the setgid bit, so new files automatically belong to the shared group no matter who creates them:

```bash
sudo groupadd webadmins
sudo usermod -aG webadmins deploy
sudo usermod -aG webadmins ops

sudo mkdir -p /var/www/site/{files,db-dump}
sudo chown -R deploy:webadmins /var/www
sudo chmod -R 2775 /var/www
```

> **Heads-up:** Docker's published ports bypass `ufw`. Don't rely on the firewall to restrict access to ports that containers publish. In our case the VM only had an internal IP, so this wasn't a problem.

One piece of advice we ignored at first and then got teased about: **don't do all this as root.** We ran the initial download under `sudo su` and ended up with a tree full of root-owned files that needed fixing later. Work as a normal user with group permissions from the start.

---

## Step 2: Pull the files over FTP

### Find the real FTP host

The first attempt went nowhere. The control panel said to connect to `ftp.site.example.com`, and `lftp` just sat there on `Connecting...` forever.

The reason: at some point the site's DNS had been moved away from the hosting provider to other infrastructure. So `site.example.com` and every subdomain under it now pointed somewhere else, not to the host. The panel was showing a hostname that simply didn't lead to their server anymore.

A few quick checks made this obvious:

```bash
getent hosts ftp.site.example.com      # -> not the hosting provider. Wrong.
nc -vz -w 5 site.example.com 21        # timeout
nc -vz -w 5 test.rebex.net 21          # public test FTP works, so outbound FTP is fine
```

The fix was the provider's internal hostname for the hosting account (something like `ftp.<account>.hosting-provider.net`), listed in the panel in small grey text. If a panel-provided hostname doesn't work, look for a "fallback" or "server" hostname nearby.

### Mirror everything

Run this in `tmux`. It takes hours, and you don't want a dropped SSH session to kill it.

```bash
tmux new -s ftp

lftp -u 'FTP_USER' ftp.ACCOUNT.hosting-provider.net -e "
  set ftp:ssl-allow yes;
  set ftp:list-options -a;
  set net:max-retries 5;
  set net:reconnect-interval-base 10;
  mirror --parallel=2 --verbose \
    --exclude ^tmp/ --exclude ^logs/ --exclude ^\.provider-internal/ \
    / /var/www/site/files;
  quit" 2>&1 | tee /var/www/site/mirror.log
```

A few notes on the flags:

- `ftp:list-options -a` makes the server list hidden files. Without it you silently lose `.htaccess`, which a multisite in subdirectories absolutely needs.
- `--parallel=2` rather than 4. With 4 parallel connections we saw a lot of transient errors; the host seemed to throttle concurrent connections.
- We excluded `tmp/` (tens of thousands of session and cache files) and `logs/`. Neither belongs in an archive.

Expect this to be slow. It took the better part of a working day. FTP is not fast when most of your files are small image thumbnails.

> **Tip:** if you paste a multi-line heredoc (`<<'EOF' ... EOF`) into a terminal and it collapses into one line, the shell will sit there waiting for `EOF` forever. The one-line `lftp -e "..."` form above avoids that entirely.

---

## Step 3: The filename encoding rabbit hole

The first mirror finished with **thousands of errors**. All of them looked like this:

```
mirror: Access failed: 550 Übersicht-Beispiel-Grafik.jpg: No such file or directory
```

Every failing file had a non-ASCII character in its name: umlauts, accents, Japanese, Cyrillic. The FTP server listed these files happily, but refused to serve them. We tried the obvious fixes (`ftp:charset iso-8859-1`, UTF-8, decomposed Unicode forms) and confirmed the raw bytes with `curl --list-only | xxd`. The listing was clean UTF-8. The server just wouldn't let you download it. Something was broken on the host's side.

The workaround: the **web server** had no such problem. The same files downloaded fine over HTTPS from the live site. So we fetched the failures from the web instead.

### Verify, don't trust

The mirror summary is misleading: lftp counts *attempts*, not successes. The only reliable way to know you have everything is to compare full file lists.

Get the remote listing (this takes a while, since lftp walks every directory):

```bash
cd /var/www/site
lftp -u 'FTP_USER' ftp.ACCOUNT.hosting-provider.net -e "
  set ftp:ssl-allow yes; set ftp:list-options -a;
  find /www > /var/www/site/remote-raw.txt; quit"
```

Compare it with what's on disk:

```bash
LC_ALL=C grep -av '/$' remote-raw.txt | sed 's|^/||' | LC_ALL=C sort > remote.txt
(cd files && find www -type f) | LC_ALL=C sort > local.txt

LC_ALL=C comm -23 remote.txt local.txt > missing.txt   # on FTP, not on disk
LC_ALL=C comm -13 remote.txt local.txt > extra.txt     # on disk, not on FTP
wc -l remote.txt local.txt missing.txt extra.txt
```

Note the `-a` on `grep`. Without it, grep hits the first invalid byte, declares the file "binary", and **silently stops printing lines**. Our first comparison was off by hundreds of files because of this.

### Fetch what's missing over HTTPS

This script reads `missing.txt`, builds the public URL for each path, and downloads it. It also handles one more surprise we found along the way (see below).

`fetch_missing.py`:

```python
#!/usr/bin/env python3
# Download files listed in missing.txt (FTP-vs-disk diff) over HTTPS.
# Names that aren't valid UTF-8 are treated as Windows-1251 and saved under their UTF-8 name.
# Usage: fetch_missing.py MISSING_TXT LOCAL_ROOT BASE_URL PREFIX
import os, subprocess, sys, urllib.parse

missing, local_root, base_url, prefix = sys.argv[1], sys.argv[2], sys.argv[3].rstrip("/"), sys.argv[4]

ok = fail = 0
with open(missing, "rb") as f:
    for raw in f:
        raw = raw.rstrip(b"\n")
        if not raw:
            continue
        try:
            path = raw.decode("utf-8")
            note = ""
        except UnicodeDecodeError:
            path = raw.decode("cp1251")
            note = " (converted from Windows-1251)"
        if not path.startswith(prefix):
            continue
        url = base_url + "/" + urllib.parse.quote(path[len(prefix):].encode("utf-8"))
        dest = os.path.join(local_root, path)
        os.makedirs(os.path.dirname(dest), exist_ok=True)
        if os.path.exists(dest) and os.path.getsize(dest) > 0:
            print("SKIP", path + note)
            continue
        r = subprocess.run(["curl", "-sf", "--max-time", "60", "-o", dest, url])
        if r.returncode == 0 and os.path.getsize(dest) > 0:
            ok += 1
            print("OK  ", path + note)
        else:
            fail += 1
            print("FAIL", path + note)
            if os.path.exists(dest) and os.path.getsize(dest) == 0:
                os.remove(dest)

print(f"Done. Downloaded: {ok}, failed: {fail}")
```

```bash
python3 fetch_missing.py /var/www/site/missing.txt /var/www/site/files https://site.example.com www/
```

Then re-run the comparison. Repeat until it comes out clean.

### The files from an old Windows machine

After the fix, a few files were still stubborn. All in the same subsite, all uploaded many years ago. Their names weren't UTF-8 at all. They were **Windows-1251** bytes, the old Cyrillic Windows encoding, most likely uploaded from a Windows machine back when nobody cared. Our first version of the script decoded them as UTF-8 with replacement characters, which turned every letter into `�` and produced nonsense URLs.

The web server, however, served them under their proper UTF-8 names (think `пример-изображения.jpg`, "example image"). So the script now tries UTF-8 first, falls back to Windows-1251, and saves the file under the UTF-8 name, exactly as the website serves it.

That leaves a permanent, expected difference between the FTP listing and the disk: the same handful of files in `missing.txt` and in `extra.txt`, identical images with differently encoded names. When those two lists match up, you're done.

### Never re-run a plain mirror

We learned this one live. After all the web downloads, someone re-ran the original `mirror` command from shell history. lftp saw that the web-downloaded files had different timestamps than on FTP, decided they were "modified", **deleted them**, and tried to re-download them over FTP, which of course failed again.

From that point on, every follow-up sync used `--only-missing`:

```bash
lftp ... -e "... mirror --only-missing --no-perms --parallel=2 --verbose ... / /var/www/site/files; quit"
```

That flag downloads new files and never touches existing ones. It also protects the `wp-config.php` and `.htaccess` you're about to modify.

Why `--no-perms`? By default, mirror also tries to copy the remote file permissions onto local directories. Once the web root belongs to `www-data` (Step 6), a regular user can still *write* there through the shared group, but can't `chmod` it. The result is a wall of harmless but alarming `chmod: Access failed: Operation not permitted` lines. Running the sync as root makes them disappear, but then new files belong to root again. `--no-perms` is the cleaner fix: let mirror fetch the files, and set ownership yourself afterwards.

One more thing when reading the logs: lftp's `550` error lines contain only the **file name**, not its path. So filtering errors by directory (`grep -v "some/dir"`) won't work on those lines. Count them instead: if the number of failures matches the files you already know about, nothing new is missing.

---

## Step 4: Get the database out

The plan was simple: run `mysqldump` from the VM against the host's database server. It didn't work. Port 3306 timed out, and the host's documentation confirmed that shared hosting databases are only reachable from their own servers.

Plan B was SSH. The panel let us upload a public key, so we generated one on the VM:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/hosting -C "archive-vm"
```

Port 22 timed out too, on every hostname we tried. Outbound SSH from our network was fine (GitHub on port 22 answered immediately), so it was something on the host's side.

We also considered uploading a small PHP script over FTP that would dump the database into a file outside the web root, then fetch that file over FTP. It works, and we had it written, but it means putting a database-dumping endpoint on a production server, however briefly. We kept it as a last resort.

In the end, the pragmatic answer was **phpMyAdmin** from the hosting panel. It runs next to the database, so no ports need opening. Export settings that worked for a multi-gigabyte database with hundreds of tables:

| Setting | Value |
|---|---|
| Export method | Custom |
| Format | SQL |
| Tables | all, structure + data |
| Output | save to file, UTF-8 |
| Enclose export in a transaction | on |
| Disable foreign key checks | **on** |
| Add DROP TABLE / VIEW / … | **on** (so re-imports overwrite cleanly) |
| Add CREATE DATABASE / USE | off (we name the database ourselves) |
| Everything else | default |

The download lands on your laptop, which bent our "no data on laptops" rule. We copied it straight to the VM and deleted it locally:

```bash
scp ~/Downloads/site_db.sql deploy@ARCHIVE_VM:/var/www/site/db-dump/site-db.sql
```

Sanity checks:

```bash
tail -n 3 /var/www/site/db-dump/site-db.sql
grep -c "^CREATE TABLE" /var/www/site/db-dump/site-db.sql
```

Compare the table count with phpMyAdmin. One trap: phpMyAdmin paginates at 250 tables. Our first look showed exactly "250 tables" and we briefly panicked before noticing the page selector. Add up all pages, then compare.

### When the export silently breaks

The first export worked fine. A few days later, on cutover day, the same export from the same phpMyAdmin produced a file that looked perfectly normal: right size, right name, download completed. The import then failed with:

```
ERROR 1064 (42000) at line 226600: You have an error in your SQL syntax ...
near '<div class="alert alert-danger" role="alert"><h1>Error</h1>...'
```

Somewhere in the middle of the SQL file was an **HTML error page**. phpMyAdmin had lost its database connection halfway through (`#2006 - MySQL server has gone away`), wrote its error message into the download stream, and stopped. A second attempt broke at a different line, which told us it wasn't bad data in one table. The export simply no longer fit into the host's time and connection limits.

The fix was to make the export smaller. The biggest tables in the database belonged to a security plugin (Wordfence's `wp_wf*` tables: traffic logs, login attempts, blocked IPs). They contain no content, and they're full of visitor IP addresses you probably don't want to keep in a long-term archive anyway. Unticking them in the table list made the export finish cleanly. If that's not enough, export the tables in two halves and concatenate the files; with *Add DROP TABLE* enabled, each part is self-contained:

```bash
cat part1.sql part2.sql > site-db.sql
```

**Always check a dump before importing it:**

```bash
grep -ac "alert-danger" site-db.sql               # must be 0
grep -ac "MySQL server has gone away" site-db.sql # must be 0
tail -n 3 site-db.sql                             # must be plain SQL, not HTML
```

A phpMyAdmin dump normally ends with a few `/*!40101 SET ... */;` lines. If the tail shows anything else, don't import it.

---

## Step 5: The Docker stack

Three files live in `/var/www`.

### `php-memory.ini`

```ini
memory_limit = 1024M
```

The default 128 MB wasn't enough for WP-CLI to even boot a multisite network with its plugins.

### `docker-compose.yml`

```yaml
services:
  db:
    image: mysql:8.0
    command: --max_allowed_packet=256M --sql-mode=NO_ENGINE_SUBSTITUTION
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: CHANGE_ME
      MYSQL_ROOT_PASSWORD: CHANGE_ME_ROOT
    volumes:
      - db:/var/lib/mysql
    restart: unless-stopped

  wp:
    image: wordpress:php7.4-apache
    volumes:
      - ./site/files/www:/var/www/html
      - ./php-memory.ini:/usr/local/etc/php/conf.d/memory.ini:ro
    depends_on: [db]
    restart: unless-stopped

  caddy:
    image: caddy:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy:/data
    depends_on: [wp]
    restart: unless-stopped

  cli:
    image: wordpress:cli-php7.4
    volumes:
      - ./site/files/www:/var/www/html
      - ./php-memory.ini:/usr/local/etc/php/conf.d/memory.ini:ro
    depends_on: [db]
    user: "33:33"
    entrypoint: wp
    profiles: ["cli"]

volumes:
  db:
  caddy:
```

Generate real passwords with `openssl rand -hex 16`. MySQL only reads them the **first** time a volume is initialised, so decide before the first `docker compose up`.

The WordPress image only provides PHP and Apache. WordPress itself (core, plugins, theme) comes from the mounted backup, so you run exactly the versions the old host ran.

The `cli` service sits behind a profile, so it only runs when you explicitly call it.

### `Caddyfile`

```
site.archive.example.com {
    tls internal
    reverse_proxy wp:80
}
```

`tls internal` makes Caddy issue its own certificate. Browsers will warn once, and people click through. For a real certificate, see Step 11.

Create this file **before** starting Caddy. If it's missing, Docker helpfully creates an empty *directory* named `Caddyfile` and Caddy fails to start.

### A note on choosing the archive domain

We first picked a hostname under a `.dev` domain. Then we remembered that the entire `.dev` TLD is on the browsers' HSTS preload list: HTTPS is mandatory and certificate warnings **cannot be clicked through**. With a VM on an internal IP, getting a valid certificate would have meant DNS-based ACME validation through the registrar's API.

We switched to a `.com` domain we already owned, where an internal certificate is good enough for an internal archive. It meant one extra search-replace pass (Step 7), which took a few minutes. Pick your domain before you start rewriting databases, and it'll cost you nothing.

---

## Step 6: Adjust wp-config.php and .htaccess

Back up the originals first. The archive should always contain the untouched files from the old host.

```bash
cp /var/www/site/files/www/wp-config.php /var/www/site/wp-config.php.original
```

### wp-config.php

Right under `<?php`, tell WordPress it's behind an HTTPS proxy:

```php
if (($_SERVER['HTTP_X_FORWARDED_PROTO'] ?? '') === 'https') { $_SERVER['HTTPS'] = 'on'; }
```

We forgot this line once. The page loaded, but with no styling at all: every stylesheet URL was `http://`, and browsers block HTTP resources on HTTPS pages. One line fixed it.

One-liner if you prefer:

```bash
sed -i "1a if ((\$_SERVER['HTTP_X_FORWARDED_PROTO'] ?? '') === 'https') { \$_SERVER['HTTPS'] = 'on'; }" wp-config.php
```

Then point it at the new database and domain:

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpress');
define('DB_PASSWORD', 'CHANGE_ME');
define('DB_HOST', 'db');
define('DOMAIN_CURRENT_SITE', 'site.archive.example.com');
define('SUNRISE', false);   // disable domain mapping (see Step 8)
```

### .htaccess: admin IP allowlist

After everything was running, `/wp-admin/` returned **403 Forbidden** from Apache. The old site restricted `wp-admin/` and `wp-login.php` to a public office IP range:

```apache
order deny,allow
deny from all
allow from 203.0.113.10
allow from 203.0.113.11
...
```

That can't work behind Caddy. Apache sees the Caddy container's IP (something in `172.16.0.0/12`), not the visitor's. And even if it could see the real IP, internal users come from a private range, not the public one.

Since the VM is only reachable internally, we allowed the Docker network and left the original rules in place:

```bash
cp /var/www/site/files/www/wp-admin/.htaccess /var/www/site/wp-admin.htaccess.original
cp /var/www/site/files/www/.htaccess          /var/www/site/root.htaccess.original

sed -i '/allow from 203.0.113.10/a allow from 172.16.0.0/12' \
  /var/www/site/files/www/wp-admin/.htaccess \
  /var/www/site/files/www/.htaccess
```

### Permissions

```bash
sudo chown -R deploy:webadmins /var/www
sudo chown -R 33:webadmins /var/www/site/files/www
sudo chmod -R g+w /var/www/site/files/www
```

UID 33 is `www-data` inside the WordPress container. That's specific to the Debian-based images like `wordpress:php7.4-apache`, and it happens to match `www-data` on an Ubuntu or Debian host. Alpine-based variants (`*-fpm-alpine`) use UID **82** instead, so adjust the `chown` and the `user:` line of the CLI container if you go that route.

---

## Step 7: Import and rewrite the domains

### Import

```bash
cd /var/www
docker compose up -d db
until docker compose exec db mysqladmin ping -h localhost --silent; do sleep 2; done

docker compose exec -T db sh -c 'exec mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"' \
  < site/db-dump/site-db.sql && echo "IMPORT OK"
```

The `sh -c '...'` wrapper lets the container use its own credentials, so the password never shows up on the host's command line or in `ps`.

Then count the tables and compare with the dump.

### Figure out the domain map

WordPress stores URLs everywhere: post content, image `guid`s, options, SEO plugin tables, serialized arrays. With domain mapping, those URLs use the *national* domains. So first, find out which domain belongs to which subsite. It's right there in the dump:

```bash
grep -A 15 "INSERT INTO \`wp_blogs\`"          site/db-dump/site-db.sql
grep -A 20 "INSERT INTO \`wp_domain_mapping\`" site/db-dump/site-db.sql
```

`wp_blogs` gives you `blog_id → path` and `wp_domain_mapping` gives you `blog_id → domain`. Join them, and be careful: the path isn't always what you'd guess from the domain. In our case the `.jp` domain lived at `/ja/` and `.com.br` at `/br/`.

### Write the replacement list

`replace.txt`, one `OLD NEW` pair per line:

```
site.example.com.br site.archive.example.com/br
site.de.example.com site.archive.example.com/de
site.example.fr site.archive.example.com/fr
site.example.jp site.archive.example.com/ja
site.example.com site.archive.example.com
```

Add one line per mapped domain.

**Order matters.** `site.example.com` is a substring of `site.example.com.br`. Replace the main domain first and the Brazilian site becomes `site.archive.example.com.br`, which doesn't exist. Longest and most specific first, main domain last.

We replace bare hostnames, not `https://…` URLs. That way the same pass catches `http://`, `https://`, protocol-relative `//` links and JSON-escaped `https:\/\/` variants.

### Run the replacements

```bash
cd /var/www
while read -u 3 OLD NEW; do
  echo "== $OLD -> $NEW"
  docker compose --profile cli run --rm -T cli search-replace "$OLD" "$NEW" \
    --network --all-tables --precise --skip-tables='wp_wf*' --report-changed-only \
    --url=site.example.com --skip-plugins --skip-themes < /dev/null
done 3< replace.txt
```

Every flag here is the result of something going wrong first:

- **`--url=site.example.com`**: `wp-config.php` already says the new domain, but the database still contains the old one. Without this, WP-CLI can't find the network (`Error: Site '…' not found`). It keeps working through the last line, because WP-CLI bootstraps before the replacement changes anything.
- **`--skip-plugins --skip-themes`**: faster, uses less memory, and nothing in a search-replace needs plugins loaded.
- **`--precise`**: forces PHP-level processing of every row. Slower, but safe for serialized data.
- **`--skip-tables='wp_wf*'`**: security plugin log tables. Huge and irrelevant for content.
- **`read -u 3` and `< /dev/null`**: our first loop ran only the first line and quietly exited. `docker compose run` reads stdin even with `-T`, and it swallowed the rest of the replacement list. Reading the list from file descriptor 3 and giving Docker `/dev/null` fixes it.

You'll see warnings like `Skipping an uninitialized class "FS_Plugin"`. Those are serialized objects from the Freemius licensing SDK that some premium plugins use. Because plugins aren't loaded, WP-CLI can't unserialize them and leaves them alone. It's license metadata, not content.

The last line (the main domain) does the most work, typically hundreds of thousands of replacements, because every subsite references it.

### Verify

```bash
docker compose --profile cli run --rm -T cli site list --fields=blog_id,url --skip-plugins --skip-themes < /dev/null

docker compose --profile cli run --rm -T cli search-replace 'site.example.com' 'X' \
  --network --all-tables --skip-tables='wp_wf*' --dry-run --skip-plugins --skip-themes < /dev/null | tail -1
```

The first command should list every subsite on the new hostname, and it succeeding at all (without `--url`) proves the database now matches `wp-config.php`. The second should say `0 replacements to be made`.

---

## Step 8: Post-import cleanup

### Main site on HTTPS

`site list` showed every subsite on `https://` except the main site, which was `http://`. On the old host that didn't matter, because an edge proxy handled HTTPS. Here it would risk a redirect loop between WordPress and Caddy.

```bash
docker compose --profile cli run --rm -T cli option update home    'https://site.archive.example.com' --url=http://site.archive.example.com/ --skip-plugins --skip-themes < /dev/null
docker compose --profile cli run --rm -T cli option update siteurl 'https://site.archive.example.com' --url=http://site.archive.example.com/ --skip-plugins --skip-themes < /dev/null
```

### Turn off domain mapping

With `SUNRISE` set to `false`, `sunrise.php` isn't loaded. But the domain mapping plugin itself can still redirect visitors to the "primary" mapped domain, so we deactivated it network-wide. The plugin files, `sunrise.php` and the `wp_domain_mapping` table all stay in the archive untouched; we only change the copy's configuration.

```bash
docker compose --profile cli run --rm -T cli plugin deactivate wordpress-mu-domain-mapping --network --skip-themes < /dev/null
```

### Turn off anything that talks to live infrastructure

This one is easy to miss. The site used a page-cache plugin that stores CDN API credentials. Edit a post in the archive, and it may happily send a cache purge to your **live** CDN zone. Deactivate plugins like that:

```bash
docker compose --profile cli run --rm -T cli plugin deactivate wp-cloudflare-page-cache --network --skip-themes < /dev/null
```

Look through your plugin list for anything else with outbound side effects: CDN integrations, newsletter syncs, SMTP senders, backup plugins pushing to cloud storage. Security plugins are worth a look too. There's nothing left to protect in a read-only internal copy, and a firewall plugin that decides to block an IP may well block the reverse proxy, which takes the whole site down for everyone.

```bash
docker compose --profile cli run --rm -T cli plugin list --fields=name,status --skip-themes < /dev/null | grep -iE "cache|cloudflare|cdn|smtp|mail|backup|security|firewall"
```

Deactivating a plugin that isn't installed just prints `Error: No plugins network deactivated.` and is harmless, which is why the refresh script in Step 10 adds `|| true` to those lines.

### Keep search engines out

```bash
for URL in $(docker compose --profile cli run --rm -T cli site list --field=url --skip-plugins --skip-themes < /dev/null); do
  docker compose --profile cli run --rm -T cli option update blog_public 0 --url="$URL" --skip-plugins --skip-themes < /dev/null
done
```

The archive is internal-only anyway, but belt and braces.

---

## Step 9: Start it up and test

```bash
cd /var/www
docker compose up -d
docker compose ps
```

From the VM:

```bash
curl -skI https://site.archive.example.com/    | head -1
curl -skI https://site.archive.example.com/de/ | head -1
```

You want `HTTP/2 200`.

Check that pages don't reference the old domains (hard-coded URLs in a theme would show up here):

```bash
curl -sk https://site.archive.example.com/de/ | grep -oE 'site\.([a-z]+\.)?example\.[a-z.]+' | sort | uniq -c
```

And that assets load over HTTPS:

```bash
curl -sk https://site.archive.example.com/ | grep -oE '<link[^>]+stylesheet[^>]*>' | head -3
```

Then open it in a browser: click through a few languages, check images, log into `/wp-admin/` with an existing account. User accounts and password hashes come with the database, so the old logins work as they did on the day of the export.

---

## Step 10: Make it repeatable

The first copy was a dry run. The real one happens on the day content is frozen. Importing a fresh dump wipes out every database fix from Steps 7 and 8, so we put all of it into one script.

`refresh-db.sh`:

```bash
#!/bin/bash
# Re-import the database from a fresh dump and re-apply every archive fix.
# Run from /var/www as a regular user: ./refresh-db.sh
set -euo pipefail
cd /var/www

DUMP=site/db-dump/site-db.sql
REPLACE=replace.txt
OLD_MAIN=site.example.com
NEW_MAIN=site.archive.example.com

run_cli() { docker compose --profile cli run --rm -T cli "$@" < /dev/null; }

echo "### check dump"
[ -s "$DUMP" ] || { echo "ERROR: $DUMP is missing or empty"; exit 1; }
if grep -aq -e "alert-danger" -e "MySQL server has gone away" "$DUMP"; then
  echo "ERROR: $DUMP contains a phpMyAdmin error page - export it again"
  exit 1
fi

echo "### import"
# Credentials are read inside the container, so the password never
# appears in the host's process list.
docker compose exec -T db sh -c 'exec mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"' < "$DUMP"

echo "### rewrite domains"
while read -u 3 OLD NEW; do
  [ -z "$OLD" ] && continue
  echo "== $OLD -> $NEW"
  run_cli search-replace "$OLD" "$NEW" --network --all-tables --precise \
    --skip-tables='wp_wf*' --report-changed-only --url="$OLD_MAIN" --skip-plugins --skip-themes
done 3< "$REPLACE"

echo "### main site on HTTPS"
run_cli option update home    "https://$NEW_MAIN" --url="http://$NEW_MAIN/" --skip-plugins --skip-themes
run_cli option update siteurl "https://$NEW_MAIN" --url="http://$NEW_MAIN/" --skip-plugins --skip-themes

echo "### disable domain mapping, CDN cache and security plugins"
run_cli plugin deactivate wordpress-mu-domain-mapping --network --skip-themes || true
run_cli plugin deactivate wp-cloudflare-page-cache    --network --skip-themes || true
run_cli plugin deactivate wordfence                   --network --skip-themes || true

echo "### discourage search engines"
for URL in $(run_cli site list --field=url --skip-plugins --skip-themes); do
  run_cli option update blog_public 0 --url="$URL" --skip-plugins --skip-themes
done

echo "### check"
run_cli site list --fields=blog_id,url --skip-plugins --skip-themes
echo "DONE"
```

```bash
chmod +x refresh-db.sh
```

The script uses the database credentials the container already has, so nothing sensitive lives in it. It's idempotent: if it fails halfway, fix the cause and run it again.

An earlier version passed the password as `-p"$PASS"` on the host. It worked, but anyone running `ps` or `pgrep -af mysql` on the VM could read the database password in plain text. Letting the container expand `$MYSQL_PASSWORD` itself keeps it off the host's process list.

The dump check is there because of the broken phpMyAdmin export described in Step 4. Without it, the import got about two-thirds of the way through, hit the HTML, and stopped (thanks to `set -e`, at least, before rewriting any domains). The database was left half old, half new. Nothing was lost, since rerunning the script with a good dump fixed it, but it's nicer to refuse a bad dump before touching anything.

### Before you run it

Make sure there's a fallback. We waited for the hypervisor's scheduled nightly VM backup and checked it had completed. A VM snapshot taken just before the run works too, but delete it as soon as everything checks out. Snapshots grow with every write, slow the VM down, and aren't a real backup: lose the storage and you lose the VM and the snapshot together.

### Running it

The run takes a couple of hours, so don't tie it to your SSH session:

```bash
cd /var/www
nohup ./refresh-db.sh > refresh-$(date +%F).log 2>&1 &
tail -f refresh-$(date +%F).log      # Ctrl+C stops watching, not the script
pgrep -af refresh-db                 # still running?
```

The `mysql` import prints nothing while it works. To see how far it has got, ask the database which table it created last:

```bash
docker compose exec -T db sh -c 'mysql -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE" -e "
  SELECT table_name, create_time FROM information_schema.tables
  WHERE table_schema = DATABASE() ORDER BY create_time DESC LIMIT 3;"' 2>/dev/null
```

The top row is the table being loaded right now. phpMyAdmin exports in natural order (`wp_2_…` through `wp_9_…`, then `wp_10_…`, `wp_11_…`), so the subsite number tells you roughly where you are.

When it's done, check the log for real problems:

```bash
tail -n 30 refresh-$(date +%F).log         # last line must be DONE
grep -a -B 3 "Error" refresh-$(date +%F).log
```

An `Error: No plugins network deactivated.` just means that plugin isn't installed.

### Cutover day checklist

1. Content freeze confirmed with the content owners. Note the time.
2. Export the database from phpMyAdmin (settings from Step 4). If the export breaks, leave out the security plugin's log tables or split it in two.
3. `scp` it to the VM, overwriting the old dump. Delete it from your laptop. Check it for embedded error pages (Step 4).
4. Sync new files with `mirror --only-missing --no-perms`, then reset ownership:
   ```bash
   sudo chown -R 33:webadmins /var/www/site/files/www
   sudo chmod -R g+w /var/www/site/files/www
   ```
   If the number of `550` errors is higher than the files you already know about, run the comparison and `fetch_missing.py` from Step 3. This only works while the old site is still online, so don't cancel the hosting before this step.
5. Confirm a fresh VM backup exists, then run the refresh with `nohup` (see above).
6. Test (Step 9), including the newest content from before the freeze. Once a real certificate is in place (Step 11), drop the `-k` from the `curl` checks.
7. Archive everything to long-term storage:
   ```bash
   tar -czf /tmp/site-archive-$(date +%F).tar.gz \
     site docker-compose.yml Caddyfile php-memory.ini \
     replace.txt refresh-db.sh fetch_missing.py
   ```
   This includes the original dump and the `*.original` config files, so the archive can be rebuilt from scratch at any time.
8. Tell the content owners where the archive lives.

---

## Step 11: Real TLS certificates (optional)

`tls internal` is fine for most internal tools. If you'd rather avoid the browser warning, and your organisation runs an internal certificate service (typically something that performs ACME DNS-01 validation against Let's Encrypt on your behalf), you can have Caddy serve that certificate instead.

The general pattern:

1. A small fetch script calls the issuer API and writes `fullchain`, `key` and a metadata file into `/var/www/certs/<zone>/`, only replacing files that actually changed.
2. A **systemd timer** runs it every couple of days. No cron needed.
3. Caddy reads the files:
   ```
   site.archive.example.com {
       tls /certs/example.com/NAME.bundle.pem /certs/example.com/NAME.key
       reverse_proxy wp:80
   }
   ```
   with `./certs:/certs:ro` mounted into the Caddy container.
4. Caddy only reads certificate files at config load, so it needs a reload after renewal.

For step 4, we deliberately didn't give the fetch script's service user access to Docker. Docker group membership is effectively root. Instead, a root-owned systemd **path unit** watches the metadata file (written last, so key and chain are guaranteed complete by then) and reloads Caddy:

`/etc/systemd/system/caddy-cert-reload.path`:

```ini
[Unit]
Description=Reload Caddy on certificate change

[Path]
PathChanged=/var/www/certs/example.com/NAME.metadata.json

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/caddy-cert-reload.service`:

```ini
[Unit]
Description=Reload Caddy configuration

[Service]
Type=oneshot
ExecStart=/usr/bin/docker compose -f /var/www/docker-compose.yml exec -T caddy caddy reload --config /etc/caddy/Caddyfile
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now caddy-cert-reload.path
```

Keep the issuer API token in a separate env file readable only by the service user, not in the script. And if the issuer offers HTTPS, use it, so the token never crosses the network in plaintext.

---

## Things we learned the hard way

**Don't trust the control panel's hostnames if your DNS has moved.** Half a day of "why won't FTP connect" came down to DNS pointing somewhere the panel didn't know about.

**Verify by comparing file lists, not by reading summaries.** lftp's "errors detected" count, the script's "downloaded" count, phpMyAdmin's paginated table count: every one of them was misleading at some point. A sorted `comm` of two file lists is boring and always right.

**Old filenames are a time capsule.** If a site has been around long enough, somebody uploaded something with a filename in an encoding you haven't thought about in a decade.

**`--only-missing` is not optional** once you've touched anything locally.

**Pick the final domain before rewriting the database.** Changing it later is cheap but not free, and `.dev` has HSTS strings attached.

**Look for plugins that phone home.** An archive that can purge your live CDN cache isn't a read-only archive.

**Script the fixes the moment they work.** The first run is exploration. The second run, on a deadline, should be one command.

**An export that worked last week can fail today.** Same tool, same settings, different result, because the data grew or the server was busier. Check every dump before you import it, not just the first one.

**`set -e` is cheap insurance.** When the broken dump hit the import, the script stopped right there instead of happily rewriting domains in a half-imported database.

**Have a way back before the big run.** A completed VM backup turned "what if this breaks everything" into "then we restore and try again", which makes it much easier to press Enter.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| lftp stuck on `Connecting...` | panel hostname points elsewhere after a DNS move | use the provider's internal FTP hostname |
| FTP `550` on files with accents or non-Latin names | host-side FTP bug | fetch over HTTPS with `fetch_missing.py` |
| `grep: binary file matches`, comparison off | invalid bytes in filenames | `grep -a` |
| Heredoc does nothing, shell shows `>` | pasted as one line | use `lftp -e "..."` |
| Web-fetched files disappear | plain `mirror` re-run | always `--only-missing` |
| `chmod: Access failed: Operation not permitted` during mirror | mirror copies permissions, web root belongs to `www-data` | add `--no-perms`, reset ownership afterwards |
| Filtering `550` errors by path doesn't work | error lines contain only the file name | count the errors instead |
| `ERROR 1064 … near '<div class="alert alert-danger"'` on import | phpMyAdmin export broke midway (`MySQL server has gone away`) | export again without log tables, or in two parts; check the dump before import |
| `Access denied for user` on import | wrong password | let the container use its own credentials (`sh -c '... $MYSQL_PASSWORD ...'`) |
| `Allowed memory size … exhausted` | 128 MB PHP default | mount `php-memory.ini` into `cli` and `wp` |
| `Site '…' not found` in WP-CLI | DB still has old domain | add `--url=<old domain>` |
| Replacement loop runs only once | Docker reads stdin | `read -u 3` + `< /dev/null` |
| `Skipping an uninitialized class "FS_…"` | Freemius license objects | harmless |
| `Error: No plugins network deactivated.` | plugin not installed | harmless |
| `/wp-admin/` → 403 | IP allowlist in `.htaccess` | allow `172.16.0.0/12` |
| Page loads without styles | assets on `http://` | `HTTP_X_FORWARDED_PROTO` line in `wp-config.php` |
| Redirect loop on main site | main site `home`/`siteurl` on `http://` | update both to `https://` |
| Caddy won't start | `Caddyfile` became a directory | create the file before `docker compose up` |
| Hostname doesn't resolve for some users | DNS rebinding protection drops public records pointing to private IPs | internal DNS record or resolver exception |

---

## Security notes

- The archive runs **PHP 7.4, which is end-of-life**. That's fine for a frozen, internal-only archive. It is not fine on the public internet. Keep it on an internal IP.
- The backup includes the old hosting account's home directory, which may contain `.bash_history`, `.mysql_history` and similar files with commands or credentials someone typed years ago. Don't share the archive outside the team.
- Never commit real credentials. Everything in this repo uses placeholders (`CHANGE_ME*`, `FTP_USER`, `example.com`, `203.0.113.x`).
- Passwords given on the command line (`mysql -pSECRET`) are visible to anyone on the host via `ps`. Use credentials from the container's environment instead, as the refresh script does, and keep them out of shell history and chat logs.
- Security plugin log tables are full of visitor IP addresses. Leaving them out of the long-term archive is usually the better choice from a data-protection point of view.
- Remove temporary SSH keys and IP allowlist entries from the hosting panel once you're done.
- When the archive is no longer needed, delete the VM *and* the long-term archive according to your retention policy.

---
