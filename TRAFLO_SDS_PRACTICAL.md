# Traflo Software Design Specification (SDS)
**Build Guide for Software Architects & Engineers**

Version: 2.0
Date: 2026-01-03
Status: **BUILD-READY**

---

## 📋 Quick Navigation

- [0. DEVELOPER SETUP GUIDE (START HERE)](#0-developer-setup-guide-start-here) ⭐
- [1. What Exists vs What to Build](#1-what-exists-vs-what-to-build)
- [2. Repository Architecture](#2-repository-architecture)
- [3. SRS Requirement Mapping](#3-srs-requirement-mapping)
- [4. Technology Stack (Current)](#4-technology-stack-current)
- [5. Implementation Guide](#5-implementation-guide)
- [6. File Structure & Shortcuts](#6-file-structure--shortcuts)

---

## 0. DEVELOPER SETUP GUIDE (START HERE)

### 🎯 For New Developers - Complete Setup from Scratch

#### Step 1: Install Prerequisites (Ubuntu/macOS)

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y \
    python3.11 python3.11-dev python3.11-venv \
    mariadb-server redis-server \
    git curl nodejs npm \
    wkhtmltopdf libmysqlclient-dev

# Install Yarn
sudo npm install -g yarn

# Verify installations
python3.11 --version  # Should be 3.11+
node --version        # Should be 18+
redis-cli ping        # Should return PONG
```

#### Step 2: Configure MariaDB

```bash
# Secure MariaDB
sudo mysql_secure_installation

# Configure for ERPNext
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf

# Add these lines under [mysqld]:
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
max_allowed_packet = 256M

# Restart MariaDB
sudo systemctl restart mariadb
```

#### Step 3: Install Frappe Bench

```bash
# Install bench CLI
sudo pip3 install frappe-bench

# Add to PATH
echo 'export PATH=$PATH:~/.local/bin' >> ~/.bashrc
source ~/.bashrc

# Verify
bench --version
```

#### Step 4: Initialize Bench & Install ERPNext

```bash
# Navigate to your workspace
cd /home/ctr/CodeBases/

# Initialize bench (creates traflo-backend/)
bench init traflo-backend --frappe-branch version-15
# This takes 5-10 minutes...

cd traflo-backend

# Get ERPNext app
bench get-app erpnext --branch version-15
# This takes 5-10 minutes...

# Create your first development site
bench new-site dev.local
# Enter MySQL root password when prompted
# Set Administrator password (save this!)

# Install ERPNext on the site
bench --site dev.local install-app erpnext
# This takes 5-10 minutes...

# ✅ Basic setup complete!
```

#### Step 5: Create & Install Your Custom Frappe App

**Option A: Create New App (First Time)**

```bash
cd /home/ctr/CodeBases/traflo-backend

# Create custom app
bench new-app traflo_custom

# Answer prompts:
# App Title: Traflo Custom
# App Description: Traflo SaaS customizations for ERPNext
# App Publisher: Your Company
# App Email: dev@traflo.com
# App License: Proprietary

# This creates: apps/traflo_custom/

# Install app on your site
bench --site dev.local install-app traflo_custom

# Verify
bench --site dev.local list-apps
# Should show: frappe, erpnext, traflo_custom
```

**Option B: Clone Existing App (Team Member)**

```bash
cd /home/ctr/CodeBases/traflo-backend/apps

# Clone your company's custom app repo
git clone git@github.com:yourcompany/traflo-erpnext-custom.git traflo_custom

# Install dependencies
cd traflo_custom
pip3 install -r requirements.txt

# Install app on site
cd ../..
bench --site dev.local install-app traflo_custom

# Run migrations to apply custom fields/workflows
bench --site dev.local migrate
```

#### Step 6: Set Up Git (ONLY for Custom App)

**⚠️ WARNING: DO NOT run `git init` in `/home/ctr/CodeBases/traflo-backend/`**

**❌ WRONG:**
```bash
cd /home/ctr/CodeBases/traflo-backend
git init                                    # ❌ NO! Don't do this!
```

**✅ CORRECT:**
```bash
cd /home/ctr/CodeBases/traflo-backend/apps/traflo_custom
git init                                    # ✅ YES! Only here!
```

**Why?**
- `traflo-backend/` contains frappe + erpnext (not your code)
- Each developer generates the bench locally
- You ONLY version control YOUR custom app

**Setup Git:**

```bash
cd /home/ctr/CodeBases/traflo-backend/apps/traflo_custom

# Check if .git exists (created by bench new-app)
ls -la .git

# If exists, configure remote:
git remote add origin git@github.com:yourcompany/traflo-erpnext-custom.git

# If doesn't exist, initialize:
git init
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:yourcompany/traflo-erpnext-custom.git
git branch -M main
git push -u origin main

# Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
*.egg-info/
.DS_Store
*.swp
*~
EOF

git add .gitignore
git commit -m "Add .gitignore"
git push
```

#### Step 7: Configure Frontend

```bash
cd /home/ctr/CodeBases/Traflo/Traflo-main

# Install dependencies
pnpm install

# Create environment file
cp .env.example .env.local

# Edit .env.local
nano .env.local

# Add these values:
NEXT_PUBLIC_ERPNEXT_URL=http://localhost:8000
ERPNEXT_API_KEY=your_api_key_here
ERPNEXT_API_SECRET=your_api_secret_here

# Generate API keys (next step)
```

#### Step 8: Generate API Keys

```bash
cd /home/ctr/CodeBases/traflo-backend

# Start bench console
bench --site dev.local console

# In the console, run:
from frappe.utils import random_string

# Generate keys
api_key = random_string(15)
api_secret = random_string(32)

# Print them
print(f"API_KEY: {api_key}")
print(f"API_SECRET: {api_secret}")

# Save to Administrator user
frappe.db.set_value("User", "Administrator", "api_key", api_key)
frappe.db.set_value("User", "Administrator", "api_secret", api_secret)
frappe.db.commit()

# Exit console
exit()

# Copy the keys to your .env.local file
```

#### Step 9: Start Development Servers

```bash
# Terminal 1: Backend
cd /home/ctr/CodeBases/traflo-backend
bench start

# Terminal 2: Frontend
cd /home/ctr/CodeBases/Traflo/Traflo-main
pnpm dev

# ✅ Access:
# ERPNext: http://localhost:8000
# Frontend: http://localhost:3000
# Login: Administrator / [your password]
```

#### Step 10: Verify Setup

**Test Backend:**
```bash
# Test ERPNext API
curl -X GET "http://localhost:8000/api/method/frappe.auth.get_logged_user" \
  -H "Authorization: token YOUR_API_KEY:YOUR_API_SECRET"

# Should return: {"message":"Administrator"}
```

**Test Frontend:**
```bash
# Open http://localhost:3000
# Check browser console - should connect to ERPNext
```

---

### 🔄 Daily Development Workflow

#### Morning Setup

```bash
# Terminal 1: Start backend
cd /home/ctr/CodeBases/traflo-backend
bench start

# Terminal 2: Start frontend
cd /home/ctr/CodeBases/Traflo/Traflo-main
pnpm dev
```

#### Making Changes to Custom App

```bash
cd /home/ctr/CodeBases/traflo-backend/apps/traflo_custom

# Create feature branch
git checkout -b feature/add-discount-approval

# Make your changes...
# Example: Add new API file
mkdir -p traflo_custom/api
touch traflo_custom/api/approvals.py

# Edit the file (see implementation examples below)

# Restart bench to load changes
cd ../../
bench restart

# Test your changes
curl http://localhost:8000/api/method/traflo_custom.api.approvals.get_pending

# If you added custom fields via UI, EXPORT FIXTURES:
bench --site dev.local export-fixtures

# Commit changes
cd apps/traflo_custom
git add .
git commit -m "Add discount approval API"
git push origin feature/add-discount-approval

# Create Pull Request on GitHub
```

#### Making Changes to Frontend

```bash
cd /home/ctr/CodeBases/Traflo/Traflo-main

# Create feature branch
git checkout -b feature/add-approval-ui

# Make changes (Next.js auto-reloads)
# Example: Edit dashboard
nano app/(main)/page.tsx

# Test in browser: http://localhost:3000

# Commit
git add .
git commit -m "Add approval UI"
git push origin feature/add-approval-ui
```

---

### 📦 CRITICAL: Git Strategy - What Goes to GitHub vs What Stays Local

**⚠️ MOST IMPORTANT CONCEPT:**

```
┌─────────────────────────────────────────────────────────────┐
│  YOUR GIT REPOSITORIES (Push to GitHub)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Frontend                                                 │
│     📁 /home/ctr/CodeBases/Traflo/Traflo-main/              │
│     🔗 git@github.com:yourcompany/traflo-frontend.git      │
│     ✅ git init HERE                                        │
│     ✅ git push HERE                                        │
│                                                              │
│  2. Custom App ONLY                                          │
│     📁 /home/ctr/CodeBases/traflo-backend/apps/traflo_custom/│
│     🔗 git@github.com:yourcompany/traflo-custom.git        │
│     ✅ git init HERE                                        │
│     ✅ git push HERE                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  GENERATED LOCALLY (DO NOT PUSH TO GIT)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  traflo-backend/                    ❌ NO .git here!        │
│  ├── apps/                                                   │
│  │   ├── frappe/                    ⚙️ Generated by bench   │
│  │   ├── erpnext/                   ⚙️ Generated by bench   │
│  │   └── traflo_custom/             ✅ YOUR Git repo        │
│  ├── sites/                         ⚙️ Generated by bench   │
│  ├── env/                           ⚙️ Generated by bench   │
│  ├── config/                        ⚙️ Generated by bench   │
│  ├── logs/                          ⚙️ Generated by bench   │
│  └── Procfile                       ⚙️ Generated by bench   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**The Process:**

```
Developer Machine 1:
┌──────────────────────────────────────────┐
│ 1. bench init traflo-backend            │ ← Creates bench structure
│ 2. bench get-app erpnext                │ ← Pulls from frappe/erpnext
│ 3. bench new-app traflo_custom          │ ← Creates custom app
│ 4. cd apps/traflo_custom                │
│ 5. git init                             │ ← Git ONLY this folder
│ 6. git push to GitHub                   │
└──────────────────────────────────────────┘
                    │
                    │ Push traflo_custom to GitHub
                    ▼
            ┌────────────────┐
            │    GitHub      │
            │  traflo_custom │
            └────────────────┘
                    │
                    │ Clone traflo_custom only
                    ▼
Developer Machine 2:
┌──────────────────────────────────────────┐
│ 1. bench init traflo-backend            │ ← Creates bench structure
│ 2. bench get-app erpnext                │ ← Pulls from frappe/erpnext
│ 3. cd apps/                             │
│ 4. git clone traflo_custom              │ ← Clone YOUR app only
│ 5. bench install-app traflo_custom      │
└──────────────────────────────────────────┘
```

#### Setup Git for Custom App Only

```bash
# Navigate to custom app directory
cd /home/ctr/CodeBases/traflo-backend/apps/traflo_custom

# This directory should have its own .git/
ls -la .git

# If .git exists (created by bench new-app):
✅ Already initialized!

# If not, initialize:
git init
git add .
git commit -m "Initial commit"

# Add remote
git remote add origin git@github.com:yourcompany/traflo-erpnext-custom.git

# Push
git push -u origin main
```

#### What to Commit vs What NOT to Commit

**✅ DO COMMIT (in traflo_custom/):**
```
apps/traflo_custom/
├── .git/                              ✅ Git repo
├── .gitignore                         ✅ Ignore rules
├── setup.py                           ✅ App metadata
├── requirements.txt                   ✅ Dependencies
├── README.md                          ✅ Documentation
├── traflo_custom/
│   ├── __init__.py                    ✅ Package init
│   ├── hooks.py                       ✅ CRITICAL - Main config
│   ├── modules.txt                    ✅ Module list
│   ├── patches.txt                    ✅ DB patches
│   ├── api/                           ✅ Your APIs
│   ├── doctype/                       ✅ Custom DocTypes
│   ├── overrides/                     ✅ ERPNext overrides
│   ├── fixtures/                      ✅ CRITICAL - Custom fields
│   ├── config/                        ✅ UI configs
│   ├── public/                        ✅ Static files
│   ├── templates/                     ✅ Email templates
│   └── tests/                         ✅ Unit tests
```

**❌ DON'T COMMIT (Generated by Bench):**
```
traflo-backend/                        ❌ DON'T git init here!
├── apps/
│   ├── frappe/                        ❌ bench get-app frappe (upstream)
│   ├── erpnext/                       ❌ bench get-app erpnext (upstream)
│   └── traflo_custom/                 ✅ ONLY THIS has .git/
├── sites/                             ❌ bench new-site creates this
├── env/                               ❌ bench init creates this
├── logs/                              ❌ Runtime generated
├── node_modules/                      ❌ yarn install creates this
├── config/                            ❌ bench init creates this
└── Procfile                           ❌ bench init creates this
```

**Why NOT to version control the bench:**
1. `frappe/` and `erpnext/` are pulled from GitHub (frappe/frappe, frappe/erpnext)
2. Each developer generates bench locally with `bench init`
3. `sites/` contains database credentials (security risk)
4. `env/` is auto-generated Python virtual environment
5. Would create a massive repo with code you don't own

**What each developer does:**
```bash
# Each developer runs these commands locally:
bench init traflo-backend              # Generates bench structure
bench get-app frappe                   # Pulls from github.com/frappe/frappe
bench get-app erpnext                  # Pulls from github.com/frappe/erpnext
cd apps/
git clone YOUR_CUSTOM_APP              # Clone YOUR app only
bench install-app traflo_custom        # Installs your app
```

#### .gitignore for Custom App

```bash
cd /home/ctr/CodeBases/traflo-backend/apps/traflo_custom

cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*.egg-info/
dist/
build/

# IDE
.vscode/
.idea/
*.swp
*~

# OS
.DS_Store
Thumbs.db

# Logs
*.log
EOF

git add .gitignore
git commit -m "Add .gitignore"
```

---

### 🚀 Installing Custom App on New Server/Developer

**Scenario: New developer joins team**

```bash
# On new developer's machine:

# 1. Set up bench (steps 1-4 above)
bench init traflo-backend --frappe-branch version-15
cd traflo-backend
bench get-app erpnext --branch version-15
bench new-site dev-newdev.local
bench --site dev-newdev.local install-app erpnext

# 2. Clone custom app (instead of creating new)
cd apps/
git clone git@github.com:yourcompany/traflo-erpnext-custom.git traflo_custom

# 3. Install dependencies
cd traflo_custom
pip3 install -r requirements.txt

# 4. Install app on site
cd ../..
bench --site dev-newdev.local install-app traflo_custom

# 5. Migrate (applies all custom fields from fixtures/)
bench --site dev-newdev.local migrate

# 6. Import demo data (optional)
bench --site dev-newdev.local import-csv path/to/demo-data.csv

# 7. Start development
bench start
```

**What happens during `bench --site dev.local install-app traflo_custom`:**
1. ✅ Installs Python dependencies from `requirements.txt`
2. ✅ Runs `before_install` hooks (if defined in `hooks.py`)
3. ✅ Creates custom DocTypes (from `doctype/` folder)
4. ✅ Imports fixtures (custom fields, workflows) from `fixtures/` folder
5. ✅ Runs `after_install` hooks (if defined)
6. ✅ Builds assets (JS, CSS) from `public/` folder

---

### 🔧 Working with Custom Fields & Fixtures

**CRITICAL: Always export fixtures after adding custom fields!**

#### Adding Custom Fields

**Method 1: Via ERPNext UI (Recommended)**

```bash
# 1. Start bench
bench start

# 2. Go to http://localhost:8000
# 3. Login as Administrator
# 4. Search "Customize Form"
# 5. Select DocType: "Sales Order"
# 6. Click "Add Row" under Custom Fields
# 7. Fill:
#    - Label: "Discount Approval Required"
#    - Type: Check
#    - Insert After: discount_amount
# 8. Click "Update"

# 9. IMMEDIATELY export fixtures:
cd /home/ctr/CodeBases/traflo-backend
bench --site dev.local export-fixtures

# This updates: apps/traflo_custom/traflo_custom/fixtures/custom_field.json

# 10. Commit the fixture:
cd apps/traflo_custom
git add traflo_custom/fixtures/custom_field.json
git commit -m "Add discount_approval_required field to Sales Order"
git push
```

**Method 2: Via Code (Advanced)**

```python
# Create file: apps/traflo_custom/traflo_custom/fixtures/custom_fields.py

def get_custom_fields():
    """Return list of custom fields to install"""
    return {
        "Sales Order": [
            {
                "fieldname": "discount_approval_required",
                "label": "Discount Approval Required",
                "fieldtype": "Check",
                "insert_after": "discount_amount",
                "default": 0
            },
            {
                "fieldname": "approver_notes",
                "label": "Approver Notes",
                "fieldtype": "Text",
                "insert_after": "discount_approval_required"
            }
        ]
    }

# Then in hooks.py, add:
# fixtures = [..., "apps/traflo_custom/traflo_custom/fixtures/custom_fields.py"]
```

#### Exporting Fixtures

```bash
# Export all fixtures (defined in hooks.py)
bench --site dev.local export-fixtures

# What gets exported:
# - Custom fields → fixtures/custom_field.json
# - Property setters → fixtures/property_setter.json
# - Workflows → fixtures/workflow.json
# - Custom DocTypes → fixtures/[doctype_name].json

# Always commit fixtures after export:
cd apps/traflo_custom
git add traflo_custom/fixtures/
git commit -m "Update fixtures"
git push
```

#### Importing Fixtures (Automatic)

```bash
# Fixtures are automatically imported when:

# 1. Installing app
bench --site dev.local install-app traflo_custom

# 2. Running migrate
bench --site dev.local migrate

# No manual import needed!
```

---

### 🐛 Common Setup Issues

#### Issue 1: "bench: command not found"

```bash
# Fix: Add to PATH
echo 'export PATH=$PATH:~/.local/bin' >> ~/.bashrc
source ~/.bashrc
```

#### Issue 2: "Access denied for user 'root'@'localhost'"

```bash
# Fix: Reset MariaDB root password
sudo mysql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'your_new_password';
FLUSH PRIVILEGES;
EXIT;
```

#### Issue 3: "Redis connection refused"

```bash
# Fix: Start Redis
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

#### Issue 4: Custom app not showing in bench

```bash
# Check if app is in apps.txt
cat sites/dev.local/apps.txt

# If not listed, manually add:
echo "traflo_custom" >> sites/dev.local/apps.txt

# Then migrate:
bench --site dev.local migrate
```

#### Issue 5: Fixtures not importing

```bash
# Check fixtures are in hooks.py:
cat apps/traflo_custom/traflo_custom/hooks.py | grep fixtures

# Force re-import:
bench --site dev.local migrate --force

# Clear cache and retry:
bench --site dev.local clear-cache
bench --site dev.local migrate
```

#### Issue 6: API endpoint not accessible

```bash
# Check if method is whitelisted:
# File should have: @frappe.whitelist()

# Restart bench:
bench restart

# Check logs:
tail -f logs/web.error.log
```

---

### ✅ Setup Verification Checklist

Before starting development, verify:

- [ ] Bench initialized: `bench --version` works
- [ ] ERPNext running: http://localhost:8000 accessible
- [ ] Custom app created: `apps/traflo_custom/` exists
- [ ] Custom app installed: `bench --site dev.local list-apps` shows it
- [ ] Git initialized: `apps/traflo_custom/.git/` exists
- [ ] Remote configured: `git remote -v` shows your repo
- [ ] Frontend running: http://localhost:3000 accessible
- [ ] API keys generated: `.env.local` has keys
- [ ] API connection works: Frontend can call backend

**Test Command:**
```bash
# Should return success
curl http://localhost:8000/api/method/ping
```

---

## 1. What Exists vs What to Build

### ✅ What We Have (Evidence-Based)

**Frontend** (`/home/ctr/CodeBases/Traflo/Traflo-main/`)
```
✅ Next.js 16 + React 19 + TypeScript
✅ ERPNext API Client → lib/erpnext-client.ts (268 lines)
✅ UI Components (shadcn/ui)
✅ Route structure (agent, main, portal)
✅ Mock data → lib/mock-data.ts
✅ Basic hooks → lib/hooks/use-erpnext.ts
```

**Backend** (`/home/ctr/CodeBases/traflo-backend/`)
```
✅ Frappe Framework 15.x installed
✅ ERPNext 15.x installed
✅ Custom app skeleton → apps/traflo_custom/ (EMPTY - needs implementation)
✅ MariaDB ready
✅ Redis ready
```

### ❌ What to Build

**Frontend Modules:**
```
❌ Dashboard with real data
❌ Sales lifecycle UI (Quotation → Order → Delivery → Invoice)
❌ Purchase lifecycle UI
❌ Inventory management UI
❌ Customer/Vendor portals
❌ Approval workflows UI
❌ Audit log viewer
❌ Setup wizard
❌ Authentication (magic link + password)
```

**Backend Customizations:**
```
❌ Custom APIs (dashboard KPIs, approvals)
❌ Custom DocTypes (Magic Link Token, Approval Request)
❌ Workflow definitions
❌ Custom fields on ERPNext DocTypes
❌ Business logic overrides
❌ Tenant provisioning logic
```

---

## 2. Repository Architecture

### 2.1 GitHub Repositories (What You Push)

**Your Company's GitHub:**

```
github.com/yourcompany/
├── traflo-frontend/              ← Repo 1: Next.js UI
│   └── Contains: Traflo-main/
│
└── traflo-erpnext-custom/        ← Repo 2: Custom Frappe App
    └── Contains: traflo_custom/
```

**What You DON'T Push (Generated Locally):**

```
Frappe/ERPNext (upstream repos):
├── github.com/frappe/frappe      ← Pulled by bench get-app
├── github.com/frappe/erpnext     ← Pulled by bench get-app
└── traflo-backend/ itself        ← Generated by bench init
```

### 2.2 Local Directory Structure

```
/home/ctr/CodeBases/
│
├── Traflo/
│   └── Traflo-main/              ✅ Git repo (push to GitHub)
│       ├── .git/                 ✅ Your code
│       ├── app/
│       ├── components/
│       └── lib/
│
└── traflo-backend/               ❌ NO .git (generated by bench)
    ├── apps/
    │   ├── frappe/               ❌ Upstream (bench get-app)
    │   ├── erpnext/              ❌ Upstream (bench get-app)
    │   └── traflo_custom/        ✅ Git repo (push to GitHub)
    │       ├── .git/             ✅ Your code
    │       └── traflo_custom/
    ├── sites/                    ❌ Generated (bench new-site)
    ├── env/                      ❌ Generated (bench init)
    └── config/                   ❌ Generated (bench init)
```

### 2.3 Git Repositories (2 Repos YOU Control)

```
┌─────────────────────────────────────────────────────────────┐
│  REPO 1: Frontend (YOUR CODE)                               │
│  📁 /home/ctr/CodeBases/Traflo/Traflo-main/                │
│  🔗 git@github.com:yourcompany/traflo-frontend.git         │
│  ✅ git init HERE                                           │
│  ✅ git push HERE                                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  REPO 2: Custom ERPNext App (YOUR CODE)                     │
│  📁 /home/ctr/CodeBases/traflo-backend/apps/traflo_custom/ │
│  🔗 git@github.com:yourcompany/traflo-erpnext-custom.git  │
│  ✅ git init HERE (bench new-app creates .git)             │
│  ✅ git push HERE                                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  UPSTREAM: Frappe + ERPNext (NOT YOUR CODE)                 │
│  📁 /home/ctr/CodeBases/traflo-backend/apps/frappe/        │
│  📁 /home/ctr/CodeBases/traflo-backend/apps/erpnext/       │
│  🔗 github.com/frappe/frappe (upstream)                    │
│  🔗 github.com/frappe/erpnext (upstream)                   │
│  ⚙️ Generated by: bench get-app frappe                     │
│  ⚙️ Generated by: bench get-app erpnext                    │
│  ❌ DON'T git init - DON'T push to your GitHub             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Bench Directory Structure

```
/home/ctr/CodeBases/traflo-backend/
├── apps/
│   ├── frappe/                 ← UPSTREAM (don't touch)
│   ├── erpnext/                ← UPSTREAM (don't touch)
│   └── traflo_custom/          ← YOUR CUSTOMIZATIONS ✅
│       ├── .git/               ✅ Git repo here
│       └── traflo_custom/
│           ├── hooks.py        ⭐ Main config
│           ├── api/            ← Custom endpoints
│           ├── doctype/        ← Custom tables
│           ├── overrides/      ← ERPNext extensions
│           └── fixtures/       ← Custom fields (JSON)
├── sites/
│   ├── dev.local/              ← Development site
│   └── tenant1.local/          ← Test tenant
├── config/                     ← NGINX, Redis config
├── env/                        ← Python virtualenv
└── logs/                       ← Error logs (check here!)
```

---

## 3. SRS Requirement Mapping

### 3.1 Core Modules: ERPNext → Custom

| SRS Module | ERPNext DocType | Location | Custom Work Needed |
|------------|----------------|----------|-------------------|
| **Inventory** | ✅ Item<br>✅ Warehouse<br>✅ Stock Ledger Entry<br>✅ Batch | `apps/erpnext/erpnext/stock/` | ❌ Custom fields<br>❌ Low-stock alerts<br>❌ Expiry tracking UI |
| **Sales** | ✅ Quotation<br>✅ Sales Order<br>✅ Delivery Note<br>✅ Sales Invoice | `apps/erpnext/erpnext/selling/` | ❌ Discount approval workflow<br>❌ Credit limit enforcement<br>❌ Agent mobile UI |
| **Purchase** | ✅ Purchase Order<br>✅ Purchase Receipt<br>✅ Purchase Invoice | `apps/erpnext/erpnext/buying/` | ❌ Vendor acknowledgment<br>❌ ETA tracking |
| **Customers** | ✅ Customer | `apps/erpnext/erpnext/selling/doctype/customer/` | ❌ Portal access<br>❌ Price lists |
| **Vendors** | ✅ Supplier | `apps/erpnext/erpnext/buying/doctype/supplier/` | ❌ Portal access<br>❌ PO collaboration |
| **Payments** | ✅ Payment Entry<br>✅ GL Entry | `apps/erpnext/erpnext/accounts/` | ❌ Lightweight ledger view<br>❌ Outstanding calc |
| **Users/RBAC** | ✅ User<br>✅ Role | `apps/frappe/frappe/core/doctype/user/` | ❌ Magic link auth<br>❌ Custom role limits |

### 3.2 Custom Modules (Build from Scratch)

| SRS Module | Implementation | Location |
|------------|----------------|----------|
| **Multi-Tenancy** | ❌ Tenant provisioning API<br>❌ Site creation automation | `apps/traflo_custom/traflo_custom/api/tenant.py` |
| **Magic Link Auth** | ❌ Token generation<br>❌ Email sending<br>❌ Verification | `apps/traflo_custom/traflo_custom/api/auth.py`<br>`apps/traflo_custom/traflo_custom/doctype/magic_link_token/` |
| **Approvals** | ❌ Approval workflow<br>❌ Request tracking | `apps/traflo_custom/traflo_custom/doctype/approval_request/`<br>`apps/traflo_custom/traflo_custom/fixtures/workflow.json` |
| **Audit Log** | ✅ ERPNext Version<br>❌ Enhanced filtering | `apps/erpnext/erpnext/core/doctype/version/`<br>❌ Custom UI |
| **Dashboard** | ❌ KPI aggregation<br>❌ Real-time metrics | `apps/traflo_custom/traflo_custom/api/dashboard.py` |
| **Setup Wizard** | ❌ Multi-step onboarding<br>❌ Excel import | `Traflo-main/app/(onboarding)/setup/` |
| **Reports** | ✅ ERPNext base reports<br>❌ Custom Traflo reports | `apps/traflo_custom/traflo_custom/report/` |

---

## 4. Technology Stack (Current)

### Frontend Stack

**Evidence:** `Traflo-main/package.json`

```json
{
  "framework": "Next.js 16.0.10",
  "runtime": "React 19.2.0",
  "language": "TypeScript 5.x",
  "styling": "TailwindCSS 4.1.9",
  "components": "Radix UI (shadcn/ui)",
  "forms": "React Hook Form 7.60.0 + Zod 3.25.76",
  "charts": "Recharts 2.15.4",
  "icons": "Lucide React 0.454.0",
  "http": "Native Fetch API"
}
```

**Existing Implementation:**
- ✅ API Client: [`lib/erpnext-client.ts`](./Traflo-main/lib/erpnext-client.ts)
- ✅ Types: [`lib/types.ts`](./Traflo-main/lib/types.ts)
- ✅ Hooks: [`lib/hooks/use-erpnext.ts`](./Traflo-main/lib/hooks/use-erpnext.ts)
- ✅ UI Components: [`components/ui/`](./Traflo-main/components/ui/)

### Backend Stack

```json
{
  "erp": "ERPNext 15.x",
  "framework": "Frappe Framework 15.x",
  "language": "Python 3.11+",
  "database": "MariaDB 10.11+",
  "cache": "Redis 7.x",
  "queue": "Redis Queue (RQ)",
  "webServer": "NGINX",
  "appServer": "Gunicorn"
}
```

---

## 5. Implementation Guide

### 5.1 MVP Implementation Phases

#### **Phase 1: Foundation (Week 1-2)**

**Backend Setup:**
```bash
# Already done:
✅ Frappe bench initialized
✅ ERPNext installed
✅ Custom app created

# TODO:
cd /home/ctr/CodeBases/traflo-backend/apps/traflo_custom

# 1. Create directory structure
mkdir -p traflo_custom/{api,doctype,overrides,fixtures,templates,public}

# 2. Configure hooks.py
```

**File:** `apps/traflo_custom/traflo_custom/hooks.py`
```python
app_name = "traflo_custom"
app_title = "Traflo Custom"
app_publisher = "Your Company"
app_description = "Traflo SaaS customizations"
app_email = "dev@traflo.com"
app_license = "Proprietary"

# Fixtures (export custom fields)
fixtures = [
    {"doctype": "Custom Field"},
    {"doctype": "Workflow"},
    {"doctype": "Workflow State"},
]

# Document Events (override ERPNext)
doc_events = {
    "Sales Order": {
        "validate": "traflo_custom.overrides.sales_order.validate_discount",
        "on_submit": "traflo_custom.overrides.sales_order.check_credit_limit"
    }
}
```

**Frontend Setup:**
```bash
cd /home/ctr/CodeBases/Traflo/Traflo-main

# Install dependencies (already done)
✅ pnpm install

# Configure environment
cp .env.example .env.local

# Edit .env.local:
NEXT_PUBLIC_ERPNEXT_URL=http://localhost:8000
ERPNEXT_API_KEY=your_api_key
ERPNEXT_API_SECRET=your_api_secret
```

#### **Phase 2: Core Features (Week 3-6)**

##### 2.1 Authentication

**Backend:**
```bash
# Create DocType: Magic Link Token
cd /home/ctr/CodeBases/traflo-backend
bench --site dev.local new-doctype --app traflo_custom --module "Traflo Core" "Magic Link Token"
```

**File:** `apps/traflo_custom/traflo_custom/api/auth.py`
```python
import frappe
import secrets
from datetime import datetime, timedelta

@frappe.whitelist(allow_guest=True)
def send_magic_link(email):
    """Send passwordless login link"""
    # Generate token
    token = secrets.token_urlsafe(32)

    # Store in DB
    magic_link = frappe.get_doc({
        "doctype": "Magic Link Token",
        "email": email,
        "token": token,
        "expires_at": datetime.now() + timedelta(minutes=15)
    }).insert(ignore_permissions=True)

    # Send email
    frappe.sendmail(
        recipients=email,
        subject="Traflo Login Link",
        message=f"<a href='https://app.traflo.com/auth/magic?token={token}'>Login</a>"
    )

    return {"success": True}

@frappe.whitelist(allow_guest=True)
def verify_magic_link(token):
    """Verify token and create session"""
    # Check token validity
    link = frappe.db.get_value("Magic Link Token",
                                {"token": token, "used": 0},
                                ["email", "expires_at"], as_dict=True)

    if not link or datetime.now() > link.expires_at:
        frappe.throw("Invalid or expired link")

    # Create session
    from frappe.auth import LoginManager
    login_manager = LoginManager()
    login_manager.login_as(link.email)

    return {"success": True, "sid": frappe.session.sid}
```

**Frontend:**
```tsx
// File: Traflo-main/app/(auth)/login/page.tsx
'use client'

import { useState } from 'react'
import { erpnext } from '@/lib/erpnext-client'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'

export default function LoginPage() {
  const [email, setEmail] = useState('')
  const [sent, setSent] = useState(false)

  const handleMagicLink = async () => {
    await erpnext.call('traflo_custom.api.auth.send_magic_link', { email })
    setSent(true)
  }

  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="w-full max-w-md space-y-4">
        <h1 className="text-2xl font-bold">Login to Traflo</h1>
        {!sent ? (
          <>
            <Input
              type="email"
              placeholder="your@email.com"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
            />
            <Button onClick={handleMagicLink} className="w-full">
              Send Magic Link
            </Button>
          </>
        ) : (
          <p>Check your email for the login link!</p>
        )}
      </div>
    </div>
  )
}
```

##### 2.2 Dashboard

**Backend API:**
```python
# File: apps/traflo_custom/traflo_custom/api/dashboard.py
import frappe

@frappe.whitelist()
def get_kpis():
    """Get dashboard metrics"""

    # Stock value
    stock_value = frappe.db.sql("""
        SELECT SUM(stock_value)
        FROM `tabBin`
    """)[0][0] or 0

    # Low stock items
    low_stock = frappe.db.count("Item", {
        "is_stock_item": 1,
        "actual_qty": ["<", "reorder_level"]
    })

    # Pending orders
    pending_orders = frappe.db.count("Sales Order", {
        "status": ["in", ["Draft", "To Deliver"]]
    })

    # Outstanding invoices
    overdue = frappe.db.count("Sales Invoice", {
        "docstatus": 1,
        "outstanding_amount": [">", 0],
        "due_date": ["<", frappe.utils.today()]
    })

    return {
        "stock_value": stock_value,
        "low_stock_items": low_stock,
        "pending_orders": pending_orders,
        "overdue_invoices": overdue
    }
```

**Frontend:**
```tsx
// File: Traflo-main/app/(main)/page.tsx
'use client'

import { useEffect, useState } from 'react'
import { erpnext } from '@/lib/erpnext-client'
import { Card } from '@/components/ui/card'

export default function DashboardPage() {
  const [kpis, setKpis] = useState(null)

  useEffect(() => {
    async function loadKPIs() {
      const data = await erpnext.call('traflo_custom.api.dashboard.get_kpis')
      setKpis(data.message)
    }
    loadKPIs()
  }, [])

  if (!kpis) return <div>Loading...</div>

  return (
    <div className="grid gap-4 md:grid-cols-4">
      <Card className="p-6">
        <h3 className="text-sm font-medium text-muted-foreground">Stock Value</h3>
        <p className="text-2xl font-bold">${kpis.stock_value.toLocaleString()}</p>
      </Card>
      <Card className="p-6">
        <h3 className="text-sm font-medium text-muted-foreground">Low Stock Items</h3>
        <p className="text-2xl font-bold">{kpis.low_stock_items}</p>
      </Card>
      <Card className="p-6">
        <h3 className="text-sm font-medium text-muted-foreground">Pending Orders</h3>
        <p className="text-2xl font-bold">{kpis.pending_orders}</p>
      </Card>
      <Card className="p-6">
        <h3 className="text-sm font-medium text-muted-foreground">Overdue Invoices</h3>
        <p className="text-2xl font-bold">{kpis.overdue_invoices}</p>
      </Card>
    </div>
  )
}
```

##### 2.3 Sales Orders

**Backend Override:**
```python
# File: apps/traflo_custom/traflo_custom/overrides/sales_order.py
import frappe

def validate_discount(doc, method):
    """Require approval for high discounts"""
    user_roles = frappe.get_roles(frappe.session.user)
    max_discount = 5.0  # Default

    if "Sales Manager" in user_roles:
        max_discount = 10.0

    for item in doc.items:
        if item.discount_percentage > max_discount:
            doc.workflow_state = "Pending Approval"
            frappe.msgprint(f"Discount {item.discount_percentage}% requires approval")

def check_credit_limit(doc, method):
    """Enforce customer credit limit"""
    customer = frappe.get_doc("Customer", doc.customer)

    if customer.credit_limit:
        outstanding = frappe.db.sql("""
            SELECT SUM(outstanding_amount)
            FROM `tabSales Invoice`
            WHERE customer = %s AND docstatus = 1
        """, doc.customer)[0][0] or 0

        if outstanding + doc.grand_total > customer.credit_limit:
            if not customer.bypass_credit_limit_check:
                frappe.throw(f"Credit limit exceeded: {outstanding + doc.grand_total} > {customer.credit_limit}")
```

**Frontend:**
```tsx
// File: Traflo-main/app/(main)/sales/orders/page.tsx
'use client'

import { useEffect, useState } from 'react'
import { erpnext } from '@/lib/erpnext-client'
import { Table } from '@/components/ui/table'

export default function SalesOrdersPage() {
  const [orders, setOrders] = useState([])

  useEffect(() => {
    async function loadOrders() {
      const data = await erpnext.getList(
        'Sales Order',
        {},
        ['name', 'customer', 'grand_total', 'status', 'transaction_date'],
        50
      )
      setOrders(data.data)
    }
    loadOrders()
  }, [])

  return (
    <div>
      <h1 className="text-2xl font-bold mb-4">Sales Orders</h1>
      <Table>
        <thead>
          <tr>
            <th>Order ID</th>
            <th>Customer</th>
            <th>Amount</th>
            <th>Status</th>
            <th>Date</th>
          </tr>
        </thead>
        <tbody>
          {orders.map(order => (
            <tr key={order.name}>
              <td>{order.name}</td>
              <td>{order.customer}</td>
              <td>${order.grand_total}</td>
              <td>{order.status}</td>
              <td>{order.transaction_date}</td>
            </tr>
          ))}
        </tbody>
      </Table>
    </div>
  )
}
```

#### **Phase 3: Portals & Advanced (Week 7-8)**

##### 3.1 Customer Portal

**Backend:**
```python
# File: apps/traflo_custom/traflo_custom/api/portal.py
import frappe

@frappe.whitelist()
def get_customer_orders():
    """Get orders for logged-in customer"""
    # Get customer linked to current user
    customer = frappe.db.get_value("Contact", {"user": frappe.session.user}, "customer")

    if not customer:
        return []

    orders = frappe.get_all(
        "Sales Order",
        filters={"customer": customer},
        fields=["name", "grand_total", "status", "transaction_date"],
        order_by="transaction_date desc"
    )

    return orders
```

**Frontend:**
```tsx
// File: Traflo-main/app/(portal)/customer/page.tsx
'use client'

import { useEffect, useState } from 'react'
import { erpnext } from '@/lib/erpnext-client'
import { Card } from '@/components/ui/card'

export default function CustomerPortalPage() {
  const [orders, setOrders] = useState([])

  useEffect(() => {
    async function loadOrders() {
      const data = await erpnext.call('traflo_custom.api.portal.get_customer_orders')
      setOrders(data.message)
    }
    loadOrders()
  }, [])

  return (
    <div className="container py-8">
      <h1 className="text-3xl font-bold mb-6">My Orders</h1>
      <div className="grid gap-4">
        {orders.map(order => (
          <Card key={order.name} className="p-4">
            <div className="flex justify-between">
              <div>
                <p className="font-semibold">{order.name}</p>
                <p className="text-sm text-muted-foreground">{order.transaction_date}</p>
              </div>
              <div className="text-right">
                <p className="font-bold">${order.grand_total}</p>
                <p className="text-sm">{order.status}</p>
              </div>
            </div>
          </Card>
        ))}
      </div>
    </div>
  )
}
```

### 5.2 Custom Fields Management

**Adding Custom Fields:**

```bash
# Method 1: Via ERPNext UI (Recommended for quick prototyping)
# 1. Go to http://localhost:8000
# 2. Search "Customize Form"
# 3. Select "Sales Order"
# 4. Add field: "discount_approval_required" (Check)

# Method 2: Via bench console
cd /home/ctr/CodeBases/traflo-backend
bench --site dev.local console

# In console:
frappe.get_doc({
    "doctype": "Custom Field",
    "dt": "Sales Order",
    "fieldname": "discount_approval_required",
    "fieldtype": "Check",
    "label": "Discount Approval Required",
    "insert_after": "discount_amount"
}).insert()

# CRITICAL: Export fixtures to version control
bench --site dev.local export-fixtures
```

**Result:**
- File created: `apps/traflo_custom/traflo_custom/fixtures/custom_field.json`
- Commit this to Git!

```bash
cd apps/traflo_custom
git add traflo_custom/fixtures/custom_field.json
git commit -m "Add discount_approval_required field to Sales Order"
git push
```

### 5.3 Deployment Workflow

**Development:**
```bash
# Frontend (auto-reloads)
cd /home/ctr/CodeBases/Traflo/Traflo-main
pnpm dev

# Backend
cd /home/ctr/CodeBases/traflo-backend
bench start
```

**Production:**
```bash
# Frontend: Push to GitHub → Vercel auto-deploys
git push origin main

# Backend: On production server
cd /home/frappe/traflo-backend/apps/traflo_custom
git pull origin main
cd ../..
bench --site all migrate
bench build
bench restart
```

---

## 6. File Structure & Shortcuts

### 6.1 Frontend Key Files

Click to open in VSCode:

**Core Files:**
- [`Traflo-main/lib/erpnext-client.ts`](./Traflo-main/lib/erpnext-client.ts) - API client
- [`Traflo-main/lib/types.ts`](./Traflo-main/lib/types.ts) - TypeScript types
- [`Traflo-main/lib/utils.ts`](./Traflo-main/lib/utils.ts) - Utilities
- [`Traflo-main/package.json`](./Traflo-main/package.json) - Dependencies

**Pages (Existing Routes):**
- [`Traflo-main/app/(main)/page.tsx`](./Traflo-main/app/(main)/page.tsx) - Dashboard
- [`Traflo-main/app/(main)/sales/page.tsx`](./Traflo-main/app/(main)/sales/page.tsx) - Sales
- [`Traflo-main/app/(main)/inventory/page.tsx`](./Traflo-main/app/(main)/inventory/page.tsx) - Inventory
- [`Traflo-main/app/(portal)/customer/page.tsx`](./Traflo-main/app/(portal)/customer/page.tsx) - Customer Portal
- [`Traflo-main/app/(agent)/agent/page.tsx`](./Traflo-main/app/(agent)/agent/page.tsx) - Sales Agent

**Components:**
- [`Traflo-main/components/app-sidebar.tsx`](./Traflo-main/components/app-sidebar.tsx) - Sidebar
- [`Traflo-main/components/ui/`](./Traflo-main/components/ui/) - UI components

### 6.2 Backend Key Files

**Custom App:**
- [`traflo-backend/apps/traflo_custom/traflo_custom/hooks.py`](./traflo-backend/apps/traflo_custom/traflo_custom/hooks.py) - Main config
- `traflo-backend/apps/traflo_custom/traflo_custom/api/` - Custom APIs (to create)
- `traflo-backend/apps/traflo_custom/traflo_custom/overrides/` - ERPNext overrides (to create)
- `traflo-backend/apps/traflo_custom/traflo_custom/fixtures/` - Custom fields (to export)

**ERPNext Core (Reference Only - Don't Modify):**
- `traflo-backend/apps/erpnext/erpnext/selling/doctype/sales_order/` - Sales Order
- `traflo-backend/apps/erpnext/erpnext/stock/doctype/item/` - Item
- `traflo-backend/apps/erpnext/erpnext/buying/doctype/purchase_order/` - PO
- `traflo-backend/apps/erpnext/erpnext/accounts/doctype/payment_entry/` - Payment

### 6.3 Configuration Files

- [`Traflo-main/.env.local`](./Traflo-main/.env.local) - Frontend env vars
- [`Traflo-main/next.config.mjs`](./Traflo-main/next.config.mjs) - Next.js config
- [`Traflo-main/tailwind.config.js`](./Traflo-main/tailwind.config.js) - Tailwind
- `traflo-backend/sites/dev.local/site_config.json` - Site config

---

## 7. Quick Start Commands

### First Time Setup

```bash
# Backend
cd /home/ctr/CodeBases/traflo-backend
bench start

# Frontend (new terminal)
cd /home/ctr/CodeBases/Traflo/Traflo-main
cp .env.example .env.local
# Edit .env.local with ERPNext credentials
pnpm dev
```

### Daily Development

```bash
# Start backend
bench start

# Start frontend (new terminal)
pnpm dev

# Export fixtures after changes
bench --site dev.local export-fixtures

# Commit changes
cd apps/traflo_custom
git add .
git commit -m "Description"
git push
```

### Creating New Features

**Backend API:**
```bash
# 1. Create file
touch apps/traflo_custom/traflo_custom/api/my_feature.py

# 2. Write code (see examples above)

# 3. Register in hooks.py

# 4. Restart
bench restart
```

**Frontend Page:**
```bash
# 1. Create file
mkdir -p Traflo-main/app/(main)/my-feature
touch Traflo-main/app/(main)/my-feature/page.tsx

# 2. Write code (see examples above)

# 3. Auto-reloads in dev mode
```

---

## 8. SRS Requirements Checklist

### MVP (Must Have)

#### Authentication (FR-1)
- [ ] Email + password login
- [ ] Magic link authentication
- [ ] Multi-org support
- [ ] RBAC enforcement

**Implementation:**
- Backend: `apps/traflo_custom/traflo_custom/api/auth.py`
- Frontend: `Traflo-main/app/(auth)/`
- DocType: `Magic Link Token`

#### Inventory (FR-3)
- [x] ERPNext Item ✅
- [x] ERPNext Warehouse ✅
- [x] ERPNext Stock Ledger Entry ✅
- [ ] Custom: Low-stock alerts
- [ ] Custom: Expiry tracking UI
- [ ] Custom: Batch selector component

**Implementation:**
- Use ERPNext: `apps/erpnext/erpnext/stock/`
- Custom API: `apps/traflo_custom/traflo_custom/api/inventory.py`
- Frontend: `Traflo-main/app/(main)/inventory/`

#### Sales Lifecycle (FR-4)
- [x] ERPNext Quotation ✅
- [x] ERPNext Sales Order ✅
- [x] ERPNext Delivery Note ✅
- [x] ERPNext Sales Invoice ✅
- [ ] Custom: Discount approval workflow
- [ ] Custom: Credit limit check

**Implementation:**
- Use ERPNext: `apps/erpnext/erpnext/selling/`
- Override: `apps/traflo_custom/traflo_custom/overrides/sales_order.py`
- Frontend: `Traflo-main/app/(main)/sales/`

#### Purchase Lifecycle (FR-5)
- [x] ERPNext Purchase Order ✅
- [x] ERPNext Purchase Receipt ✅
- [x] ERPNext Purchase Invoice ✅
- [ ] Custom: Vendor acknowledgment
- [ ] Custom: ETA tracking

**Implementation:**
- Use ERPNext: `apps/erpnext/erpnext/buying/`
- Custom fields: `acknowledgement_status`, `eta_date`
- Frontend: `Traflo-main/app/(main)/purchases/`

#### Customer Portal (FR-8)
- [ ] Product catalog with stock
- [ ] Order placement
- [ ] Invoice viewing
- [ ] Order tracking

**Implementation:**
- Backend: `apps/traflo_custom/traflo_custom/api/portal.py`
- Frontend: `Traflo-main/app/(portal)/customer/`

#### Approvals (FR-10)
- [ ] Discount overrides
- [ ] Stock adjustments
- [ ] Warehouse transfers
- [ ] Approval logging

**Implementation:**
- DocType: `Approval Request`
- Workflow: `apps/traflo_custom/traflo_custom/fixtures/workflow.json`
- Frontend: `Traflo-main/app/(main)/approvals/`

#### Audit (FR-11)
- [x] ERPNext Version DocType ✅
- [ ] Custom: Enhanced filtering
- [ ] Custom: Export functionality

**Implementation:**
- Use ERPNext: `apps/frappe/frappe/core/doctype/version/`
- Frontend: `Traflo-main/app/(main)/audit/`

---

## 9. Common Tasks

### Add Custom Field

```bash
# Via UI
1. Go to Customize Form
2. Select DocType
3. Add field
4. Export fixtures: bench --site dev.local export-fixtures
5. Commit: git add fixtures/ && git commit -m "Add field"
```

### Create Custom API

```python
# File: apps/traflo_custom/traflo_custom/api/example.py
import frappe

@frappe.whitelist()
def my_custom_method(param1, param2):
    """My custom endpoint"""
    # Your logic here
    return {"result": "success"}

# Register in hooks.py (no need - whitelisted methods auto-exposed)

# Call from frontend:
# erpnext.call('traflo_custom.api.example.my_custom_method', {param1: 'x', param2: 'y'})
```

### Override ERPNext Behavior

```python
# File: apps/traflo_custom/traflo_custom/overrides/my_doctype.py
import frappe

def before_save(doc, method):
    """Triggered before saving"""
    # Your logic

def validate(doc, method):
    """Triggered on validation"""
    # Your logic

# Register in hooks.py:
# doc_events = {
#     "Sales Order": {
#         "before_save": "traflo_custom.overrides.my_doctype.before_save"
#     }
# }
```

---

## 10. Troubleshooting

### Backend Issues

**Check Logs:**
```bash
tail -f /home/ctr/CodeBases/traflo-backend/logs/web.error.log
tail -f /home/ctr/CodeBases/traflo-backend/logs/worker.error.log
```

**Clear Cache:**
```bash
bench --site dev.local clear-cache
bench restart
```

**Database Issues:**
```bash
bench --site dev.local migrate
bench --site dev.local rebuild-global-search
```

### Frontend Issues

**Clear Build:**
```bash
rm -rf .next
pnpm dev
```

**Check API Connection:**
```bash
curl http://localhost:8000/api/method/frappe.auth.get_logged_user
```

---

## 11. Next Steps

1. ✅ Read this SDS
2. ⬜ Set up development environment
3. ⬜ Create first custom API
4. ⬜ Build dashboard UI
5. ⬜ Implement sales order workflow
6. ⬜ Add custom fields & workflows
7. ⬜ Build customer portal
8. ⬜ Deploy to staging
9. ⬜ User testing
10. ⬜ Production deployment

---

**This is your BUILD GUIDE. Keep it open while developing.**

**Questions? Check:**
- ERPNext Docs: https://docs.erpnext.com
- Frappe Docs: https://frappeframework.com/docs
- This SDS

---

*Last Updated: 2026-01-03*
*Status: Ready to Build*
