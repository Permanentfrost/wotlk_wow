# AzerothCore WotLK on Proxmox (Debian VM) + Steam Deck Client (Proton Experimental)

*A full end-to-end guide including common pitfalls you could hit during set up of a private World of Warcraft Server to play from your Steam Deck.*

Trust me... I fell into all of them: Docker Compose v2 issue, permissions, long client-data init, realm IP wrongly assigned as 127.0.0.1, etc..

Bear in mind: the server is most important part of this guide as you can GPT your way of setting up the client installation on any plattform (Windows, other Linux Distros and MacOS). 


### The Game Plan (What this guide covers)
* **Prep the Server:** Spin up a well-optimized Debian VM on Proxmox.
* **Build the Core:** Safely install Docker, clone AzerothCore, and compile the
server (using `tmux` so you don't lose progress).
* **Account Setup:** Launch the worldserver and create your in-game admin account.
* **Steam Deck Config:** Point your `realmlist.wtf` to the VM and run the WoW client via SteamOS Proton Experimental.
* **Squash the Bugs:** Fix the "`127.0.0.1` database routing issue" so your Deck can actually log in.
* **Protect Your Loot:** Set up automated weekly security patches and daily database backups so you never lose your progress.


---

## Target setup (what this guide assumes)

- **Host:** Proxmox VE
- **Server VM:** Debian (headless), running AzerothCore WotLK via Docker 
- **Client device:** Steam Deck (SteamOS) running the WoW 3.3.5a client via **Proton Experimental**
- **Network:** Home LAN (e.g., `192.168.1.0/24`), server VM has a *fixed IP* (static or DHCP reservation) - for this guide we take "192.168.1.118" as static VM IP

---

## 0) Proxmox VM sizing + best-practice tweaks

### Recommended VM resources (stable “set it & forget it”)
- **vCPU:** 4  
- **RAM:** 8 GB  
- **Disk:** ~80 GB (SSD-backed storage recommended)

### Proxmox tweaks that improve performance/stability
- **CPU type:** `host` (best compile + runtime performance)
- **RAM ballooning:** disable (avoids weird pauses under memory pressure)
- **Disk:** `virtio-scsi single` + SSD emulation + discard/TRIM (if your storage supports it)
- **NIC model:** VirtIO (best throughput/latency)

### Fixed IP for the VM
If you already assigned a fixed IP then good. Otherwise locate your routers settings and assign a static IP to the VM/server. Note it down.

Example VM IP used in this guide: `192.168.1.118`

General tip regarding passwords: 
Normally I'd be writing down security related stuff and tipps here but honestly I kept it pretty easy through the whole set up for me. 
Just to simplify your set up and testing with "user-easy" usernames and passwords - otherwise you question your sanity (because I did!). 
**Also:** You are **local LAN only** (hopefully) so ideally attack vectors are pretty much non-existent.  

**So that means-->** In this guide I took username `banana` and pw `banana`just to keep it simple :-) 

---

## 1) SSH workflow: don’t lose long-running tasks (tmux)

If you SSH from a laptop and disconnect or close the lid, **foreground commands can be killed**.
Use **tmux** for anything long-running (clone/build).

```bash
sudo apt update
sudo apt install -y tmux
tmux new -s acore
```

Detach (leave tasks running): **Ctrl + b**, then **d**  
Re-attach later:
```bash
tmux attach -t acore
```
List sessions:
```bash
tmux ls
```

---

## 2) Install Docker correctly on Debian (Compose v2 plugin matters)

### Why this matters (pitfall I hit)
AzerothCore’s Docker wrapper uses **`docker compose ...`** (Compose v2).
If you install the Debian `docker.io` packages only, you may **not** get the Compose v2 plugin, leading to errors like:
- `docker: 'compose' is not a docker command`
- `Unable to locate package docker-compose-plugin` (if your APT sources don’t expose it)

**Solution:** install Docker from Docker’s official Debian repository (includes compose plugin).

### Install Docker Engine + Compose v2 plugin (recommended)

1) Remove potentially conflicting packages:
```bash
sudo apt remove -y docker.io docker-compose docker-doc podman-docker containerd runc || true
```

