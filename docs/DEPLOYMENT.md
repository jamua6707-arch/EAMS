# Deploying SEAMS to https://seams.garsame.com

Step-by-step guide for putting SEAMS on the Ubuntu 24.04 server
(`onepercenttechteamserver`, IP `104.237.2.66`) that already hosts other sites
in `/var/www` (`baroplatform`, `cscareer`, ...).

## How it fits together

```
Browser ──HTTPS──> Nginx (seams.garsame.com)
                     ├── /       -> static files in /var/www/seams/frontend/dist  (React build)
                     └── /api/*  -> http://127.0.0.1:5050                          (Express API, run by PM2)
                                          └── MongoDB on 127.0.0.1:27017 (database: seams_db)
```

- No application code changes are needed: the frontend already calls the API
  at the relative path `/api`, and Nginx serves both from the same domain.
- The API listens on `127.0.0.1` only — it is never exposed directly to the internet.
- HTTPS is required, not optional: the phone camera ID scanner only works on a
  secure (`https://`) page.

> All commands below run **on the server** as `root` unless marked
> "(your PC)". Values you must choose yourself are marked `<LIKE_THIS>`.

---

## Step 0 — DNS (already done)

Namecheap → `garsame.com` → Advanced DNS has:

| Type | Host  | Value          |
|------|-------|----------------|
| A    | seams | 104.237.2.66   |

Check it resolves (should print `104.237.2.66`):

```bash
dig +short seams.garsame.com
```

---

## Step 1 — Look at what the server already has

The server runs other apps, so check before installing anything:

```bash
node -v                      # need v18+ (v20 or v22 recommended)
pm2 -v                       # process manager, probably already used by other apps
nginx -v                     # web server
systemctl status nginx --no-pager | head -5
systemctl status mongod --no-pager | head -5
certbot --version
ss -tlnp | grep -E ':5050|:27017|:80 |:443 '   # 5050 must be FREE for SEAMS
docker ps 2>/dev/null        # only matters if Nginx turns out to run in Docker
```

- If anything is already listening on **5050**, pick another free port and use
  it everywhere this guide says `5050` (backend `.env` and the Nginx config).
- If Nginx runs **inside Docker** instead of as a system service, skip the
  Nginx part of Step 6 and add the server block from Step 6 to that container's
  config instead (and use its certbot setup — `/var/www/certbot` suggests the
  webroot method).

## Step 2 — Install whatever is missing

Only run the parts Step 1 showed are missing.

**Node.js 22 + PM2**

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs
npm install -g pm2
```

**Nginx + Certbot**

```bash
apt-get update
apt-get install -y nginx certbot python3-certbot-nginx git
```

**MongoDB 8.0** (Ubuntu 24.04 "noble")

```bash
apt-get install -y gnupg curl
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" \
  > /etc/apt/sources.list.d/mongodb-org-8.0.list
