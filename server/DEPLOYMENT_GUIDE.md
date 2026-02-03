# SentinelEdge Server - Complete Portability & Deployment Guide

## Overview
This document provides a complete guide for deploying the SentinelEdge server on any machine with automatic setup, database migrations, and production-ready configuration.

---

## 🎯 Quick Start Options

### Option 1: Production Deployment (Recommended)
```bash
python start_production.py
```
- **Auto-detects OS**: Uses Waitress (Windows) or Gunicorn (Linux)
- **Auto-setup**: Runs all 6 setup steps automatically
- **Production-ready**: Optimized for performance and stability

### Option 2: Development Server
```bash
python start_server.py
```
- **Flask dev server**: Good for testing and development
- **Auto-setup**: Same 6-step verification as production
- **Hot reload**: Easier debugging

### Option 3: Setup Only (No Server Start)
```bash
python start_server.py --no-start
```
- **Database setup**: Creates tables, procedures, schema fixes
- **No server**: Useful for CI/CD or manual server control

---

## 📦 Deployment Package

### Creating a Deployment Package
```bash
python create_deploy_zip.py
```

**Output**: `sentineledge_server_deploy_YYYYMMDD_HHMMSS.zip`

**What's Included:**
- All Python server files (*.py)
- Templates and static assets
- Migration scripts
- Configuration templates (.env.template)
- Documentation (QUICKSTART.md, README files)
- Startup scripts (start_production.py, start_server.py)

**What's Excluded (for security):**
- Actual .env file (contains secrets)
- Log files
- Database backups
- Python cache files
- Git repository

---

## 🔧 The 6-Step Auto-Setup Process

Every startup automatically runs these critical steps:

### Step 1: Database Connection Check
- Verifies PostgreSQL is accessible
- Tests DATABASE_URL from .env
- Provides troubleshooting hints if failed

### Step 2: Create Database Tables
- Runs `db.create_all()` to ensure all tables exist
- Safe to run multiple times (idempotent)

### Step 3: Install Stored Procedures
- **Critical for telemetry**: Installs 3 core procedures
  - `process_screentime_event()`
  - `process_app_switch_event()`
  - `process_domain_switch_event()`
- Uses raw DB connection to avoid SQLAlchemy parsing issues
- Drops old versions before recreation

### Step 4: Fix Schema Issues
- Adds missing columns (e.g., `idempotency_key` in `domain_sessions`)
- Creates missing indices
- Safe to run multiple times

### Step 5: Create Default Admin User
- Username: `admin`
- Password: `changeme123`
- Only creates if doesn't exist
- **⚠️ CHANGE PASSWORD IMMEDIATELY!**

### Step 6: Start Server
- Launches production server (Waitress/Gunicorn) or dev server
- Displays dashboard URL and credentials

---

## 🌐 Production Server Options

### Windows: Waitress
```python
# Automatically used by start_production.py on Windows
from waitress import serve
serve(app, host='0.0.0.0', port=5000, threads=8)
```

**Features:**
- Pure Python (no C dependencies)
- Multi-threaded
- Production-grade performance
- Auto-installed if missing

### Linux: Gunicorn
```bash
# Automatically used by start_production.py on Linux
gunicorn -c gunicorn_config.py server_main:application
```

**Features:**
- Pre-fork worker model
- Configurable workers (CPU * 2 + 1)
- Graceful restarts
- Request limits and timeouts

**Configuration**: `gunicorn_config.py`
- Workers: Auto-calculated based on CPU cores
- Timeout: 120 seconds
- Max requests: 1000 (prevents memory leaks)
- Logging: `logs/gunicorn_access.log`, `logs/gunicorn_error.log`

---

## 📋 Migration Checklist: Moving to New Machine

### Pre-Migration (Old Machine)
```bash
# Optional: Backup database
python scripts/backup_database.py
```

### New Machine Setup
1. **Install Prerequisites**
   ```bash
   # PostgreSQL 12+
   # Python 3.8+
   ```

2. **Extract Deployment Package**
   ```bash
   unzip sentineledge_server_deploy_YYYYMMDD_HHMMSS.zip
   cd sentineledge_server
   ```

3. **Configure Environment**
   ```bash
   cp .env.template .env
   # Edit .env with your DATABASE_URL and SECRET_KEY
   ```

4. **Install Dependencies**
   ```bash
   pip install -r server_requirements.txt
   ```

5. **Start Server**
   ```bash
   python start_production.py
   ```

**That's it!** The auto-setup handles everything else.

---

## 🔐 Security Checklist

### Required Actions
- [ ] Change default admin password (`admin`/`changeme123`)
- [ ] Set strong `SECRET_KEY` in .env (32+ characters)
- [ ] Configure `REGISTRATION_SECRET` for production
- [ ] Review CORS settings in .env

### Optional (Production)
- [ ] Enable HTTPS (update `gunicorn_config.py`)
- [ ] Configure firewall rules
- [ ] Set up SSL certificates
- [ ] Enable rate limiting

---

## 🚨 Troubleshooting

### Database Connection Failed
```
[1/6] Checking database connection...
  ❌ Database connection failed: could not connect to server
```

**Solutions:**
1. Verify PostgreSQL is running: `pg_isready`
2. Check DATABASE_URL in .env
3. Ensure database exists: `createdb sentineledge`
4. Test connection: `psql -U postgres -d sentineledge`