2) Install prerequisites:
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
```

3) Add Docker’s GPG key + repo:
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4) Install Docker + plugins:
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

5) Verify:
```bash
docker --version
docker compose version
```

### Docker permissions (pitfall I hit)
If you see:
`permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock`

Fix it properly:
```bash
sudo usermod -aG docker "$USER"
# log out and back in OR run:
newgrp docker
```

Test:
```bash
docker ps
```

(Temporary quick fix is prefixing commands with `sudo`, but group membership is of course much cleaner.)

---

## 3) Clone AzerothCore (folder naming + tmux)

Inside tmux:
```bash
cd ~
git clone https://github.com/azerothcore/azerothcore-wotlk.git --branch master --single-branch azerothcore
cd azerothcore
```

**Folder naming note (pitfall you asked about):**  
If you omit the final folder argument, Git clones into `azerothcore-wotlk/`.  
This is **not bad**—it only changes the folder name.

---

## 4) Build + start the server (Docker)

### Install Docker configuration
```bash
./acore.sh docker install
```

You may see:
`NOTICE: file </home/<user>/azerothcore/conf/config.sh> not found, we use default configuration only.`  
That’s **fine** (defaults are used).

### Build (compile) the images
This step can take **15–45+ minutes**, depending on CPU/storage.
```bash
./acore.sh docker build
```

### Start the services (detached)
```bash
./acore.sh docker start:app:d
```

Verify:
```bash
docker ps
```

You should see something like:
- `ac-database` (healthy)
- `ac-authserver`
- `ac-worldserver`

---

## 5) The “client data init” step can take forever (and that’s normal)

### Pitfall I hit
You may see a container like:
- `ac-client-data-init` sitting in “Waiting” for a long time

This is often downloading/extracting/preparing large client data artifacts. First run can be **10–60+ minutes** (sometimes longer on slow storage/network).

Watch progress:
```bash
docker logs -f ac-client-data-init
```

As long as logs continue and the container shows activity, you’re fine.

---

## 6) Create your account (and optionally GM)

Your active containers will look like (example):
- `ac-authserver`
- `ac-worldserver`
- `ac-database`

### Attach to the worldserver console
```bash
docker attach ac-worldserver
```

At the `AC>` prompt:
```text
account create banana banana
```

Optional GM (admin) privileges:
```text
account set gmlevel banana 3 -1
```

**What does that mean?**
- `3` = high GM/admin level (very powerful)
- `-1` = apply to all realms

**Recommendation for normal solo play:**  
You can skip GM entirely and add it later. You can also remove it later:
```text
account set gmlevel banana 0 -1
```

Detach from the console **without stopping the server**:
- **Ctrl + p**, then **Ctrl + q**

Confirm it’s still running:
```bash
docker ps
```

---

## 7) Steam Deck client setup (Proton Experimental)

### Put the client folder anywhere in your home (this is fine)
Example:
`/home/deck/Games/Wow/ChromieCraft3.3.5client/`  
As long as it contains:
- `Wow.exe`
- `Data/`
- `Interface/`
- etc.

Tip: Transfer your pre-installed WotLK client (the whole folder!) from your PC to your Steam Deck using a USB-C flash drive, miniSD Card or via network transfer. 

### Set realmlist.wtf
Edit one of:
- `Data/enUS/realmlist.wtf` or
- `Data/enGB/realmlist.wtf`

Set it to your **VM IP**:
```text
set realmlist 192.168.1.118
```

### Add it to Steam as a Non-Steam Game
1) Steam → **Library** → **Add a Game** → **Add a Non-Steam Game**
2) Select `Wow.exe`
3) Right-click the entry → **Properties** → **Compatibility**
4) Check **Force the use of a specific Steam Play compatibility tool**
5) Choose **Proton Experimental**
6) Launch

---

## 8) Realm selection “doesn’t accept” (major pitfall I hit + fix)

### Symptom (what I saw)
- Login works (username/password OK)
- Realm list shows “AzerothCore”
- Clicking realm doesn’t enter / appears to do nothing

### Root cause
Auth works on `3724`, but the realm list in the DB was advertising **`127.0.0.1:8085`**:
That tells the client to connect to *itself* (Steam Deck), not the VM. Remember --> we are on `192.1.168/24`, not the above. 

### Check realm advertisement in the database
On the VM:
```bash
docker exec -it ac-database printenv MYSQL_ROOT_PASSWORD
```

Then:
```bash
docker exec -it ac-database mysql -uroot -p -e "USE acore_auth; SELECT id,name,address,port FROM realmlist;"
```

If it shows `127.0.0.1`, fix it (replace IP with your VM’s LAN IP):
```bash
docker exec -it ac-database mysql -uroot -p -e "USE acore_auth; UPDATE realmlist SET address='192.168.1.118', port=8085 WHERE id=1; SELECT id,name,address,port FROM realmlist;"
docker restart ac-authserver ac-worldserver
```

After this, realm selection should work immediately.

> [!WARNING]
> Above SQL update query specifies `WHERE id=1`. This is perfectly fine for a fresh install (which we assume here).
However, if you messed around and created a second realm during setup, this ID might be 2 or 3. 


---

## 9) Firewall / Ports

Ensure these are reachable from your Steam Deck to the VM:
- **3724/tcp** (Auth)
- **8085/tcp** (World)

In your `docker ps` you should see mappings like:
- `0.0.0.0:3724->3724/tcp`
- `0.0.0.0:8085->8085/tcp`

---

## 10) Safe shutdown + Proxmox snapshots (recommended workflow)

Snapshots are safest when the VM is **powered off** (consistent DB state).

### Recommended weekly routine
1) Stop playing
2) **Shut down the VM cleanly**
3) Take Proxmox snapshot (or better: scheduled Proxmox backup)
4) Boot VM when you want to play again

This does **not** damage Docker or AzerothCore.

### Starting the stack after reboot
```bash
cd ~/azerothcore
./acore.sh docker start:app:d
```

If you ever want to fully stop it:
```bash
cd ~/azerothcore
docker compose down
```

> [!CAUTION]
> Watch the control flags `-v` !! DO NOT run docker `compose down -v` unless you intend to` completely wipe your databases and server volumes. 

