# IT Device Tracker

Internal Django tool for tracking IT devices.

- **Project package:** `tracker/` (settings, URLs, WSGI)
- **App:** `core/` (models, admin, views)
- **Database:** SQLite (`db.sqlite3`, not committed to git)
- **Dev dependencies:** `django`, `python-dotenv` (see `requirements.txt`)
- **Production extra:** `gunicorn` (Linux server only, installed separately)

---

## 1. Development setup

Requirements: Python 3.10+ and Git.

### Windows

Install Python (https://www.python.org/downloads/, tick "Add python.exe to PATH") and Git (`winget install --id Git.Git -e`). Open a new terminal afterward.

**PowerShell**

```powershell
git clone https://github.com/YOUR-ORG/IT-Device-Tracker.git
cd IT-Device-Tracker
python -m venv venv
venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
```

If PowerShell blocks the activate script, run this once:
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

**Command Prompt**

```bat
git clone https://github.com/YOUR-ORG/IT-Device-Tracker.git
cd IT-Device-Tracker
python -m venv venv
venv\Scripts\activate.bat
pip install -r requirements.txt
copy .env.example .env
```

### Ubuntu (development machine)

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git
git clone https://github.com/YOUR-ORG/IT-Device-Tracker.git
cd IT-Device-Tracker
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

### Configure `.env` (all platforms)

Generate a secret key:

```
python -c "from django.core.management.utils import get_random_secret_key as k; print(k())"
```

Edit `.env`:

```
DJANGO_SECRET_KEY=<paste the generated key>
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
```

### Initialize and run

With the venv activated:

```
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

Open http://127.0.0.1:8000/admin/ and log in.

> Each machine gets its own empty `db.sqlite3`. The database is not shared through git.

---

## 2. Production deployment (Ubuntu server)

This runs the app with gunicorn under systemd, with a reverse proxy (Caddy or Apache) in front. Adjust the paths, user, and hostname to suit.

### 2.1 Create a service user and get the code

```bash
sudo apt update
sudo apt install -y python3 python3-venv git
sudo useradd --system --create-home --shell /usr/sbin/nologin toolsvc
sudo mkdir -p /srv/it-device-tracker
sudo chown toolsvc:toolsvc /srv/it-device-tracker
sudo -u toolsvc git clone https://github.com/YOUR-ORG/IT-Device-Tracker.git /srv/it-device-tracker
```

For a private repo, set up a read-only deploy key or token for the `toolsvc` user, or clone as yourself and `chown -R toolsvc:toolsvc /srv/it-device-tracker` afterward.

### 2.2 Virtualenv and dependencies

gunicorn is installed here only. It is not in `requirements.txt` because it doesn't run on Windows.

```bash
cd /srv/it-device-tracker
sudo -u toolsvc python3 -m venv venv
sudo -u toolsvc venv/bin/pip install -r requirements.txt gunicorn
```

### 2.3 Production `.env`

```bash
sudo -u toolsvc cp .env.example .env
sudo -u toolsvc nano .env
sudo chmod 600 /srv/it-device-tracker/.env
```

```
DJANGO_SECRET_KEY=<a NEW key, different from development>
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=tools.example.com
```

Generate the key with `venv/bin/python -c "from django.core.management.utils import get_random_secret_key as k; print(k())"`.

If the site is served over HTTPS, `tracker/settings.py` also needs:

```python
CSRF_TRUSTED_ORIGINS = ["https://tools.example.com"]
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

(Without these you may see CSRF failures on login.)

### 2.4 Database, static files, admin user

```bash
sudo -u toolsvc venv/bin/python manage.py migrate
sudo -u toolsvc venv/bin/python manage.py collectstatic --noinput
sudo -u toolsvc venv/bin/python manage.py createsuperuser
sudo -u toolsvc venv/bin/python manage.py check --deploy
```

`toolsvc` must be able to write to `db.sqlite3` **and** the folder containing it (SQLite creates temporary journal files there). Owning `/srv/it-device-tracker` as above covers that.

### 2.5 systemd service

Create `/etc/systemd/system/it-device-tracker.service`:

```ini
[Unit]
Description=IT Device Tracker (gunicorn)
After=network.target

[Service]
User=toolsvc
Group=toolsvc
WorkingDirectory=/srv/it-device-tracker
ExecStart=/srv/it-device-tracker/venv/bin/gunicorn tracker.wsgi:application --bind 127.0.0.1:8000 --workers 3
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now it-device-tracker
sudo systemctl status it-device-tracker
```

Logs: `sudo journalctl -u it-device-tracker -n 50 --no-pager`

### 2.6 Reverse proxy

Pick one.

**Option A: Caddy** (automatic HTTPS for public domains)

```bash
sudo apt install -y caddy
```

`/etc/caddy/Caddyfile`:

```
tools.example.com {
    handle_path /static/* {
        root * /srv/it-device-tracker/staticfiles
        file_server
    }
    reverse_proxy 127.0.0.1:8000
}
```

```bash
sudo systemctl reload caddy
```

**Option B: Apache**

```bash
sudo apt install -y apache2
sudo a2enmod proxy proxy_http headers
```

`/etc/apache2/sites-available/it-device-tracker.conf`:

```apache
<VirtualHost *:80>
    ServerName tools.example.com

    Alias /static/ /srv/it-device-tracker/staticfiles/
    <Directory /srv/it-device-tracker/staticfiles>
        Require all granted
    </Directory>

    ProxyPreserveHost On
    ProxyPass /static/ !
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
    RequestHeader set X-Forwarded-Proto "http"
</VirtualHost>
```

```bash
sudo a2ensite it-device-tracker
sudo apachectl configtest && sudo systemctl reload apache2
```

If you terminate HTTPS in Apache, change the header to `"https"`. Apache must be able to read `staticfiles/` (the default permissions are fine).

### 2.7 Deploying updates

```bash
cd /srv/it-device-tracker
sudo -u toolsvc git pull
sudo -u toolsvc venv/bin/pip install -r requirements.txt
sudo -u toolsvc venv/bin/python manage.py migrate
sudo -u toolsvc venv/bin/python manage.py collectstatic --noinput
sudo systemctl restart it-device-tracker
```

### 2.8 Backups

The whole dataset is `db.sqlite3`. Use SQLite's backup command for a consistent copy:

```bash
sudo apt install -y sqlite3
sudo -u toolsvc sqlite3 /srv/it-device-tracker/db.sqlite3 ".backup '/srv/backups/db-$(date +%F).sqlite3'"
```

Schedule it with cron and copy backups off the machine.

---

## 3. Everyday development workflow

```
# After changing core/models.py
python manage.py makemigrations
python manage.py migrate

# After installing a new package
pip install <package>
pip freeze > requirements.txt     # (use Command Prompt on Windows, or see note)

# Sanity check
python manage.py check
```

Commit the generated `core/migrations/*.py` files, since they're part of the code.

> **Windows PowerShell note:** `pip freeze > requirements.txt` can write UTF-16, which breaks `pip install -r`. Use Command Prompt, or
> `pip freeze | Out-File -Encoding utf8 requirements.txt`.
>
> Don't add `gunicorn` to `requirements.txt` while developing on Windows. It's installed on the server separately.

## 4. Troubleshooting

| Symptom | Likely cause |
|---|---|
| `No changes detected` on `makemigrations` | App not in `INSTALLED_APPS`, or nothing changed since the last migration |
| Admin has no CSS in production | `collectstatic` not run, or the proxy's static path is wrong |
| `Bad Request (400)` in production | Hostname missing from `DJANGO_ALLOWED_HOSTS` |
| CSRF error on login behind HTTPS | Missing `CSRF_TRUSTED_ORIGINS` / `SECURE_PROXY_SSL_HEADER` |
| `attempt to write a readonly database` | `toolsvc` lacks write access to `db.sqlite3` or its folder |
| `KeyError: 'DJANGO_SECRET_KEY'` | `.env` missing or not in the project root |
| 502 Bad Gateway | gunicorn not running: check `systemctl status it-device-tracker` |

## 5. Files that are not in git

`.env` (secrets), `db.sqlite3` (data), `venv/`, `staticfiles/`, `__pycache__/`. Only `.env.example` is committed, as a template.
