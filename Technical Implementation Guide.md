# Apigee Certificate Management Solution
## Design Document - Part 3: Technical Implementation Guide

### Table of Contents
1. [Prerequisites and Dependencies](#prerequisites-and-dependencies)
2. [Repository Structure](#repository-structure)
3. [Configuration Setup](#configuration-setup)
4. [Secrets Management](#secrets-management)
5. [Script Implementation Details](#script-implementation-details)
6. [GitHub Actions Setup](#github-actions-setup)
7. [Testing Strategy](#testing-strategy)

---

## 1. Prerequisites and Dependencies

### 1.1 System Requirements

| Component | Requirement | Purpose |
|-----------|-------------|---------|
| GitHub | Enterprise or Pro account | GitHub Actions, Environments, Secrets |
| Apigee Edge | Enterprise account with API access | Target platform for certificates |
| Ubuntu Runner | 20.04 or latest | GitHub Actions execution environment |
| OpenSSL | 1.1.1 or higher | Certificate generation and conversion |
| Certbot | Latest version | Let's Encrypt certificate generation |
| jq | 1.6 or higher | JSON parsing in scripts |
| curl | 7.68 or higher | API interactions |

### 1.2 External Service Requirements

| Service | Requirement | Purpose |
|---------|-------------|---------|
| Let's Encrypt | Account with email | Free SSL certificates for non-prod |
| DigiCert | API access (optional) | Production-grade certificates |
| DNS Provider | API access (Route53/Cloudflare) | DNS challenge for Let's Encrypt |
| Slack | Webhook URL | Notifications |
| Email Server | SMTP access (optional) | Email notifications |

### 1.3 Access Requirements

```yaml
Apigee Edge Access:
  - SSO passcode per environment
  - API access to organization
  - Permission to manage keystores and certificates

GitHub Repository:
  - Admin access for secrets configuration
  - Write access for workflow files
  - Environment configuration permissions

DNS Provider (for Let's Encrypt):
  - API credentials for DNS challenge
  - Permission to create TXT records
```

## 2. Repository Structure

### 2.1 Complete Repository Layout

```
repository-root/
├── .github/
│   └── workflows/
│       ├── apigee-cert-management.yml          # Non-production workflow
│       └── apigee-cert-prod-deployment.yml     # Production workflow
│
├── scripts/
│   ├── apigee_auth.sh                          # Authentication script
│   ├── apigee_keystore.sh                      # Keystore management
│   └── apigee_certs.sh                         # Certificate operations
│
├── config/
│   ├── apigee-config.json                      # Global configuration
│   └── environments/
│       ├── dev.json                             # Dev environment config
│       ├── test.json                            # Test environment config
│       ├── uat.json                             # UAT environment config
│       └── prod.json                            # Production config
│
├── certs/                                       # Optional: existing certificates
│   ├── dev/
│   ├── test/
│   ├── uat/
│   └── prod/
│
├── docs/
│   ├── setup-guide.md                          # Setup instructions
│   ├── troubleshooting.md                      # Common issues
│   └── runbook.md                              # Operational procedures
│
└── README.md                                    # Project overview
```

### 2.2 File Permissions

```bash
# Set correct permissions for scripts
chmod +x scripts/*.sh

# Ensure config files are readable
chmod 644 config/*.json
chmod 644 config/environments/*.json

# Protect any existing certificates
chmod 600 certs/*/*.pem
```

## 3. Configuration Setup

### 3.1 Global Configuration (config/apigee-config.json)

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
      "recipients": ["team@example.com"],
      "smtp_secret": "SMTP_CREDENTIALS"
    },
    "teams": {
      "enabled": false,
      "webhook_url_secret": "TEAMS_WEBHOOK_URL"
    }
  },
  "workflow_settings": {
    "max_retry_attempts": 3,
    "retry_delay_seconds": 30,
    "artifact_retention_days": {
      "summaries": 7,
      "backups": 90,
      "certificates": 7
    }
  }
}
```

### 3.2 Environment Configuration Template

```json
{
  "keystore_name": "{env}-app-keystore",
  "domain": "app-{env}.example.com",
  "alias_name": "app-{env}-cert",
  "cert_type": "self-signed|lets-encrypt|digicert|existing",
  "cert_validity_days": 365,
  "key_size": 2048,
  "additional_domains": [],
  "dns_plugin": "route53|cloudflare",
  "dns_credentials_secret": "LETS_ENCRYPT_DNS_CREDENTIALS_{ENV}",
  "notification_overrides": {
    "slack_channel": "#cert-alerts-{env}",
    "email_recipients": ["{env}-team@example.com"]
  }
}
```

### 3.3 Production-Specific Configuration

```json
{
  "keystore_name": "prod-app-keystore",
  "domain": "api.example.com",
  "alias_name": "api-prod-cert",
  "cert_type": "digicert",
  "additional_domains": ["www.api.example.com", "api-v2.example.com"],
  "domain_pattern": "^(api\\.|www\\.api\\.|api-v2\\.)?example\\.com$",
  "ticket_pattern": "^CHG[0-9]{7}$",
  "require_approval": true,
  "approval_teams": ["security", "platform", "devops"],
  "approval_timeout_minutes": 60,
  "backup_retention_days": 180,
  "rollback_enabled": true,
  "post_deployment_tests": {
    "enabled": true,
    "endpoints": [
      "https://api.example.com/health",
      "https://www.api.example.com/health"
    ],
    "timeout_seconds": 30
  }
}
```

## 4. Secrets Management

### 4.1 Required GitHub Secrets

```yaml
# Core Apigee Secrets
APIGEE_ORG: "your-organization-name"
APIGEE_SSO_URL: "https://your-sso.login.apigee.com"
APIGEE_API_URL: "https://api.enterprise.apigee.com/v1"

# Environment-Specific SSO Passcodes
APIGEE_SSO_PASSCODE_DEV: "dev-passcode"
APIGEE_SSO_PASSCODE_TEST: "test-passcode"
APIGEE_SSO_PASSCODE_UAT: "uat-passcode"
APIGEE_PROD_SSO_PASSCODE: "prod-passcode"

# Keystore Passwords (per environment)
KEYSTORE_PASSWORD_DEV: "secure-dev-password"
KEYSTORE_PASSWORD_TEST: "secure-test-password"
KEYSTORE_PASSWORD_UAT: "secure-uat-password"
PROD_KEYSTORE_PASSWORD: "secure-prod-password"

# Certificate Generation Secrets
LETS_ENCRYPT_EMAIL: "certs@example.com"
LETS_ENCRYPT_PROD_EMAIL: "prod-certs@example.com"
LETS_ENCRYPT_DNS_CREDENTIALS: "dns-api-credentials"
LETS_ENCRYPT_DNS_CREDENTIALS_PROD: "prod-dns-credentials"
LETS_ENCRYPT_DNS_PLUGIN_PROD: "route53"

# DigiCert Integration (Production)
DIGICERT_API_KEY: "digicert-api-key"
DIGICERT_ORDER_ID_PROD: "order-id"
DIGICERT_PRIVATE_KEY_PROD: "private-key-content"

# Certificate Metadata
CERT_COUNTRY: "US"
CERT_STATE: "California"
CERT_CITY: "San Francisco"
CERT_ORG: "Example Corporation"

# Production Specific
PROD_DOMAIN_PATTERN: "^(api\\.|www\\.api\\.)?example\\.com$"
TICKET_PATTERN: "^CHG[0-9]{7}$"

# Notification Secrets
SLACK_WEBHOOK_URL: "https://hooks.slack.com/services/..."
TEAMS_WEBHOOK_URL: "https://outlook.office.com/webhook/..."
SMTP_CREDENTIALS: "smtp-username:smtp-password"
```

### 4.2 GitHub Environment Configuration

```yaml
Environments:
  development:
    - No protection rules
    - Secrets: DEV specific secrets
    
  test:
    - No protection rules
    - Secrets: TEST specific secrets
    
  uat:
    - Optional: Require approval from QA team
    - Secrets: UAT specific secrets
    
  production-backup:
    - Required reviewers: 1 from security team
    - Secrets: Read-only production access
    
  production-cert-generation:
    - Required reviewers: 1 from security team
    - Secrets: Certificate generation secrets
    
  production:
    - Required reviewers: 2 from [security, platform] teams
    - Wait timer: 5 minutes
    - Deployment branches: main only
    - Secrets: Full production access
```

## 5. Script Implementation Details

### 5.1 Authentication Script Enhancement

```bash
#!/bin/bash
# apigee_auth.sh - Enhanced with retry logic

# Add retry mechanism
retry_auth() {
    local max_attempts=3
    local attempt=1
    local delay=5
    
    while [ $attempt -le $max_attempts ]; do
        echo "Authentication attempt $attempt of $max_attempts..." >&2
        
        local token=$(exchange_passcode_for_token "$1")
        if [ $? -eq 0 ] && [ -n "$token" ]; then
            echo "$token"
            return 0
        fi
        
        if [ $attempt -lt $max_attempts ]; then
            echo "Retrying in $delay seconds..." >&2
            sleep $delay
            delay=$((delay * 2))  # Exponential backoff
        fi
        
        attempt=$((attempt + 1))
    done
    
    echo "Error: Authentication failed after $max_attempts attempts." >&2
    return 1
}
```

### 5.2 Certificate Validation Enhancement

```bash
#!/bin/bash
# apigee_certs.sh - Enhanced validation

# Enhanced certificate validation
validate_certificate_comprehensive() {
    local cert_file="$1"
    local expected_domain="$2"
    local min_days_valid="${3:-30}"
    
    echo "=== Comprehensive Certificate Validation ===" >&2
    
    # Check certificate exists and is readable
    if [ ! -r "$cert_file" ]; then
        echo "Error: Cannot read certificate file: $cert_file" >&2
        return 1
    fi
    
    # Extract certificate details
    local subject=$(openssl x509 -noout -subject -in "$cert_file" 2>/dev/null)
    local issuer=$(openssl x509 -noout -issuer -in "$cert_file" 2>/dev/null)
    local start_date=$(openssl x509 -noout -startdate -in "$cert_file" 2>/dev/null | cut -d= -f2)
    local end_date=$(openssl x509 -noout -enddate -in "$cert_file" 2>/dev/null | cut -d= -f2)
    
    # Check certificate validity period
    local now_epoch=$(date +%s)
    local end_epoch=$(date -d "$end_date" +%s 2>/dev/null || date -j -f "%b %d %H:%M:%S %Y %Z" "$end_date" +%s 2>/dev/null)
    
    if [ -z "$end_epoch" ]; then
        echo "Error: Cannot parse certificate end date" >&2
        return 1
    fi
    
    local days_remaining=$(( (end_epoch - now_epoch) / 86400 ))
    
    echo "Subject: $subject" >&2
    echo "Issuer: $issuer" >&2
    echo "Valid from: $start_date" >&2
    echo "Valid until: $end_date" >&2
    echo "Days remaining: $days_remaining" >&2
    
    # Validate minimum days
    if [ $days_remaining -lt $min_days_valid ]; then
        echo "Error: Certificate expires in $days_remaining days (minimum required: $min_days_valid)" >&2
        return 1
    fi
    
    # Check domain match (CN and SANs)
    local cn=$(echo "$subject" | grep -oP 'CN\s*=\s*\K[^,/]+' || true)
    local sans=$(openssl x509 -noout -text -in "$cert_file" 2>/dev/null | grep -A1 "Subject Alternative Name" | tail -1 | tr ',' '\n' | grep "DNS:" | cut -d: -f2 | tr -d ' ')
    
    echo "Common Name: $cn" >&2
    echo "SANs: $(echo $sans | tr '\n' ', ')" >&2
    
    # Check if expected domain matches CN or any SAN
    local domain_matched=false
    
    if [[ "$cn" == "$expected_domain" ]] || [[ "$cn" == "*."* && "$expected_domain" =~ ${cn#\*.} ]]; then
        domain_matched=true
    else
        for san in $sans; do
            if [[ "$san" == "$expected_domain" ]] || [[ "$san" == "*."* && "$expected_domain" =~ ${san#\*.} ]]; then
                domain_matched=true
                break
            fi
        done
    fi
    
    if [ "$domain_matched" != "true" ]; then
        echo "Warning: Expected domain '$expected_domain' not found in certificate" >&2
    fi
    
    # Check key size
    local key_file="${cert_file%.pem}.key.pem"
    if [ -f "$key_file" ]; then
        local key_size=$(openssl rsa -in "$key_file" -text -noout 2>/dev/null | grep "Private-Key:" | grep -oP '\d+')
        echo "Key size: $key_size bits" >&2
        
        if [ "$key_size" -lt 2048 ]; then
            echo "Error: Key size $key_size is less than minimum 2048 bits" >&2
            return 1
        fi
    fi
    
    echo "=== Certificate validation passed ===" >&2
    return 0
}
```

### 5.3 Keystore Management Enhancement

```bash
#!/bin/bash
# apigee_keystore.sh - Enhanced with batch operations

# Batch check multiple aliases
batch_check_aliases() {
    local token="$1"
    local env="$2"
    local keystore_name="$3"
    shift 3
    local aliases=("$@")
    
    echo "=== Batch Checking Aliases ===" >&2
    local results=""
    
    for alias in "${aliases[@]}"; do
        echo "Checking alias: $alias" >&2
        local status=$(check_alias "$token" "$env" "$keystore_name" "$alias" 2>/dev/null)
        if [ $? -eq 0 ]; then
            results+="$alias:EXISTS;"
        else
            results+="$alias:NOT_FOUND;"
        fi
    done
    
    echo "$results"
}

# Enhanced certificate upload with pre-validation
upload_certificate_validated() {
    local token="$1"
    local env="$2"
    local keystore_name="$3"
    local alias_name="$4"
    local pkcs12_file="$5"
    local pkcs12_password="$6"
    local expected_domain="$7"
    
    echo "=== Validated Certificate Upload ===" >&2
    
    # Pre-upload validation
    echo "Validating PKCS12 file..." >&2
    openssl pkcs12 -info -in "$pkcs12_file" -passin "pass:$pkcs12_password" -noout
    if [ $? -ne 0 ]; then
        echo "Error: Invalid PKCS12 file or password" >&2
        return 1
    fi
    
    # Extract and validate certificate from PKCS12
    local temp_cert="/tmp/cert_${RANDOM}.pem"
    openssl pkcs12 -in "$pkcs12_file" -out "$temp_cert" -nokeys -passin "pass:$pkcs12_password" 2>/dev/null
    
    if validate_certificate_comprehensive "$temp_cert" "$expected_domain" 30; then
        echo "Certificate validation passed. Proceeding with upload..." >&2
        upload_certificate "$token" "$env" "$keystore_name" "$alias_name" "$pkcs12_file" "$pkcs12_password" "true"
        local result=$?
        rm -f "$temp_cert"
        return $result
    else
        echo "Error: Certificate validation failed. Upload aborted." >&2
        rm -f "$temp_cert"
        return 1
    fi
}
```

## 6. GitHub Actions Setup

### 6.1 Non-Production Workflow Configuration

```yaml
# .github/workflows/apigee-cert-management.yml
name: Apigee Certificate Management (Dev/Test/UAT)

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM UTC
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: ['dev', 'test', 'uat']
      keystore_name:
        description: 'Keystore name'
        required: false
      domain:
        description: 'Domain for certificate'
        required: false
      cert_type:
        description: 'Certificate type'
        required: true
        type: choice
        options: ['self-signed', 'lets-encrypt', 'existing']
        default: 'self-signed'
      check_only:
        description: 'Only check certificate status'
        required: false
        type: boolean
        default: false
      force_renewal:
        description: 'Force renewal regardless of expiry'
        required: false
        type: boolean
        default: false

env:
  CERT_RENEWAL_DAYS: 30
  LOG_LEVEL: INFO  # DEBUG for troubleshooting

# Add concurrency control
concurrency:
  group: cert-mgmt-${{ github.event.inputs.environment || 'scheduled' }}
  cancel-in-progress: false
```

### 6.2 Production Workflow Configuration

```yaml
# .github/workflows/apigee-cert-prod-deployment.yml
name: Apigee Certificate Production Deployment

on:
  workflow_dispatch:
    inputs:
      keystore_name:
        description: 'Apigee Keystore name'
        required: true
      domain:
        description: 'Primary domain'
        required: true
      alias_name:
        description: 'Certificate alias name'
        required: false
      cert_source:
        description: 'Certificate source'
        required: true
        type: choice
        options: ['lets-encrypt', 'digicert', 'existing-file']
      approval_ticket:
        description: 'Change approval ticket'
        required: true
      emergency_deployment:
        description: 'Emergency deployment (skip some validations)'
        required: false
        type: boolean
        default: false
      dry_run:
        description: 'Dry run (validate only, no deployment)'
        required: false
        type: boolean
        default: false

env:
  APIGEE_ENV: prod
  MAX_RETRY_ATTEMPTS: 3
  DEPLOYMENT_TIMEOUT: 3600  # 1 hour

# Strict concurrency for production
concurrency:
  group: cert-prod-deployment
  cancel-in-progress: false
```

### 6.3 Reusable Workflow Components

```yaml
# .github/workflows/reusable-cert-validation.yml
name: Reusable Certificate Validation

on:
  workflow_call:
    inputs:
      certificate_path:
        required: true
        type: string
      expected_domain:
        required: true
        type: string
      environment:
        required: true
        type: string
    secrets:
      APIGEE_ORG:
        required: true
      APIGEE_API_URL:
        required: true

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Validate Certificate
        run: |
          chmod +x ./scripts/apigee_certs.sh
          ./scripts/apigee_certs.sh validate "${{ inputs.certificate_path }}" \
            "${{ inputs.expected_domain }}"
```

## 7. Testing Strategy

### 7.1 Unit Tests for Scripts

```bash
#!/bin/bash
# tests/test_apigee_auth.sh

# Test authentication with valid passcode
test_valid_authentication() {
    local result=$(APIGEE_SSO_URL="$TEST_SSO_URL" \
                   APIGEE_API_URL="$TEST_API_URL" \
                   APIGEE_ORG="$TEST_ORG" \
                   ./scripts/apigee_auth.sh "$TEST_PASSCODE")
    
    if [ -n "$result" ] && [[ "$result" =~ ^[A-Za-z0-9._-]+$ ]]; then
        echo "✓ Valid authentication test passed"
        return 0
    else
        echo "✗ Valid authentication test failed"
        return 1
    fi
}

# Test authentication with invalid passcode
test_invalid_authentication() {
    local result=$(APIGEE_SSO_URL="$TEST_SSO_URL" \
                   APIGEE_API_URL="$TEST_API_URL" \
                   APIGEE_ORG="$TEST_ORG" \
                   ./scripts/apigee_auth.sh "invalid-passcode" 2>&1)
    
    if [[ "$result" =~ "Error" ]]; then
        echo "✓ Invalid authentication test passed"
        return 0
    else
        echo "✗ Invalid authentication test failed"
        return 1
    fi
}
```

### 7.2 Integration Test Workflow

```yaml
# .github/workflows/integration-tests.yml
name: Integration Tests

on:
  pull_request:
    paths:
      - 'scripts/**'
      - '.github/workflows/**'
      - 'config/**'

jobs:
  test-scripts:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup test environment
        run: |
          sudo apt-get update
          sudo apt-get install -y jq openssl curl
      
      - name: Run script tests
        run: |
          chmod +x tests/*.sh
          ./tests/run_all_tests.sh
      
      - name: Test configuration parsing
        run: |
          # Validate JSON configs
          for config in config/*.json config/environments/*.json; do
            echo "Validating $config"
            jq empty "$config" || exit 1
          done
```

### 7.3 End-to-End Test Scenarios

```yaml
Test Scenarios:
  1. Certificate Expiry Detection:
     - Deploy cert with 1 day validity
     - Run check workflow
     - Verify renewal triggered
  
  2. Keystore Creation:
     - Target non-existent keystore
     - Verify automatic creation
     - Confirm certificate upload
  
  3. Production Approval Flow:
     - Trigger production deployment
     - Verify approval gate activation
     - Test timeout behavior
  
  4. Rollback Scenario:
     - Simulate failed deployment
     - Verify backup restoration
     - Confirm service recovery
  
  5. Multi-Domain Certificate:
     - Configure additional domains
     - Generate certificate
     - Verify all domains included
```

### 7.4 Monitoring and Alerting Setup

```yaml
# monitoring/alerts.yml
Alerts:
  - name: certificate_expiry_warning
    condition: days_until_expiry < 7
    severity: warning
    channels: [slack, email]
    
  - name: certificate_expired
    condition: certificate_expired == true
    severity: critical
    channels: [slack, email, pagerduty]
    
  - name: workflow_failure
    condition: workflow_status == failed
    severity: error
    channels: [slack, email]
    
  - name: authentication_failure
    condition: auth_failures > 3
    severity: error
    channels: [slack, email]

Metrics:
  - certificate_days_remaining
  - workflow_execution_time
  - certificate_renewal_count
  - api_call_count
  - error_count_by_type
```

## 8. Troubleshooting Guide

### 8.1 Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| Authentication Failure | "401 Unauthorized" errors | 1. Verify SSO passcode is current<br>2. Check token expiry<br>3. Validate API endpoints |
| Certificate Upload Fails | "400 Bad Request" on upload | 1. Verify PKCS12 format<br>2. Check password correctness<br>3. Ensure keystore exists |
| Let's Encrypt Failure | DNS challenge fails | 1. Verify DNS credentials<br>2. Check domain ownership<br>3. Review rate limits |
| Workflow Timeout | Job exceeds time limit | 1. Increase timeout value<br>2. Check for API slowness<br>3. Review retry logic |

### 8.2 Debug Mode Activation

```bash
# Enable debug mode in workflows
env:
  LOG_LEVEL: DEBUG
  CURL_VERBOSE: true
  OPENSSL_DEBUG: true

# Add debug output to scripts
set -x  # Enable command tracing
set -v  # Enable verbose mode
```

### 8.3 Recovery Procedures

```bash
# Manual certificate recovery
./scripts/manual_recovery.sh \
  --environment prod \
  --keystore "prod-keystore" \
  --backup-file "backup.json" \
  --force

# Force sync from Apigee
./scripts/sync_certificates.sh \
  --environment all \
  --output-dir ./backups/sync_$(date +%Y%m%d)
```
