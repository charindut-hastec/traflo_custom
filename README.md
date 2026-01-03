# Traflo Custom

Custom Frappe/ERPNext application for Traflo - A multi-tenant B2B SaaS platform for wholesale trading and distribution.

This app extends ERPNext with Traflo-specific features including custom workflows, vendor portals, inventory management, and multi-tenant capabilities.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Development Setup](#development-setup)
- [Running the Application](#running-the-application)
- [API Documentation](#api-documentation)
- [Development Workflow](#development-workflow)
- [Contributing](#contributing)
- [License](#license)

---

## Prerequisites

Before installing this app, ensure you have the following installed:

### System Requirements
- **Python**: 3.10 or 3.11
- **Node.js**: 18+ or 20+
- **MariaDB**: 10.6+ (or PostgreSQL)
- **Redis**: Latest stable
- **Git**: Latest stable
- **Frappe Bench**: Latest stable

### Install Prerequisites

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install -y python3-dev python3-pip python3-venv \
    redis-server mariadb-server mariadb-client \
    git curl build-essential libffi-dev libssl-dev \
    libmysqlclient-dev wkhtmltopdf

# Install Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install Yarn
sudo npm install -g yarn
```

#### macOS
```bash
brew install python@3.11 git redis mariadb node@18 pkg-config mariadb-connector-c
brew install --cask wkhtmltopdf
```

### Configure MariaDB

```bash
# Secure installation
sudo mysql_secure_installation

# Configure character set
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add under `[mysqld]`:
```ini
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

```bash
# Restart MariaDB
sudo systemctl restart mariadb
```

### Install Frappe Bench

**Official Documentation**: [Bench Installation](https://docs.frappe.io/framework/user/en/installation)

```bash
# Install bench via pip
sudo pip3 install frappe-bench

# Verify installation
bench --version
```

---

## Installation

### Step 1: Initialize Frappe Bench

**Official Command Reference**: [bench init](https://docs.frappe.io/framework/user/en/bench/bench-commands#bench-init)

```bash
# Navigate to your projects directory
cd ~/projects

# Initialize bench with Frappe v15
bench init traflo-backend --frappe-branch version-15
cd traflo-backend
```

### Step 2: Install ERPNext

**Official Command Reference**: [bench get-app](https://docs.frappe.io/framework/user/en/bench/bench-commands#bench-get-app)

```bash
# Get ERPNext v15
bench get-app erpnext --branch version-15
```

### Step 3: Install Traflo Custom App

```bash
# Get this app from GitHub
bench get-app https://github.com/YOUR-USERNAME/traflo_custom.git --branch develop

# Or if you have SSH configured:
# bench get-app git@github.com:YOUR-USERNAME/traflo_custom.git --branch develop
```

### Step 4: Create a New Site

**Official Command Reference**: [bench new-site](https://docs.frappe.io/framework/user/en/bench/bench-commands#bench-new-site)

```bash
# Create development site
bench new-site dev.local

# You'll be prompted for:
# 1. MySQL root password
# 2. Administrator password (remember this for login)
```

**Add to /etc/hosts** (for local development):
```bash
echo "127.0.0.1 dev.local" | sudo tee -a /etc/hosts
```

### Step 5: Install Apps on Site

**Official Command Reference**: [bench install-app](https://docs.frappe.io/framework/user/en/bench/bench-commands#bench-install-app)

```bash
# Install ERPNext first
bench --site dev.local install-app erpnext

# Then install Traflo Custom
bench --site dev.local install-app traflo_custom
```

---

## Development Setup

### Start Development Server

```bash
cd ~/projects/traflo-backend

# Start all services
bench start
```

This starts:
- **Web Server** (port 8000) - Main application
- **SocketIO** (port 9000) - Real-time updates
- **Redis Cache** - Caching layer
- **Redis Queue** - Background jobs
- **Scheduler** - Scheduled tasks
- **Worker** - Background job processor
- **Watch** - Frontend asset builder

### Access the Application

- **URL**: http://dev.local:8000
- **Username**: Administrator
- **Password**: [password you set during site creation]

---

## Running the Application

### Production Mode

```bash
# Setup production with nginx and supervisor
sudo bench setup production your-user

# Enable scheduler
bench --site dev.local enable-scheduler
```

### Multi-Site Setup

Frappe supports multi-tenancy out of the box. Each site has its own database.

**Official Documentation**: [Multi-tenancy in Frappe](https://frappeframework.com/docs/user/en/tutorial/install-and-setup-bench)

```bash
# Create additional tenant sites
bench new-site tenant1.traflo.com
bench --site tenant1.traflo.com install-app erpnext
bench --site tenant1.traflo.com install-app traflo_custom

# Each site gets:
# - Separate database
# - Separate configuration
# - Shared codebase
```

---

## API Documentation

### Generate API Keys

For frontend integration or API access:

1. Login to ERPNext at http://dev.local:8000
2. Go to **User Menu** → **My Settings**
3. Scroll to **API Access** section
4. Click **Generate Keys**
5. Copy **API Key** and **API Secret**

### API Endpoints

All custom APIs are in `traflo_custom/api/`:

```
traflo_custom/
└── api/
    ├── dashboard.py      # Dashboard APIs
    ├── vendor.py         # Vendor portal APIs
    ├── inventory.py      # Inventory APIs
    └── workflow.py       # Workflow APIs
```

### Example API Call

```bash
# Test API endpoint
curl -X GET http://dev.local:8000/api/method/traflo_custom.api.dashboard.hello_traflo \
  -H "Authorization: token <api_key>:<api_secret>"
```

---

## Development Workflow

### Directory Structure

```
apps/traflo_custom/
├── traflo_custom/
│   ├── api/              # Custom REST APIs
│   ├── overrides/        # ERPNext overrides
│   ├── doctype/          # Custom DocTypes (tables)
│   ├── fixtures/         # Custom fields & workflows
│   ├── public/           # Static assets (JS, CSS)
│   ├── templates/        # Jinja templates
│   ├── www/              # Web pages
│   └── hooks.py          # Main configuration
├── .gitignore
├── pyproject.toml        # Python dependencies
├── package.json          # Node dependencies
└── README.md
```

### Making Code Changes

1. **Edit files** in `apps/traflo_custom/traflo_custom/`

2. **Clear cache** after changes:
   ```bash
   bench --site dev.local clear-cache
   ```

3. **Run migrations** if you added DocTypes:
   ```bash
   bench --site dev.local migrate
   ```

4. **Restart** if needed:
   ```bash
   bench restart
   ```

### Creating Custom DocTypes

```bash
# Via UI: http://dev.local:8000/app/doctype/new

# Or via bench:
bench --site dev.local console
```

### Adding Custom APIs

Create file in `traflo_custom/api/`:

```python
# traflo_custom/api/example.py
import frappe

@frappe.whitelist()
def my_custom_api():
    """Custom API endpoint"""
    return {
        "message": "Hello from Traflo!",
        "status": "success"
    }
```

Access at: `http://dev.local:8000/api/method/traflo_custom.api.example.my_custom_api`

### Git Workflow

```bash
cd apps/traflo_custom

# Create feature branch
git checkout -b feature/my-feature

# Make changes
git add .
git commit -m "Add new feature"

# Push to remote
git push origin feature/my-feature

# Create pull request on GitHub
```

### Updating Frappe/ERPNext

**Official Command**: `bench update`

```bash
# Update all apps
bench update

# This updates frappe and erpnext but NOT your custom app
# Your custom app must be updated via git pull
```

---

## Contributing

This app uses `pre-commit` for code formatting and linting.

### Install Pre-commit

**Official Documentation**: [pre-commit.com](https://pre-commit.com)

```bash
# Install pre-commit
pip install pre-commit

# Enable for this repository
cd apps/traflo_custom
pre-commit install
```

### Code Quality Tools

Pre-commit is configured to use:
- **ruff** - Python linter and formatter
- **eslint** - JavaScript linter
- **prettier** - Code formatter
- **pyupgrade** - Python syntax upgrader

### Running Tests

```bash
# Run all tests
bench --site dev.local run-tests --app traflo_custom

# Run specific test
bench --site dev.local run-tests --app traflo_custom --module traflo_custom.tests.test_api
```

### Code Style

- Follow [Frappe Framework guidelines](https://github.com/frappe/frappe/wiki/Contribution-Guidelines)
- Use descriptive variable names
- Add docstrings to functions
- Keep functions small and focused

---

## CI/CD

This app uses GitHub Actions for continuous integration.

### Workflows

- **CI**: Runs tests on every push to `develop`
- **Linters**: Runs [Frappe Semgrep Rules](https://github.com/frappe/semgrep-rules) and [pip-audit](https://pypi.org/project/pip-audit/) on PRs

### Workflow Files

See `.github/workflows/` for configuration.

---

## Troubleshooting

### Port conflicts

If you see "Address already in use" errors:

```bash
# Check what's using the port
sudo lsof -i :8000

# Or change bench ports
bench set-config -g http_port 8001
```

### Database connection errors

```bash
# Check MariaDB is running
sudo systemctl status mariadb

# Restart MariaDB
sudo systemctl restart mariadb
```

### Clear cache and rebuild

```bash
# Clear all caches
bench --site dev.local clear-cache
bench --site dev.local clear-website-cache

# Rebuild assets
bench build

# Restart
bench restart
```

### Reset everything

```bash
# Drop and recreate site (WARNING: Deletes all data!)
bench drop-site dev.local --force
bench new-site dev.local
bench --site dev.local install-app erpnext
bench --site dev.local install-app traflo_custom
```

---

## Documentation Links

### Official Frappe/ERPNext Documentation
- [Frappe Framework Docs](https://docs.frappe.io)
- [ERPNext Documentation](https://docs.erpnext.com)
- [Bench Commands](https://docs.frappe.io/framework/user/en/bench/bench-commands)
- [Frappe App Development](https://frappeframework.com/docs/user/en/tutorial)

### Traflo Project Documentation
- **SDS**: See `TRAFLO_SDS.md` in project root
- **Analysis**: See `TRAFLO_ANALYSIS.md` in project root
- **Setup Guide**: See `SETUP_INSTRUCTIONS.md` in project root

---

## Quick Reference

```bash
# Start development
bench start

# Clear cache
bench --site dev.local clear-cache

# Run migrations
bench --site dev.local migrate

# Console
bench --site dev.local console

# Run tests
bench --site dev.local run-tests --app traflo_custom

# Update apps
bench update

# Backup
bench --site dev.local backup

# Restore
bench --site dev.local restore /path/to/backup
```

---

## License

AGPL-3.0

---

## Support

For issues and questions:
- **Frappe Forum**: https://discuss.frappe.io
- **Project Issues**: [GitHub Issues](https://github.com/YOUR-USERNAME/traflo_custom/issues)

---

## Summary: Quick Setup for New Developers

```bash
# 1. Install bench
sudo pip3 install frappe-bench

# 2. Initialize bench
bench init traflo-backend --frappe-branch version-15
cd traflo-backend

# 3. Get ERPNext
bench get-app erpnext --branch version-15

# 4. Get Traflo Custom (this app)
bench get-app https://github.com/YOUR-USERNAME/traflo_custom.git --branch develop

# 5. Create site
bench new-site dev.local
echo "127.0.0.1 dev.local" | sudo tee -a /etc/hosts

# 6. Install apps
bench --site dev.local install-app erpnext
bench --site dev.local install-app traflo_custom

# 7. Start development server
bench start

# 8. Visit http://dev.local:8000
# Login: Administrator / [your password]
```

**Happy Coding! 🚀**