### Stored Procedures Failed
```
[3/6] Installing stored procedures...
  ❌ Procedure installation failed
```

**Solutions:**
1. Check PostgreSQL version (12+ required)
2. Verify user has CREATE FUNCTION permission
3. Run manually: `python start_server.py --no-start`

### Port Already in Use
```
OSError: [Errno 98] Address already in use
```

**Solutions:**
1. Change port in .env: `SERVER_PORT=5001`
2. Kill existing process: `lsof -ti:5000 | xargs kill -9` (Linux)
3. Or: `netstat -ano | findstr :5000` (Windows)

### Waitress/Gunicorn Not Found
```
ModuleNotFoundError: No module named 'waitress'
```

**Solutions:**
- `start_production.py` auto-installs Waitress
- Manual: `pip install waitress` (Windows) or `pip install gunicorn` (Linux)

---

## 📁 File Structure

```
server_v3/server/
├── start_production.py       # Production startup (auto-detects OS)
├── start_server.py            # Development startup + auto-setup
├── create_deploy_zip.py       # Deployment package creator
├── QUICKSTART.md              # Quick reference guide
├── .env.template              # Environment configuration template
├── server_requirements.txt    # Python dependencies
├── gunicorn_config.py         # Gunicorn production config
├── server_main.py             # Main Flask application
├── server_app.py              # App factory
├── templates/                 # Dashboard HTML templates
├── static/                    # CSS, JS, images
├── migrations/                # Database migrations
├── scripts/                   # Utility scripts
│   ├── backup_database.py
│   ├── restore_database.py
│   └── fixes/                 # Schema fix scripts
└── logs/                      # Server logs (created on first run)
```

---

## 🎓 Advanced Usage

### Running as a Service (Linux)

**systemd service file** (`/etc/systemd/system/sentineledge.service`):
```ini
[Unit]
Description=SentinelEdge Server
After=network.target postgresql.service

[Service]
Type=simple
User=sentineledge
WorkingDirectory=/opt/sentineledge/server
Environment="PATH=/opt/sentineledge/venv/bin"
ExecStart=/opt/sentineledge/venv/bin/python start_production.py
Restart=always

[Install]
WantedBy=multi-user.target
```

**Commands:**
```bash
sudo systemctl enable sentineledge
sudo systemctl start sentineledge
sudo systemctl status sentineledge
```

### Custom Port Configuration

**Option 1: Environment Variable**
```bash
# In .env
SERVER_PORT=8080
```

**Option 2: Gunicorn Config**
```python
# In gunicorn_config.py
bind = "0.0.0.0:8080"
```

### Database Backup Strategy

**Automated daily backups:**
```bash
# Add to crontab
0 2 * * * cd /opt/sentineledge/server && python scripts/backup_database.py
```

---

## 📊 Performance Tuning

### Gunicorn Workers
```python
# gunicorn_config.py
workers = multiprocessing.cpu_count() * 2 + 1  # Default
workers = 4  # Fixed number for predictable performance
```

### Database Connection Pool
```python
# server_config.py
SQLALCHEMY_POOL_SIZE = 10
SQLALCHEMY_MAX_OVERFLOW = 20
```

### Waitress Threads
```python
# start_production.py
serve(app, host='0.0.0.0', port=5000, threads=16)  # Increase for high load
```

---

## 🔄 Update Procedure

1. **Backup current installation**
   ```bash
   python scripts/backup_database.py
   cp -r /opt/sentineledge /opt/sentineledge.backup
   ```

2. **Deploy new version**
   ```bash
   unzip sentineledge_server_deploy_NEW.zip
   cp sentineledge_server/* /opt/sentineledge/server/
   ```

3. **Restart server**
   ```bash
   # Auto-setup runs on restart
   sudo systemctl restart sentineledge
   ```

---

## 📞 Support

- **Documentation**: See `QUICKSTART.md` for quick reference
- **Logs**: Check `logs/gunicorn_error.log` or Flask console output
- **Database**: Use `psql` to inspect tables and procedures

---

## ✅ Success Indicators

After running `python start_production.py`, you should see:

```
======================================================================
  SentinelEdge Server - Smart Startup
  Automatic Setup & Configuration
======================================================================

[1/6] Checking database connection...
  ✅ Database connected
[2/6] Creating/verifying database tables...
  ✅ All tables created/verified
[3/6] Installing stored procedures...
  - Dropping legacy functions...
  - Creating process_screentime_event...
  - Creating process_app_switch_event...
  - Creating process_domain_switch_event...
  ✅ Stored procedures installed
[4/6] Fixing schema (adding missing columns)...
  ✅ Schema fixed
[5/6] Checking admin user...
  ✅ Admin user exists
[6/6] Starting server...

============================================================
✅ AUTO-SETUP COMPLETE!
============================================================

🚀 Server starting...
   Dashboard: http://localhost:5000/dashboard
   Login: admin / changeme123

⚠️  Press Ctrl+C to stop

✅ OS: Windows detected. Using Waitress for production.
Serving on http://0.0.0.0:5000
```

**Test the server:**
```bash
curl http://localhost:5000/heartbeat
# Should return: {"error":"Missing authentication..."}
# (This confirms the server is running and auth is active)
```

---

**Last Updated**: 2026-01-29  
**Version**: 3.0.1