---

## 11) Quick troubleshooting cheatsheet

### See what’s running
```bash
docker ps
```

### View logs
```bash
docker logs -f ac-worldserver
docker logs -f ac-authserver
docker logs -f ac-client-data-init
```

### Restart services
```bash
docker restart ac-authserver ac-worldserver
```

### Realm IP fix (most common “realm click doesn’t work”)
```bash
docker exec -it ac-database mysql -uroot -p -e "USE acore_auth; SELECT id,name,address,port FROM realmlist;"
# then update address to your VM IP
```

---

## You’re done ✅

At this point you have:
- A Debian VM running AzerothCore via Docker
- A Steam Deck client running via Proton Experimental
- Realmlist set to the VM IP
- Realm address in the DB set to the VM IP (and not 127.0.0.1)

Happy adventuring and hope the nostalgia hits ;) 🏔️⚔️



Almost forgot this addition: 

# 12) Security + Backups (minimal impact, low effort)

This chapter adds **basic hardening** (without breaking LAN gameplay) and a **space-friendly backup plan**. 
At the time of writing (February 2026) prices for RAM + SSDs are enormous so trying to keep a low overhead. 

---

## Question you might have: Is my VM/server exposed to the Internet?

In a typical home LAN, your WoW server is **not Internet-exposed** unless you configured **router port forwarding (NAT)** to the VM (or put the VM in a DMZ / gave it a public IP).

Of course you can always 
**Check what’s listening on the VM (ports + owning processes):**
```bash
sudo ss -tulpn | egrep ':(22|3724|8085)\b'
```
- `ss`: socket status
- `-tulpn`: **t**cp, **u**dp, **l**istening, **p**rocess, **n**umeric ports/IPs
- `egrep ...`: filter for SSH (22), Auth (3724), World (8085)

**Check what Docker publishes (your “attack surface” on the VM’s network interfaces):**
```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```
If you see `0.0.0.0:3724->3724/tcp` / `0.0.0.0:8085->8085/tcp`, that’s normal for **LAN access**.  
Again: To confirm Internet exposure, check your **router** for any **port forward rules** to your VM IP on **3724/8085**. I find this to be the best source of information. 

---

## 12.2 Firewall (UFW): allow only LAN access

This keeps gameplay working (Steam Deck on the same LAN) while blocking inbound traffic from anywhere else.

```bash
sudo apt update
sudo apt install -y ufw

# deny all inbound by default; allow outbound
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH only from LAN (tighten further to your admin PC IP if you want)
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

# WoW ports only from LAN
sudo ufw allow from 192.168.1.0/24 to any port 3724 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8085 proto tcp

sudo ufw enable
sudo ufw status verbose
```

Notes:
- `from 192.168.1.0/24` restricts access to **your LAN subnet** (adjust if yours differs).
- `proto tcp` is explicit (WoW Auth/World are TCP in this setup).

