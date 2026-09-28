# Deploying hustlefrolic.com to Hostinger (Business shared hosting)

The site is a normal Laravel app. Code is built on GitHub (Composer + Vite), uploaded over SSH,
and switched live with a symlink. Nothing is compiled on the server.

```
~/hf/staging/                      ~/hf/production/        (same layout, created later)
├── releases/20260928120000/       one folder per deploy (last 5 kept)
├── current -> releases/2026...    what the website serves
├── shared/.env                    the ONLY .env - never in Git
├── shared/storage/                logs, sessions, cache, uploads/ (kept between deploys)
└── backups/db-*.sql.gz            automatic dump before every migration (last 14 kept)

staging web root  ->  ~/hf/staging/current/public       (symlink)
```

---

## 1. One-time server setup (about 20 minutes)

### 1.1 PHP version and extensions
hPanel → **Advanced → PHP Configuration**
- PHP version: **8.3 or 8.4** (Laravel 13 needs 8.3+).
- Extensions tab, make sure these are ticked: `intl`, `mbstring`, `gd` (or `imagick`), `zip`, `fileinfo`,
  `pdo_mysql`, `sodium`, `openssl`, `bcmath`, `exif`.

### 1.2 SSH access
hPanel → **Advanced → SSH Access** → enable. Note the **IP, port (65002) and username (u123456789)**.
Add a new SSH key (generate one on your PC with `ssh-keygen -t ed25519 -C github-deploy`) and paste the
**public** key (`.pub`) here. The **private** key goes into GitHub (step 3).

Connect once to check the CLI PHP version:
```bash
ssh -p 65002 u123456789@YOUR_IP
php -v                      # must say 8.3+ ; if not, use the full path below
ls /opt/alt/                # e.g. php83 -> /opt/alt/php83/usr/bin/php
```

### 1.3 Database
hPanel → **Databases → MySQL Databases** → create database + user for staging
(e.g. `u123456789_hfstaging`). Keep the password.

### 1.4 Staging subdomain
hPanel → **Domains → Subdomains** → create `staging.hustlefrolic.com`. Hostinger creates a folder such as
`~/domains/hustlefrolic.com/public_html/staging` (the exact path is shown after creation).
Enable SSL for it (hPanel → Security → SSL).

### 1.5 Folders, .env and the web-root symlink
```bash
mkdir -p ~/hf/staging/{releases,shared/storage,backups}
nano ~/hf/staging/shared/.env          # paste .env.example, then fill in the values below
```
Values to fill in `shared/.env` for staging:
```
APP_URL=https://staging.hustlefrolic.com
APP_KEY=                                   # run: php -r "echo 'base64:'.base64_encode(random_bytes(32)).PHP_EOL;"
SITE_NOINDEX=true                          # staging is never indexed
DB_DATABASE=u123456789_hfstaging
DB_USERNAME=u123456789_hfstaging
DB_PASSWORD=...
MAIL_PASSWORD=...                          # hello@hustlefrolic.com mailbox password
ADMIN_PASSWORD=...                         # 12+ characters, remove after the first seed
```

**Point the subdomain at Laravel's `public/` folder.** hPanel only lets you pick a folder *inside*
`public_html` for a subdomain, and not an arbitrary path, so we replace that folder with a symlink
(Hostinger's Apache follows symlinks):
```bash
SUB=~/domains/hustlefrolic.com/public_html/staging      # use the path Hostinger showed you
mv "$SUB" "$SUB.bak"                                    # keep Hostinger's default files, just in case
ln -s ~/hf/staging/current/public "$SUB"
```
(The link shows "broken" until the first deploy creates `current/`. That's expected.)

> **If symlinks are refused** (you get a 403 after deploying): use the fallback in section 6.

### 1.6 Cron (scheduler + queue)
hPanel → **Advanced → Cron Jobs** → Custom, every minute (`* * * * *`):
```
/opt/alt/php83/usr/bin/php /home/u123456789/hf/staging/current/artisan schedule:run >> /dev/null 2>&1
```
(Use `php` instead of the full path if `php -v` already shows 8.3+.) This one cron line runs the
scheduler, which starts a queue worker each minute (`queue:work --stop-when-empty`) for emails.

---

## 2. GitHub repository
1. Create a **private** repo (e.g. `hustlefrolic/website`) and push this folder to it.
2. Create two branches: `staging` (auto-deploys to staging) and `main` (production, used at launch).

## 3. GitHub secrets
Repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value |
|---|---|
| `HOSTINGER_HOST` | server IP from SSH Access |
| `HOSTINGER_PORT` | `65002` |
| `HOSTINGER_USER` | `u123456789` |
| `HOSTINGER_SSH_KEY` | the **private** key (whole file, including BEGIN/END lines) |
| `HOSTINGER_PHP_BIN` | only if needed, e.g. `/opt/alt/php83/usr/bin/php` |

## 4. Deploy
Push to the `staging` branch. GitHub Actions (`.github/workflows/deploy.yml`) will:
1. run the test suite,
2. `composer install --no-dev` and `npm run build`,
3. upload a release, link `shared/.env` + `shared/storage`,
4. **dump the database**, run `migrate --force`, `config:cache`, `route:cache`, `view:cache`,
5. switch `current` to the new release.

### First deploy only: create the admin
```bash
cd ~/hf/staging/current
php artisan db:seed --force          # creates the admin from ADMIN_* in .env + the 7 blog categories
nano ~/hf/staging/shared/.env        # delete the ADMIN_PASSWORD value now
php artisan config:cache
```
Open `https://staging.hustlefrolic.com/admin`, sign in, and scan the QR code with Google Authenticator
(or 1Password, Authy…). **Save the 8 recovery codes.** 2FA is required; there is no way into the panel without it.

## 5. Checks after each deploy
- `https://staging.hustlefrolic.com/` loads with the new layout.
- Response header `X-Robots-Tag: noindex, nofollow` is present on staging (`curl -I https://staging.hustlefrolic.com/`).
- `https://staging.hustlefrolic.com/about` redirects **301** to `/about/`.
- `/admin` asks for password, then the 6-digit code.
- `~/hf/staging/shared/storage/logs/` has no new errors.

## 6. Fallbacks
**Symlinks not allowed for the web root:** put the app inside the subdomain folder and route everything
into `public/` with an `.htaccess` at the folder root:
```apache
# ~/domains/hustlefrolic.com/public_html/staging/.htaccess
RewriteEngine On
RewriteRule ^(\.env|\.git|composer\.(json|lock)|artisan|deploy|storage|vendor|app|bootstrap|config|database|resources|routes) - [F,L,NC]
RewriteRule ^(.*)$ public/$1 [L]
```
and set `APP_DIR` in the workflow to that folder. Tell me if you need this and I'll switch the pipeline.

**`storage:link` fails:** set `FILESYSTEM_DISK=public` and add in `config/filesystems.php`
`'public' => ['root' => public_path('uploads') ...]`; uploads then live under `public/uploads`.
The deploy script prints a clear message if the link fails.

## 7. Backups
- hPanel → **Files → Backups**: daily backups are included on Business. Check it is on.
- Every deploy writes `~/hf/<env>/backups/db-<release>.sql.gz` **before** migrating.
- Roll back code: `ln -sfn ~/hf/staging/releases/<previous> ~/hf/staging/current && php artisan optimize:clear`.

## 8. Going live (Phase 6, after sign-off)
Same steps with `~/hf/production`, the `main` branch, a production database, `APP_URL=https://hustlefrolic.com`
and **`SITE_NOINDEX=false`**. The WordPress files and a full database export stay on the server for 30 days.
The detailed cutover plan comes in Phase 6.
