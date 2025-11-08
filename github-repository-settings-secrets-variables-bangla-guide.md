# GitHub Repository Settings - Secrets & Variables সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [GitHub Repository Settings Overview](#github-repository-settings-overview)
- [Secrets এবং Variables কি?](#secrets-এবং-variables-কি)
- [Secrets Management বিস্তারিত](#secrets-management-বিস্তারিত)
- [Environment Variables Management](#environment-variables-management)
- [Repository Secrets vs Environment Secrets](#repository-secrets-vs-environment-secrets)
- [Laravel Project এ Secrets ব্যবহার](#laravel-project-এ-secrets-ব্যবহার)
- [Security Best Practices](#security-best-practices)
- [Troubleshooting ও Common Issues](#troubleshooting-ও-common-issues)

---

## GitHub Repository Settings Overview

### 🏗️ **General Section বিস্তারিত**

**1. Repository Name:**
```
Repository: legacylock-api
- নাম পরিবর্তন করতে "Rename" বাটন ক্লিক করুন
- নাম পরিবর্তনের পর সব clone URLs আপডেট হবে
- Collaborators দের নতুন URL জানাতে হবে
```

**2. Template Repository:**
```
✅ Template repository হিসেবে mark করলে:
- অন্যরা এই repository structure ব্যবহার করতে পারবে
- "Use this template" বাটন দেখাবে
- নতুন repository তৈরিতে সহায়ক
```

**3. Require Contributors Sign-off:**
```
✅ Enable করলে:
- সব web-based commits এ DCO (Developer Certificate of Origin) প্রয়োজন
- Legal compliance নিশ্চিত করে
- Enterprise projects এর জন্য গুরুত্বপূর্ণ
```

### 🌿 **Default Branch Configuration**

**Dev Branch as Default:**
```bash
# কেন dev branch default করা হয়েছে:
- Production code main/master branch এ থাকে
- Development work dev branch এ হয়
- Pull requests dev → main এ merge হয়
- CI/CD pipeline dev branch trigger করে

# Branch protection rules:
- Direct push prevent করা
- Pull request review required
- Status checks must pass
```

### 📦 **Features Section**

**1. Wikis:**
```
✅ Wikis enabled:
- Project documentation লেখার জন্য
- User guides, API docs তৈরি করা যায়
- Markdown format support
- Version control সহ
```

**2. Issues:**
```
✅ Issues tracking:
- Bug reports
- Feature requests  
- Project management
- Labels ও milestones
```

**3. Projects:**
```
✅ Project boards:
- Kanban style task management
- Sprint planning
- Progress tracking
```

---

## Secrets এবং Variables কি?

### 🔐 **GitHub Secrets**

**Secrets হলো encrypted environment variables** যা sensitive information store করার জন্য ব্যবহৃত হয়।

**কি ধরনের তথ্য Secrets এ রাখা হয়:**
- API Keys (AWS, Google Cloud, Stripe)
- Database passwords
- SSH private keys
- OAuth tokens
- Encryption keys
- Third-party service credentials

### 📊 **GitHub Variables**

**Variables হলো non-sensitive configuration data** যা publicly visible হতে পারে।

**কি ধরনের তথ্য Variables এ রাখা হয়:**
- Environment names (production, staging)
- Public API endpoints
- Configuration flags
- Build settings
- Non-sensitive URLs

---

## Secrets Management বিস্তারিত

### 🔒 **Repository Secrets Setup**

**Step 1: Secrets Page এ যাওয়া**
```
Repository → Settings → Security → Secrets and variables → Actions
```

**Step 2: New Repository Secret তৈরি**
```
1. "New repository secret" ক্লিক করুন
2. Secret name দিন (uppercase, underscore allowed)
3. Secret value paste করুন
4. "Add secret" ক্লিক করুন
```

### 🎯 **Laravel Project এর জন্য Common Secrets**

**Database Secrets:**
```
DB_PASSWORD=your_database_password
DB_HOST=your_database_host
DB_USERNAME=your_database_username
```

**API Keys:**
```
STRIPE_SECRET_KEY=sk_live_xxxxxxxxxxxxx
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
PUSHER_APP_SECRET=your_pusher_secret
MAIL_PASSWORD=your_mail_password
```

**OAuth Credentials:**
```
GOOGLE_CLIENT_SECRET=your_google_client_secret
FACEBOOK_CLIENT_SECRET=your_facebook_client_secret
GITHUB_CLIENT_SECRET=your_github_client_secret
```

**Deployment Keys:**
```
SSH_PRIVATE_KEY=-----BEGIN OPENSSH PRIVATE KEY-----
DEPLOY_TOKEN=your_deployment_token
SERVER_HOST=your_server_ip
SERVER_USER=your_server_username
```

### 🌍 **Environment-Specific Secrets**

**Production Environment:**
```
Environment name: production

Secrets:
- DB_PASSWORD_PROD=production_db_password
- API_KEY_PROD=production_api_key
- REDIS_PASSWORD_PROD=production_redis_password
```

**Staging Environment:**
```
Environment name: staging

Secrets:
- DB_PASSWORD_STAGING=staging_db_password
- API_KEY_STAGING=staging_api_key
- REDIS_PASSWORD_STAGING=staging_redis_password
```

**Development Environment:**
```
Environment name: development

Secrets:
- DB_PASSWORD_DEV=development_db_password
- API_KEY_DEV=development_api_key
- REDIS_PASSWORD_DEV=development_redis_password
```

---

## Environment Variables Management

### 📝 **Repository Variables Setup**

**Non-sensitive Configuration:**
```
APP_ENV=production
APP_DEBUG=false
LOG_LEVEL=error
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
```

**Build Configuration:**
```
NODE_VERSION=18
PHP_VERSION=8.2
COMPOSER_VERSION=2.5
BUILD_ENVIRONMENT=production
```

**Feature Flags:**
```
ENABLE_FEATURE_X=true
MAINTENANCE_MODE=false
DEBUG_TOOLBAR=false
API_RATE_LIMIT=1000
```

### 🎛️ **Environment Variables vs Secrets**

| Type | Variables | Secrets |
|------|-----------|---------|
| **Visibility** | Public (in logs) | Encrypted |
| **Use Case** | Configuration | Sensitive data |
| **Examples** | APP_ENV, DEBUG | API_KEY, PASSWORD |
| **Access** | Anyone with repo access | Restricted |
| **Logging** | Visible in workflow logs | Hidden (***) |

---

## Repository Secrets vs Environment Secrets

### 🏢 **Repository Level Secrets**

**কখন ব্যবহার করবেন:**
- সব environments এ same secret
- Development/testing এর জন্য
- Simple projects

**Example Setup:**
```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run tests
      env:
        DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        API_KEY: ${{ secrets.API_KEY }}
      run: php artisan test
```

### 🌐 **Environment Level Secrets**

**কখন ব্যবহার করবেন:**
- Different secrets for different environments
- Production vs staging separation
- Enhanced security

**Environment Setup:**
```
1. Repository → Settings → Environments
2. "New environment" ক্লিক করুন
3. Environment name দিন (production, staging, development)
4. Protection rules set করুন
5. Environment-specific secrets add করুন
```

**Workflow এ Environment ব্যবহার:**
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to staging
      env:
        DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        API_KEY: ${{ secrets.API_KEY }}
        SERVER_HOST: ${{ secrets.SERVER_HOST }}
      run: |
        echo "Deploying to staging..."
        # Deployment commands

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to production
      env:
        DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        API_KEY: ${{ secrets.API_KEY }}
        SERVER_HOST: ${{ secrets.SERVER_HOST }}
      run: |
        echo "Deploying to production..."
        # Production deployment commands
```

---

## Laravel Project এ Secrets ব্যবহার

### 🚀 **Complete Laravel Deployment Workflow**

```yaml
name: Laravel CI/CD

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: ${{ secrets.DB_PASSWORD }}
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3
      
      redis:
        image: redis:7.0
        ports:
          - 6379:6379

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
        extensions: mbstring, dom, fileinfo, mysql, gd, redis

    - name: Install dependencies
      run: composer install --prefer-dist --no-progress

    - name: Create .env file
      run: |
        cp .env.example .env
        echo "APP_KEY=${{ secrets.APP_KEY }}" >> .env
        echo "DB_CONNECTION=mysql" >> .env
        echo "DB_HOST=127.0.0.1" >> .env
        echo "DB_PORT=3306" >> .env
        echo "DB_DATABASE=testing" >> .env
        echo "DB_USERNAME=root" >> .env
        echo "DB_PASSWORD=${{ secrets.DB_PASSWORD }}" >> .env
        echo "REDIS_HOST=127.0.0.1" >> .env
        echo "REDIS_PORT=6379" >> .env

    - name: Run migrations
      run: php artisan migrate --force

    - name: Run tests
      run: php artisan test

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    environment: staging
    if: github.ref == 'refs/heads/dev'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Deploy to staging server
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.STAGING_HOST }}
        username: ${{ secrets.STAGING_USER }}
        key: ${{ secrets.STAGING_SSH_KEY }}
        script: |
          cd /var/www/staging
          git pull origin dev
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          php artisan queue:restart
          sudo service php8.2-fpm restart

  deploy-production:
    needs: test
    runs-on: ubuntu-latest
    environment: production
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Deploy to production server
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.PROD_HOST }}
        username: ${{ secrets.PROD_USER }}
        key: ${{ secrets.PROD_SSH_KEY }}
        script: |
          cd /var/www/production
          php artisan down
          git pull origin main
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          php artisan queue:restart
          sudo service php8.2-fpm restart
          php artisan up

    - name: Notify deployment
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
      if: always()
```

### 🔧 **Environment File Generation**

**Dynamic .env Creation:**
```yaml
- name: Create production .env
  run: |
    echo "APP_NAME=${{ vars.APP_NAME }}" > .env
    echo "APP_ENV=production" >> .env
    echo "APP_KEY=${{ secrets.APP_KEY }}" >> .env
    echo "APP_DEBUG=false" >> .env
    echo "APP_URL=${{ vars.APP_URL }}" >> .env
    echo "" >> .env
    echo "LOG_CHANNEL=stack" >> .env
    echo "LOG_LEVEL=error" >> .env
    echo "" >> .env
    echo "DB_CONNECTION=mysql" >> .env
    echo "DB_HOST=${{ secrets.DB_HOST }}" >> .env
    echo "DB_PORT=3306" >> .env
    echo "DB_DATABASE=${{ secrets.DB_DATABASE }}" >> .env
    echo "DB_USERNAME=${{ secrets.DB_USERNAME }}" >> .env
    echo "DB_PASSWORD=${{ secrets.DB_PASSWORD }}" >> .env
    echo "" >> .env
    echo "BROADCAST_DRIVER=redis" >> .env
    echo "CACHE_DRIVER=redis" >> .env
    echo "FILESYSTEM_DISK=local" >> .env
    echo "QUEUE_CONNECTION=redis" >> .env
    echo "SESSION_DRIVER=redis" >> .env
    echo "SESSION_LIFETIME=120" >> .env
    echo "" >> .env
    echo "REDIS_HOST=${{ secrets.REDIS_HOST }}" >> .env
    echo "REDIS_PASSWORD=${{ secrets.REDIS_PASSWORD }}" >> .env
    echo "REDIS_PORT=6379" >> .env
    echo "" >> .env
    echo "MAIL_MAILER=smtp" >> .env
    echo "MAIL_HOST=${{ secrets.MAIL_HOST }}" >> .env
    echo "MAIL_PORT=587" >> .env
    echo "MAIL_USERNAME=${{ secrets.MAIL_USERNAME }}" >> .env
    echo "MAIL_PASSWORD=${{ secrets.MAIL_PASSWORD }}" >> .env
    echo "MAIL_ENCRYPTION=tls" >> .env
    echo "" >> .env
    echo "AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}" >> .env
    echo "AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}" >> .env
    echo "AWS_DEFAULT_REGION=${{ vars.AWS_DEFAULT_REGION }}" >> .env
    echo "AWS_BUCKET=${{ vars.AWS_BUCKET }}" >> .env
    echo "" >> .env
    echo "PUSHER_APP_ID=${{ secrets.PUSHER_APP_ID }}" >> .env
    echo "PUSHER_APP_KEY=${{ secrets.PUSHER_APP_KEY }}" >> .env
    echo "PUSHER_APP_SECRET=${{ secrets.PUSHER_APP_SECRET }}" >> .env
    echo "PUSHER_APP_CLUSTER=${{ vars.PUSHER_APP_CLUSTER }}" >> .env
```

---

## Security Best Practices

### 🛡️ **Secrets Security Guidelines**

**1. Naming Conventions:**
```
✅ Good naming:
- DB_PASSWORD
- API_KEY_STRIPE
- SSH_PRIVATE_KEY_PROD

❌ Bad naming:
- password
- key
- secret
```

**2. Secret Rotation:**
```bash
# Regular rotation schedule:
- Database passwords: Every 90 days
- API keys: Every 6 months
- SSH keys: Every year
- OAuth tokens: When compromised

# Rotation process:
1. Generate new secret
2. Update in GitHub Secrets
3. Deploy to all environments
4. Verify functionality
5. Revoke old secret
```

**3. Access Control:**
```
Repository Settings → Manage access:
- Limit who can view/edit secrets
- Use teams for group access
- Regular access review
- Remove inactive collaborators
```

**4. Environment Protection:**
```
Environment Settings:
✅ Required reviewers
✅ Wait timer (production)
✅ Deployment branches (main only)
✅ Environment secrets isolation
```

### 🔍 **Secret Scanning**

**GitHub Advanced Security:**
```
Repository → Security → Code scanning alerts

Features:
- Automatic secret detection
- Partner pattern matching
- Custom pattern rules
- Alert notifications
- Remediation guidance
```

**Common Detected Patterns:**
```
- AWS Access Keys
- Google API Keys
- Stripe API Keys
- GitHub Personal Access Tokens
- SSH Private Keys
- Database Connection Strings
- JWT Tokens
```

### 📊 **Audit ও Monitoring**

**Secret Usage Tracking:**
```yaml
# Log secret usage (without exposing values)
- name: Audit secret usage
  run: |
    echo "Using DB_PASSWORD for environment: ${{ github.environment }}"
    echo "Deployment triggered by: ${{ github.actor }}"
    echo "Commit SHA: ${{ github.sha }}"
    echo "Workflow: ${{ github.workflow }}"
```

**Monitoring Checklist:**
```
✅ Failed deployment alerts
✅ Unauthorized access attempts
✅ Secret rotation reminders
✅ Environment access logs
✅ Workflow execution history
```

---

## Troubleshooting ও Common Issues

### 🚨 **Common Problems**

**1. Secret Not Found Error:**
```yaml
# Error: Secret DB_PASSWORD not found

# Solution: Check secret name spelling
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # Correct
  # DB_PASSWORD: ${{ secrets.db_password }}  # Wrong (case sensitive)
```

**2. Environment Secret Access:**
```yaml
# Error: Cannot access environment secret

# Solution: Specify environment in job
jobs:
  deploy:
    environment: production  # Required for environment secrets
    steps:
      - name: Use secret
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

**3. Secret Value Issues:**
```bash
# Issue: Multiline secrets (SSH keys, certificates)

# Solution: Use proper formatting
SSH_PRIVATE_KEY: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAFwAAAAdzc2gtcn
  ...
  -----END OPENSSH PRIVATE KEY-----
```

**4. Secret Visibility in Logs:**
```yaml
# Issue: Secret appears in logs

# Solution: GitHub automatically masks secrets
- name: Debug (secrets are masked)
  run: |
    echo "Database password: ${{ secrets.DB_PASSWORD }}"
    # Output: Database password: ***
```

### 🔧 **Debugging Secrets**

**1. Test Secret Availability:**
```yaml
- name: Test secrets
  run: |
    if [ -z "${{ secrets.DB_PASSWORD }}" ]; then
      echo "DB_PASSWORD secret is not set"
      exit 1
    else
      echo "DB_PASSWORD secret is available"
    fi
```

**2. Environment Variable Check:**
```yaml
- name: Check environment
  run: |
    echo "Environment: ${{ github.environment }}"
    echo "Repository: ${{ github.repository }}"
    echo "Actor: ${{ github.actor }}"
    env | grep -E '^(APP_|DB_|REDIS_)' | sort
```

**3. Conditional Secret Usage:**
```yaml
- name: Use secret conditionally
  if: ${{ secrets.OPTIONAL_SECRET != '' }}
  env:
    OPTIONAL_SECRET: ${{ secrets.OPTIONAL_SECRET }}
  run: |
    echo "Optional secret is available"
```

### 📋 **Best Practices Checklist**

**Setup Checklist:**
```
✅ Secrets properly named (UPPERCASE_WITH_UNDERSCORES)
✅ Environment-specific secrets separated
✅ Non-sensitive data in Variables
✅ Regular secret rotation schedule
✅ Access control configured
✅ Secret scanning enabled
✅ Backup of critical secrets (securely stored)
✅ Documentation of secret purposes
✅ Team training on secret management
✅ Incident response plan for compromised secrets
```

**Security Checklist:**
```
✅ No secrets in code/comments
✅ No secrets in commit messages
✅ Environment protection rules
✅ Required reviewers for production
✅ Audit logs monitoring
✅ Regular access review
✅ Automated secret detection
✅ Secure secret sharing process
```

---

## 🎯 Quick Reference

### Daily Operations:
```bash
# Add new secret
Repository → Settings → Secrets → New repository secret

# Update existing secret
Repository → Settings → Secrets → [Secret Name] → Update

# Check secret usage
Repository → Actions → [Workflow] → Check logs for masked values
```

### Emergency Procedures:
```bash
# Compromised secret:
1. Immediately revoke/change the secret value
2. Update GitHub secret
3. Redeploy all affected environments
4. Monitor for unauthorized access
5. Review access logs
```

### Environment Setup:
```bash
# Create environment:
Repository → Settings → Environments → New environment

# Add environment secrets:
Environment → Secrets → New environment secret

# Configure protection rules:
Environment → Protection rules → Configure
```

এই comprehensive guide follow করে আপনি GitHub repository এর secrets এবং variables professionally manage করতে পারবেন, যা আপনার Laravel project এর security এবং deployment process কে significantly improve করবে।