> [!IMPORTANT]
> Apparently Docker so somehow bypasses UFW (not sure how though). This firewall setup
restricts non-Docker services (like SSH), but Docker containers will still be accessible to
your entire LAN.


---

## 12.3 Automatic security updates (weekly ~04:00)

Install unattended upgrades:
```bash
sudo apt update
sudo apt install -y unattended-upgrades
```
- `-y` auto-confirms prompts (non-interactive install)

Enable it (creates default config):
```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```
- `-plow` uses the low-priority dialog flow (simple prompts)

### Run updates weekly via systemd timer 

1) Create override directories:
```bash
sudo mkdir -p /etc/systemd/system/unattended-upgrades.service.d
sudo mkdir -p /etc/systemd/system/unattended-upgrades.timer.d
```

2) Ensure the service runs the upgrade command:
```bash
sudo tee /etc/systemd/system/unattended-upgrades.service.d/override.conf > /dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/unattended-upgrade
EOF
```
- The blank `ExecStart=` line **clears** the original command so we can replace it.

3) Schedule weekly at 04:00 (Sunday) + small random delay:
```bash
sudo tee /etc/systemd/system/unattended-upgrades.timer.d/override.conf > /dev/null <<'EOF'
[Timer]
OnCalendar=
OnCalendar=Sun *-*-* 04:00:00
Persistent=true
RandomizedDelaySec=15m
EOF
```
- `OnCalendar=Sun *-*-* 04:00:00`: weekly every Sunday at 04:00
- `Persistent=true`: if the VM was off, it runs after next boot
- `RandomizedDelaySec=15m`: spreads load (useful if many hosts patch)

4) Reload + enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now unattended-upgrades.timer
systemctl list-timers | grep unattended
```

---

## 12.4 Backups (Important!) 

We have two of these, once the VM on Proxmox level and once the DB inside the VM - your gold and items ;-). 
Remember: below applies to how I play (mostly weekends). 

### A) Proxmox VM backups (weekly, keep only 3)

Because storage is limited and I play is mostly on weekends:
- run **one Proxmox backup per week**
- keep only the last **3** backups (≈ last 3 weeks)

In Proxmox: **Datacenter → Backup → Add**
- Schedule: weekly (e.g., Sunday night / early morning)
- Mode: **Stop** is the most consistent for databases (Snapshot is okay, but less strict)
- Retention: **Keep last = 3**

---

### B) Database backups (daily 03:15, keep only a few)

Back up only what matters most:
- `acore_auth`
- `acore_characters`

Create folders:
```bash
mkdir -p ~/acore_backups/sql
```

Create the dump script:
```bash
#!/usr/bin/env bash

set -euo pipefail
# ^ Safety settings (strongly recommended in scripts):
#   -e  : exit immediately if any command fails (non-zero exit code)
#   -u  : treat unset variables as an error (catches typos / missing vars)
#   -o pipefail : if you pipe commands (A | B), the script fails if ANY part fails

OUTDIR="$HOME/acore_backups/sql"
# ^ Where backups will be stored on the VM. "$HOME" is your user's home folder.
#   Example path: /home/banana/acore_backups/sql

KEEP_DAYS=7
# ^ Retention: keep backups for 7 days. Older ones will be deleted.

DBCONT="ac-database"
# ^ Name of the Docker container running the database (MariaDB/MySQL) in AzerothCore.
#   This matches what you see in "docker ps" (usually ac-database).

mkdir -p "$OUTDIR"
# ^ Create the backup folder if it doesn't exist yet.
#   -p means: also create parent folders, and don't error if it already exists.

DBPASS="$(docker exec "$DBCONT" printenv MYSQL_ROOT_PASSWORD | tr -d '\r\n')"
# ^ Read the MySQL root password from INSIDE the DB container:
#   - docker exec <container> <command> runs a command inside the container.
#   - printenv MYSQL_ROOT_PASSWORD prints the value of that environment variable.
#   - tr -d '\r\n' strips line breaks (carriage return + newline) so the password
#     becomes a clean single-line string we can safely use.

[[ -n "$DBPASS" ]] || { echo "Could not read MYSQL_ROOT_PASSWORD from $DBCONT" >&2; exit 1; }
# ^ Sanity check:
#   [[ -n "$DBPASS" ]] means "is DBPASS non-empty?"
#   If it IS empty, we:
#     - print an error message to stderr (>&2)
#     - exit with error code 1
#   This prevents creating empty/invalid dumps if the password can't be read.