apt-get update
apt-get install -y mongodb-org
systemctl enable --now mongod
```

Make sure MongoDB only listens locally (default). This should show
`bindIp: 127.0.0.1`:

```bash
grep bindIp /etc/mongod.conf
```

## Step 3 — Get the code from GitHub

```bash
cd /var/www
git clone https://github.com/jamua6707-arch/EAMS.git seams
cd /var/www/seams
git log --oneline -3
```

If the repository is private, GitHub will ask for a username and a
**personal access token** (not your GitHub password), or set up a deploy key.

## Step 4 — Configure and start the backend (API)

Generate a long random secret and copy the output:

```bash
openssl rand -hex 48
```

Create the production environment file:

```bash
cd /var/www/seams/backend
nano .env
```

Paste this, replacing `<PASTE_RANDOM_SECRET_HERE>`:

```ini
PORT=5050
HOST=127.0.0.1
MONGO_URI=mongodb://127.0.0.1:27017/seams_db
JWT_SECRET=<PASTE_RANDOM_SECRET_HERE>
JWT_EXPIRES_IN=8h
CLIENT_URL=https://seams.garsame.com
```

Lock the file down, install dependencies:

```bash
chmod 600 .env
npm ci --omit=dev
```

### First-time data (run ONCE, on an empty database only)

The app needs at least one admin account to log in. The seed script creates
the demo accounts and sample data:

```bash
npm run seed
```

> **WARNING — `npm run seed` deletes ALL users, students, exams and attendance
> before re-creating the demo data.** Run it only now, on the brand-new
> database. Never run it again once real data exists.

Start the API with PM2 and make it survive reboots:

```bash
pm2 start server.js --name seams-api
pm2 save
pm2 startup          # if it prints a command, copy-paste and run that command
```

Test it from the server itself:

```bash
curl http://127.0.0.1:5050/api/health
# {"success":true,"message":"Smart Examination Attendance API is running"}
pm2 logs seams-api --lines 20     # should show "MongoDB connected"
```

## Step 5 — Build the frontend

```bash
cd /var/www/seams/frontend
npm ci
npm run build
ls dist          # index.html + assets/
```

No `frontend/.env` is needed: the build uses `/api` by default, which Nginx
routes to the backend.

## Step 6 — Nginx site for seams.garsame.com

```bash
nano /etc/nginx/sites-available/seams.garsame.com
```

Paste:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name seams.garsame.com;

    root /var/www/seams/frontend/dist;
    index index.html;

    # For certbot's webroot method (harmless if you use the --nginx plugin)
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # API -> Express (PM2)
    location /api/ {
        proxy_pass http://127.0.0.1:5050;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 5m;
    }

    # Hashed build assets can be cached for a long time
    location /assets/ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # React Router: every other path serves index.html
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Enable it, test the config, reload (reload does not interrupt the other sites):

```bash
ln -s /etc/nginx/sites-available/seams.garsame.com /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

Check it over plain HTTP before adding the certificate:

```bash
curl -I http://seams.garsame.com
curl http://seams.garsame.com/api/health
```

## Step 7 — HTTPS certificate (Let's Encrypt)

```bash
certbot --nginx -d seams.garsame.com
```

Choose the option to **redirect HTTP to HTTPS**. Certbot edits the Nginx file
for you and sets up auto-renewal. Confirm renewal works:

```bash
certbot renew --dry-run
```

## Step 8 — Firewall

Only web traffic and SSH should be open — **not** 5050 or 27017:

```bash
ufw status
# if ufw is active and these are missing:
ufw allow OpenSSH
ufw allow 'Nginx Full'
```

## Step 9 — Verify and secure the accounts

1. Open **https://seams.garsame.com** — padlock, login page appears.
2. Log in as `admin1` / `Admin@123`.
3. **Immediately** go to **Manage Users** and reset the password of every demo
   account (`admin1`, `admin2`, `invig1`, `invig2`, `invig3`), or deactivate the
   ones you don't need. These default passwords are public in the README.
4. Replace the demo students/exams with real ones (or delete them) via the
   admin screens.
5. On a phone, log in as an invigilator, open **Scan Attendance** and confirm
   the camera starts (it only works because the site is on HTTPS).

---

## Updating the live site later

Every time new code is pushed to GitHub `main`:

```bash
cd /var/www/seams
git pull origin main

cd backend
npm ci --omit=dev
pm2 restart seams-api

cd ../frontend
npm ci
npm run build          # Nginx serves the new dist/ immediately, no reload needed
```

Never run `npm run seed` during an update.

## Backups

Dump the database (do this before risky updates, and ideally daily via cron):

```bash
mkdir -p /root/backups
mongodump --db seams_db --archive=/root/backups/seams_db-$(date +%F).gz --gzip
```

Restore:

```bash
mongorestore --archive=/root/backups/seams_db-YYYY-MM-DD.gz --gzip --drop
```

## Troubleshooting

| Symptom | Check |
|---|---|
| `502 Bad Gateway` on `/api/...` | `pm2 status`, `pm2 logs seams-api` — API crashed or wrong port in Nginx |
| API log says `MongoDB connection error` | `systemctl status mongod`, `MONGO_URI` in `backend/.env` |
| Login fails with "Not allowed by CORS" | `CLIENT_URL` must be exactly `https://seams.garsame.com` (no trailing slash); `pm2 restart seams-api` after editing |
| Refreshing a page like `/exams` gives 404 | the `try_files ... /index.html` line is missing from the Nginx config |
| Camera does not start on phone | site must be opened via `https://`; check the browser's camera permission |
| Certbot fails | `dig +short seams.garsame.com` must return `104.237.2.66`; port 80 must be reachable |
