# Apigee Certificate Management Solution

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![Apigee](https://img.shields.io/badge/Platform-Apigee%20Edge-FF6C37?logo=google-cloud&logoColor=white)](https://cloud.google.com/apigee)

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Workflows](#workflows)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [Support](#support)
- [License](#license)

## 🎯 Overview

The Apigee Certificate Management Solution is an enterprise-grade automation framework that manages the complete lifecycle of SSL/TLS certificates in Apigee Edge environments. It eliminates manual certificate management, prevents expiration-related outages, and ensures compliance with security policies.

### Business Value

- **🚀 Zero Downtime**: Proactive certificate renewal before expiration
- **🤖 Full Automation**: Eliminates manual certificate management tasks
- **🔒 Enhanced Security**: Enforces approval workflows and audit trails
- **💰 Cost Reduction**: Prevents costly certificate expiration incidents
- **📊 Complete Visibility**: Real-time monitoring and alerting

## ✨ Features

### Core Capabilities

- **Automated Certificate Lifecycle Management**
  - Daily monitoring of certificate expiration dates
  - Automatic renewal 30 days before expiry
  - Support for multiple certificate types (Self-signed, Let's Encrypt, DigiCert)
  
- **Multi-Environment Support**
  - Separate configurations for Dev, Test, UAT, and Production
  - Environment-specific security policies
  - Isolated credential management

- **Production-Grade Security**
  - Approval gates for production deployments
  - Change ticket validation
  - Automated backup before modifications
  - Complete audit trail

- **Intelligent Notifications**
  - Slack integration for real-time alerts
  - Email notifications for critical events
  - Customizable alert thresholds

## 🏗️ Architecture

### High-Level Architecture

```
┌─────────────────────┐
│   GitHub Actions    │
│  (Orchestration)    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐     ┌─────────────────────┐
│   Core Scripts      │     │  Configuration      │
│  - apigee_auth.sh   │────▶│  - apigee-config    │
│  - apigee_keystore  │     │  - env configs      │
│  - apigee_certs.sh  │     │  - secrets          │
└─────────┬───────────┘     └─────────────────────┘
          │
          ▼
┌─────────────────────┐     ┌─────────────────────┐
│   Apigee Edge API   │     │ Certificate Sources │
│  - Environments     │     │  - Self-Signed      │
│  - Keystores        │────▶│  - Let's Encrypt    │
│  - Certificates     │     │  - DigiCert         │
└─────────┬───────────┘     └─────────────────────┘
          │
          ▼
┌─────────────────────┐
│   Notifications     │
│  - Slack            │
│  - Email            │
└─────────────────────┘
```

### Components

| Component | Description | Technology |
|-----------|-------------|------------|
| **Orchestration** | Workflow automation and scheduling | GitHub Actions |
| **Core Scripts** | Business logic implementation | Bash |
| **Configuration** | Environment and organization settings | JSON |
| **API Integration** | Apigee Edge management | REST API |
| **Certificate Sources** | Certificate generation and retrieval | OpenSSL, Certbot |
| **Notifications** | Alert and status reporting | Slack, Email |

## 📚 Prerequisites

### System Requirements

- **GitHub**: Enterprise or Pro account with Actions enabled
- **Apigee Edge**: Enterprise account with API access
- **Operating System**: Ubuntu 20.04+ (for GitHub runners)

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Bash | 4.0+ | Script execution |
| OpenSSL | 1.1.1+ | Certificate operations |
| Certbot | Latest | Let's Encrypt integration |
| jq | 1.6+ | JSON parsing |
| curl | 7.68+ | API communication |

### Access Requirements

- Apigee Edge SSO credentials for each environment
- GitHub repository admin access
- DNS provider API credentials (for Let's Encrypt)
- Slack webhook URL (optional)

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/apigee-cert-management.git
cd apigee-cert-management
```

### 2. Set Up GitHub Secrets

```bash
# Using GitHub CLI
gh secret set APIGEE_ORG --body "your-organization"
gh secret set APIGEE_SSO_URL --body "https://your-sso.login.apigee.com"
gh secret set APIGEE_API_URL --body "https://api.enterprise.apigee.com/v1"

# Set environment-specific secrets
gh secret set APIGEE_SSO_PASSCODE_DEV --env dev
gh secret set KEYSTORE_PASSWORD_DEV --env dev
```

### 3. Configure Your Environment

```bash
# Edit global configuration
vi config/apigee-config.json

# Edit environment configuration
vi config/environments/dev.json
```

### 4. Run Your First Certificate Check

```bash
# Trigger manual workflow
gh workflow run apigee-cert-management.yml \
  -f environment=dev \
  -f check_only=true
```

## 📦 Installation

### Step 1: Repository Setup

```bash
# Create directory structure
mkdir -p {.github/workflows,scripts,config/environments,certs,docs,tests}

# Copy workflow files
cp workflows/*.yml .github/workflows/

# Copy scripts
cp scripts/*.sh scripts/
chmod +x scripts/*.sh

# Create configuration templates
cp templates/apigee-config.json config/
cp templates/env-config.json config/environments/dev.json
```

### Step 2: Configure GitHub Environments

1. Navigate to Settings → Environments
2. Create environments: `dev`, `test`, `uat`, `production`
3. Configure protection rules for production:
   - Required reviewers: 2
   - Restrict to protected branches
   - Add environment secrets

### Step 3: Set Up Secrets

Required secrets for each environment:

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `APIGEE_ORG` | Organization name | `my-company` |
| `APIGEE_SSO_PASSCODE_[ENV]` | SSO passcode | `abc123...` |
| `KEYSTORE_PASSWORD_[ENV]` | Keystore password | `SecureP@ss` |
| `LETS_ENCRYPT_EMAIL` | Let's Encrypt email | `certs@company.com` |
| `SLACK_WEBHOOK_URL` | Slack notifications | `https://hooks.slack.com/...` |

## ⚙️ Configuration

### Global Configuration (`config/apigee-config.json`)

```json
{
  "organization": "your-org-name",
  "sso_url": "https://your-org-sso.login.apigee.com",
  "api_url": "https://api.enterprise.apigee.com/v1",
  "scheduled_environments": ["dev", "test", "uat"],
  "cert_renewal_days": 30,
  "notification_channels": {
    "slack": {
      "enabled": true,
      "webhook_url_secret": "SLACK_WEBHOOK_URL"
    },
    "email": {
      "enabled": false,
      "recipients": ["team@example.com"]
    }
  }
}
```

### Environment Configuration (`config/environments/dev.json`)

```json
{
  "keystore_name": "dev-app-keystore",
  "domain": "app-dev.example.com",
  "alias_name": "app-dev-cert",
  "cert_type": "self-signed",
  "additional_domains": [],
  "notification_overrides": {
    "slack_channel": "#dev-alerts"
  }
}
```

### Production Configuration (`config/environments/prod.json`)

```json
{
  "keystore_name": "prod-app-keystore",
  "domain": "api.example.com",
  "alias_name": "api-prod-cert",
  "cert_type": "lets-encrypt",
  "dns_plugin": "route53",
  "additional_domains": ["www.api.example.com"],
  "domain_pattern": "^(api\\.|www\\.api\\.)?example\\.com$",
  "ticket_pattern": "^CHG[0-9]{7}$",
  "require_approval": true,
  "approval_teams": ["security", "platform"]
}
```

## 📖 Usage

### Automated Daily Checks

The solution automatically runs daily at 2 AM UTC to check and renew certificates:

```yaml
# Automatic schedule
on:
  schedule:
    - cron: '0 2 * * *'
```

### Manual Certificate Operations

#### Check Certificate Status

```bash
gh workflow run apigee-cert-management.yml \
  -f environment=dev \
  -f check_only=true
```

#### Force Certificate Renewal

```bash
gh workflow run apigee-cert-management.yml \
  -f environment=uat \
  -f keystore_name=uat-keystore \
  -f domain=app-uat.example.com \
  -f cert_type=lets-encrypt \
  -f force_renewal=true
```

#### Production Deployment

```bash
gh workflow run apigee-cert-prod-deployment.yml \
  -f keystore_name=prod-keystore \
  -f domain=api.example.com \
  -f cert_source=lets-encrypt \
  -f approval_ticket=CHG0001234
```

## 🔄 Workflows

### Non-Production Workflow

**File**: `.github/workflows/apigee-cert-management.yml`

- **Schedule**: Daily at 2 AM UTC
- **Environments**: Dev, Test, UAT
- **Features**:
  - Automatic certificate renewal
  - Self-signed and Let's Encrypt support
  - No approval required
  - Parallel environment processing

### Production Workflow

**File**: `.github/workflows/apigee-cert-prod-deployment.yml`

- **Trigger**: Manual only
- **Environment**: Production
- **Features**:
  - Approval gates
  - Change ticket validation
  - Backup before deployment
  - Rollback capability
  - Post-deployment validation

## 🔐 Security

### Security Features

- **Environment Isolation**: Separate credentials per environment
- **Secret Management**: GitHub encrypted secrets
- **Access Control**: Environment-based permissions
- **Audit Trail**: Complete operation logging
- **Approval Workflow**: Multi-stage approval for production

### Best Practices

1. **Rotate Secrets Quarterly**
   ```bash
   # Update SSO passcode
   gh secret set APIGEE_SSO_PASSCODE_PROD --env production
   ```

2. **Use Strong Passwords**
   - Minimum 16 characters
   - Mix of uppercase, lowercase, numbers, symbols

3. **Regular Security Audits**
   - Review access logs monthly
   - Validate approval workflows
   - Check certificate configurations

## 🔧 Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| **Authentication Failed** | 1. Verify SSO passcode is current<br>2. Check API endpoint URLs<br>3. Ensure network connectivity |
| **Certificate Upload Failed** | 1. Verify keystore exists<br>2. Check PKCS12 password<br>3. Validate certificate format |
| **Workflow Timeout** | 1. Check Apigee API response time<br>2. Review retry configuration<br>3. Increase timeout limits |
| **Let's Encrypt Failed** | 1. Verify DNS credentials<br>2. Check rate limits<br>3. Validate domain ownership |

### Debug Mode

Enable debug logging:

```yaml
env:
  LOG_LEVEL: DEBUG
  CURL_VERBOSE: true
```

### Manual Recovery

```bash
# Restore from backup
./scripts/manual_recovery.sh \
  --env production \
  --keystore prod-keystore \
  --backup-file backups/cert_backup_20240115.json
```

## 🤝 Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

### Development Setup

```bash
# Install development dependencies
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Run tests
./tests/run_all_tests.sh

# Lint scripts
shellcheck scripts/*.sh
```

## 📞 Support

### Getting Help

- **Documentation**: Check the [docs](./docs) directory
- **Issues**: Create a GitHub issue
- **Slack**: Join #apigee-cert-management
- **Email**: platform-team@example.com

### Service Level Agreement

| Priority | Response Time | Resolution Time |
|----------|---------------|-----------------|
| P1 (Production Down) | 15 minutes | 4 hours |
| P2 (Production Risk) | 1 hour | 8 hours |
| P3 (Non-Production) | 4 hours | 2 days |
| P4 (Enhancement) | 2 days | Best effort |

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

**Maintained by**: Platform Engineering Team  
**Last Updated**: January 2024  
**Version**: 1.0.0