for db in acore_auth acore_characters; do
  # ^ Loop over the two databases we actually care about:
  #   acore_auth       = accounts / realm login related data
  #   acore_characters = your characters, gold, items, etc.
  #   (acore_world is big and usually rebuildable, so we skip it to save space.)

  out="$OUTDIR/${db}-$(date +%F).sql"
  # ^ Output filename for this database dump.
  #   $(date +%F) produces YYYY-MM-DD (e.g., 2026-02-18)
  #   Example: acore_characters-2026-02-18.sql

  docker exec "$DBCONT" sh -c \
    "mysqldump -uroot -p\"$DBPASS\" --single-transaction --quick --routines --triggers $db" \
    > "$out"
  # ^ This creates a SQL dump of the database from inside the container:
  #   - docker exec "$DBCONT" sh -c "..." runs a shell in the container and executes the string.
  #
  #   mysqldump options explained:
  #   -uroot                 : connect as MySQL user "root"
  #   -p"$DBPASS"            : use the password we read earlier
  #   --single-transaction   : makes a consistent snapshot for InnoDB without stopping the DB
  #                           (best practice for online backups)
  #   --quick                : stream rows instead of buffering them (uses less RAM)
  #   --routines --triggers  : include stored routines and triggers (if present)
  #
  #   The "> $out" at the end:
  #   - redirects the dump output (stdout) into a file on the VM (outside the container).
done

find "$OUTDIR" -maxdepth 1 -type f -name "*.sql" -print0 | xargs -0 -r gzip -9f
# ^ Compress any newly created .sql files to save space:
#   find ... -name "*.sql"   : locate SQL dump files in OUTDIR
#   -maxdepth 1              : only this folder (not subfolders)
#   -type f                  : files only
#   -print0                  : safe output even if filenames contain spaces
#   xargs -0                 : read that safe (NUL-separated) list
#   -r                       : do nothing if the list is empty (prevents errors)
#   gzip -9f                 : compress with max compression (-9) and overwrite (-f) if needed
# Result: *.sql becomes *.sql.gz and the original .sql is replaced.

find "$OUTDIR" -type f -name "*.gz" -mtime +"$KEEP_DAYS" -delete
# ^ Retention cleanup (keep disk usage bounded):
#   -type f                  : only files
#   -name "*.gz"             : only compressed backups
#   -mtime +7                : files older than 7 * 24 hours
#   -delete                  : remove them
#
# Note: -mtime is "age in days" (24h blocks). This is close to "keep 7 days",
# which is what you want for simple rolling retention.
```

make it executable: 
`chmod +x dump-acore-db.sh`

Test once:
```bash
~/acore_backups/dump-acore-db.sh
ls -lh ~/acore_backups/sql
```

Schedule daily at **03:15**:
```bash
crontab -e
```

Add:
```cron
15 3 * * * /bin/bash -lc '/home/banana/acore_backups/dump-acore-db.sh >/dev/null 2>&1'
```

**Restore example (during rebuild):** --> hope this never happens but it it should, no worries, we'll fix it ;) 
```
# Read DB root password from inside the running DB container and store it in a variable.
# - DBPASS is just a variable name ("box") that holds the password as text.
# - $(...) runs the command and captures its output.
# - docker exec ... printenv MYSQL_ROOT_PASSWORD prints the env var from the container.
# - tr -d '\r\n' removes line breaks so the password is clean.

DBPASS="$(docker exec ac-database printenv MYSQL_ROOT_PASSWORD | tr -d '\r\n')"

# Restore: stream the compressed SQL dump directly into MySQL inside the container.
# - gunzip -c outputs decompressed SQL to stdout (does NOT create a .sql file).
# - | pipes that SQL into the next command.
# - docker exec -i keeps stdin open so MySQL can read the piped SQL.
# - mysql -uroot -p"$DBPASS" connects as root using the password from DBPASS.
# - acore_characters is the target database to import into.
gunzip -c ~/acore_backups/sql/acore_characters-YYYY-MM-DD.sql.gz \
  | docker exec -i ac-database mysql -uroot -p"$DBPASS" acore_characters

```


# 13) Optional: Add Playerbots + Auction House Bot (Solo-friendly “alive world”)

So after a while this might strike you as boring since of course a big part of the fun are other players and functions like an auction house. 
Therefore this chapter upgrades our standard AzerothCore Docker setup with:

- **Playerbots**: These are NPC “players” that level, group, quest, etc.
- **Auction House Bot**: keeps the Auction House populated

> [!IMPORTANT]
> Playerbots and an Auction House Bot are **separate modules**. Playerbots alone does **not** auto-fill the AH. (At first I thought it is, so just FYI.)

---

## 13.1 Before starting back it up (don’t lose your character / progress)

1) **DB backup (point of no return)**  
Run the DB dump script once before migrating anything:
```bash
~/acore_backups/dump-acore-db.sh

#verify its there by just using ls
ls -lh
```

2) **Stop containers safely**  
Some `acore.sh` wrappers do **not** support `docker stop` / `docker install` and will print `Unknown or empty arg`.
This happened to me and azerothcore actually prints the solution I just didnt get it first. 

Use Docker Compose directly:
```bash
cd ~/azerothcore
docker compose down --remove-orphans
```

> [!WARNING]
> Again: **Never** use `docker compose down -v` unless you want to wipe your DB volumes.

---

## 13.2 Install Playerbots (Playerbot branch + module)

I mean, yes you ideally have a) the proxmox snapshot and b) the backup job you did separately, but I **love** redundancy when it comes to backups therefore: 

### A) Keep an “undo button”
```bash
cd ~
mv azerothcore azerothcore_old
```

### B) Clone the Playerbot branch (fresh install)
```bash
cd ~
git clone https://github.com/mod-playerbots/azerothcore-wotlk.git --branch Playerbot azerothcore
```

### C) Add the playerbots module
```bash
mkdir -p ~/azerothcore/modules
cd ~/azerothcore/modules
git clone https://github.com/mod-playerbots/mod-playerbots.git --branch master
```

### D) Mount modules into the worldserver container
Create:
```bash
nano ~/azerothcore/docker-compose.override.yml
```

Paste:
```yaml
services:
  ac-worldserver:
    volumes:
      - ./modules:/azerothcore/modules:ro
```

### E) Build + start
```bash
cd ~/azerothcore
./acore.sh docker build
./acore.sh docker start:app:d
docker compose ps
```

---

## 13.3 Configure Playerbots (reduce load + make grouping less annoying)

### A) Find the correct config folder (pitfall I stumbled into)
Depending on your Docker env, configs may live under **`env/dist`** (not `env/docker`).

Check:
```bash
ls -la ~/azerothcore/env
ls -la ~/azerothcore/env/dist/etc/modules 2>/dev/null || true
```

### B) Create your real config from the `.dist` template
```bash
cp ~/azerothcore/env/dist/etc/modules/playerbots.conf.dist \
   ~/azerothcore/env/dist/etc/modules/playerbots.conf
nano ~/azerothcore/env/dist/etc/modules/playerbots.conf
```

### C) Recommended settings (solo-friendly)
- Cap bots at **50** (less lag)
- Make bots accept group invites (fixes “you’re too low level” which means they'll just deny you)

```ini
AiPlayerbot.RandomBotAutologin = 1
AiPlayerbot.MinRandomBots = 50
AiPlayerbot.MaxRandomBots = 50

# 0=GM only, 1=level-based, 2=always accept
AiPlayerbot.GroupInvitationPermission = 2
```

Restart:
```bash
docker restart ac-worldserver
```

---

## 13.4 Add an Auction House Bot (AHBot) so the AH is not empty

### Why this is needed
Playerbots ≠ AH population. If you want a living AH, add the separate **mod-ah-bot** module.

### A) Install module
```bash
cd ~/azerothcore/modules
git clone https://github.com/azerothcore/mod-ah-bot.git
```

Rebuild (new C++ module):
```bash
cd ~/azerothcore
./acore.sh docker build
./acore.sh docker start:app:d
```

### B) Import required SQL into `acore_world` (critical step)
```bash
DBPASS="$(docker exec ac-database printenv MYSQL_ROOT_PASSWORD | tr -d '\r\n')"
docker exec -i ac-database mysql -uroot -p"$DBPASS" acore_world \
  < ~/azerothcore/modules/mod-ah-bot/data/sql/db-world/mod_auctionhousebot.sql
```

### C) Create an AHBot account + character
In worldserver console:
```bash
docker attach ac-worldserver
```
Then:
```text
account create ahbot ahbot
```
Detach: **Ctrl+p**, then **Ctrl+q**  
Log into WoW once as `ahbot/ahbot`, create a character (e.g., `Ahbot`), enter world once, log out.

> [!IMPORTANT]
> Its essential you create this `Ahbot` character, enter the world and walk around a few seconds, I've read of some glitches otherwise. 


### D) Create the real config (don’t edit only the `.dist`)
Common pitfalls:
- Don’t put your final config only in `~/azerothcore/modules/mod-ah-bot/conf/` (that’s just the template).
- Don’t use `/azerothcore/...` on the VM (that path is inside containers). Use `~/azerothcore/...` on the host.

```bash
mkdir -p ~/azerothcore/env/dist/etc/modules
cp ~/azerothcore/modules/mod-ah-bot/conf/mod_ahbot.conf.dist \
   ~/azerothcore/env/dist/etc/modules/mod_ahbot.conf
nano ~/azerothcore/env/dist/etc/modules/mod_ahbot.conf
```

Set:
```ini
AuctionHouseBot.EnableSeller = 1
AuctionHouseBot.EnableBuyer  = 1
AuctionHouseBot.Account      = <ACCOUNT_ID>
AuctionHouseBot.GUID         = <CHAR_GUID>
```

Get `<ACCOUNT_ID>` + `<CHAR_GUID>`:
```bash
DBPASS="$(docker exec ac-database printenv MYSQL_ROOT_PASSWORD | tr -d '\r\n')"

docker exec -i ac-database mysql -uroot -p"$DBPASS" -e "
SELECT id, username FROM acore_auth.account WHERE username='AHBOT';
SELECT guid, name, account FROM acore_characters.characters
WHERE account=(SELECT id FROM acore_auth.account WHERE username='AHBOT');
"
```

> [!IMPORTANT]
> Bot fields `AuctionHouseBot.Account      = <ACCOUNT_ID>` and also `AuctionHouseBot.GUID         = <CHAR_GUID>` need to be populated. Then and only then proceed with below :-) . 


Restart:
```bash
docker restart ac-worldserver
```

### E) Verify AH is being populated
```bash
DBPASS="$(docker exec ac-database printenv MYSQL_ROOT_PASSWORD | tr -d '\r\n')"
docker exec -i ac-database mysql -uroot -p"$DBPASS" -e "
USE acore_characters;
SELECT COUNT(*) AS auctions_total FROM auctionhouse;
"
```

---

## 13.5 Optional: make the AH “bigger” (more leveling gear)
AHBot’s “how many items should exist” is controlled in:
`acore_world.mod_auctionhousebot` (`minitems`/`maxitems` per AH).

Example (Alliance/Horde/Neutral = 2/6/7):
```bash
DBPASS="$(docker exec ac-database printenv MYSQL_ROOT_PASSWORD | tr -d '\r\n')"
docker exec -i ac-database mysql -uroot -p"$DBPASS" -e "
USE acore_world;
UPDATE mod_auctionhousebot
SET minitems=3000, maxitems=4000
WHERE auctionhouse IN (2,6,7);
SELECT auctionhouse,name,minitems,maxitems FROM mod_auctionhousebot;
"
docker restart ac-worldserver
```

---

## 13.6 Again: Quick pitfalls I hit (hope this saves your sanity)
- **`./acore.sh docker stop/install` may not exist** → use `docker compose down/stop`.
- **Config paths differ**: your setup used `env/dist/...` (not `env/docker/...`).
- **`.dist` files are templates** → copy to `.conf` (your real settings).
- **Playerbots does not include AH population** → install **mod-ah-bot** separately.
- **Seeing “only bots” in SQL**: don’t query “top 50 by level”, just query characters by your account if needed.

---

## 13.7 Troubleshooting (3 commands)

### 1) Logs (Playerbots/AHBot)
```bash
docker logs --tail 200 ac-worldserver | egrep -i "playerbot|ahbot|auctionhousebot|error|warn"
```

### 2) AH health-check (DB auction count)
```bash
DBPASS="$(docker exec ac-database printenv MYSQL_ROOT_PASSWORD | tr -d '\r\n')"
docker exec -i ac-database mysql -uroot -p"$DBPASS" -e "USE acore_characters; SELECT COUNT(*) AS auctions_total FROM auctionhouse;"
```

### 3) Confirm configs exist inside the container
```bash
docker exec -it ac-worldserver ls -la /azerothcore/env/dist/etc/modules | egrep -i "playerbots|ahbot|auction"

