# Deployment Plan - Rollout Strategy

**Purpose:** Safe, controlled deployment of migrated system
**Deployment Type:** Blue-Green with feature flags
**Risk Level:** HIGH - Production system
**Timeline:** Weeks 21-22 (after migration complete)

---

## Executive Summary

**Deployment Strategy:**
1. Set up parallel production environment
2. Deploy to new environment
3. Run smoke tests
4. Switch traffic gradually (5% → 25% → 50% → 100%)
5. Monitor metrics
6. Rollback plan if issues

**Timeline:**
- Week 21: Preparation + Staging deployment
- Week 22: Production rollout (5 days)

---

## Pre-Deployment Checklist

### Infrastructure Preparation (Week 21, Days 1-2)

**Server Requirements:**
- [ ] PHP 8.3 installed on all servers
- [ ] MySQL 8.0+ configured
- [ ] Redis 6.0+ for cache
- [ ] Nginx 1.24+ configured
- [ ] SSL certificates valid
- [ ] Load balancer configured
- [ ] CDN configured (optional)
- [ ] Backup systems tested

**Database Preparation:**
- [ ] Full production backup taken
- [ ] Backup integrity verified
- [ ] Test restore performed
- [ ] UTF-8 conversion validated
- [ ] Indexes rebuilt
- [ ] Query cache warmed

**Code Deployment:**
- [ ] Repository tagged as release
- [ ] Docker images built
- [ ] Artifacts uploaded
- [ ] Environment variables configured
- [ ] Secrets loaded
- [ ] Composer dependencies installed

---

## Phase 1: Staging Deployment (Week 21, Days 3-4)

### Step 1: Deploy to Staging

**Server List:**
```
staging-1.example.com
staging-2.example.com
```

**Deployment Script:**

```bash
#!/bin/bash
# scripts/deploy-staging.sh

set -e

echo "=== Deploying to Staging ==="

# Configuration
STAGING_HOSTS=("staging-1.example.com" "staging-2.example.com")
REPO_URL="git@github.com:yourorg/modern-php-discuz.git"
BRANCH="main"
RELEASE_TAG="v1.0.0"

# Pull latest code
for host in "${STAGING_HOSTS[@]}"; do
    echo "Deploying to $host..."

    ssh $host << ENDSSH
        cd /var/www/discuz

        # Backup current version
        cp -r current current.backup.$(date +%Y%m%d_%H%M%S)

        # Pull new code
        git fetch --all
        git checkout $BRANCH
        git pull origin $BRANCH
        git checkout $RELEASE_TAG

        # Install dependencies
        composer install --no-dev --optimize-autoloader

        # Run migrations
        php artisan migrate --force

        # Clear cache
        php artisan cache:clear
        php artisan config:clear
        php artisan view:clear

        # Warm cache
        php artisan cache:warm

        # Restart PHP-FPM
        sudo systemctl reload php8.3-fpm

        echo "✅ Deployed to $host"
ENDSSH
done

echo "=== Staging Deployment Complete ==="
```

---

### Step 2: Smoke Tests

**Automated Smoke Tests:**

```php
<?php
declare(strict_types=1);

use GuzzleHttp\Client;

class SmokeTest
{
    private Client $client;
    private array $errors = [];

    public function __construct()
    {
        $this->client = new Client(['base_uri' => 'https://staging.example.com']);
    }

    public function runAll(): void
    {
        $this->testHomepage();
        $this->testLogin();
        $this->testForum();
        $this->testThread();

        if (empty($this->errors)) {
            echo "✅ All smoke tests passed!\n";
        } else {
            echo "❌ Smoke tests failed:\n";
            foreach ($this->errors as $error) {
                echo "  - $error\n";
            }
            exit(1);
        }
    }

    private function testHomepage(): void
    {
        try {
            $response = $this->client->get('/');
            $this->assertEquals(200, $response->getStatusCode());
            echo "✅ Homepage loads\n";
        } catch (\Exception $e) {
            $this->errors[] = "Homepage failed: " . $e->getMessage();
        }
    }

    private function testLogin(): void
    {
        try {
            $response = $this->client->post('/login', [
                'form_params' => [
                    'username' => 'testuser',
                    'password' => 'password',
                ]
            ]);
            $this->assertEquals(302, $response->getStatusCode()); // Redirect
            echo "✅ Login works\n";
        } catch (\Exception $e) {
            $this->errors[] = "Login failed: " . $e->getMessage();
        }
    }

    // ... more tests ...
}
```

**Manual Smoke Tests:**
- [ ] Homepage loads
- [ ] Can login
- [ ] Can view forum
- [ ] Can read thread
- [ ] Can post reply
- [ ] Search works
- [ ] Admin panel accessible

---

### Step 3: Load Testing

**Apache Bench (ab):**

```bash
# Homepage load test
ab -n 1000 -c 10 https://staging.example.com/

# Forum page test
ab -n 1000 -c 10 https://staging.example.com/forum-1

# Thread page test
ab -n 1000 -c 10 https://staging.example.com/thread-1
```

**Acceptance Criteria:**
- 95%+ requests succeed
- Average response time <500ms
- No errors in logs

---

## Phase 2: Production Deployment (Week 22)

### Day 1: Initial Setup

**Create Blue-Green Environments:**

```
Current (Blue):
  - production-1.example.com (old system)
  - production-2.example.com (old system)

New (Green):
  - production-new-1.example.com (new system)
  - production-new-2.example.com (new system)
```

**Deploy to Green:**

```bash
#!/bin/bash
# scripts/deploy-green.sh

GREEN_HOSTS=("production-new-1.example.com" "production-new-2.example.com")

for host in "${GREEN_HOSTS[@]}"; do
    echo "Deploying to $host..."
    # Same deployment script as staging
    # ...
done

echo "✅ Green environment deployed"
```

---

### Day 2: 5% Traffic

**Load Balancer Configuration:**

```nginx
# nginx.conf

upstream discuz_blue {
    server production-1.example.com weight=95;
    server production-2.example.com weight=95;
}

upstream discuz_green {
    server production-new-1.example.com weight=5;
    server production-new-2.example.com weight=5;
}

server {
    listen 443 ssl;
    server_name forum.example.com;

    location / {
        proxy_pass http://discuz_mixed;
    }
}

# Split traffic 95% blue, 5% green
split_clients $remote_addr $backend {
    95% "discuz_blue";
    5% "discuz_green";
}

upstream discuz_mixed {
    server backend1.example.com; # Uses split_clients result
}
```

**Monitor Metrics:**

```bash
# Monitor error rate
watch -n 5 'tail -100 /var/log/nginx/error.log | grep ERROR | wc -l'

# Monitor response time
watch -n 5 'curl -w "%{time_total}\n" -o /dev/null -s https://forum.example.com/'

# Monitor database
watch -n 5 'mysql -e "SHOW PROCESSLIST" | wc -l'
```

**Rollback Decision Matrix:**

| Metric | Threshold | Action |
|--------|-----------|--------|
| Error Rate | >5% | Rollback |
| Response Time | >2000ms | Investigate |
| Database Load | >80% | Investigate |
| Server Load | >10 | Investigate |

**If rollback needed:**
```bash
# Revert load balancer to 100% blue
nginx -s reload

# Stop green servers
ssh production-new-1 'systemctl stop php8.3-fpm'
ssh production-new-2 'systemctl stop php8.3-fpm'
```

---

### Day 3: 25% Traffic

**Increase Traffic to Green:**

```nginx
split_clients $remote_addr $backend {
    75% "discuz_blue";
    25% "discuz_green";
}
```

**Monitoring Focus:**
- User-reported issues
- Transaction volume
- Revenue (if applicable)
- Search queries
- Post volume

---

### Day 4: 50% Traffic

**Increase Traffic to Green:**

```nginx
split_clients $remote_addr $backend {
    50% "discuz_blue";
    50% "discuz_green";
}
```

**Validation:**
- Compare metrics between blue/green
- Feature parity check
- User feedback survey
- Performance comparison

---

### Day 5: 100% Traffic (Cutover)

**Final Cutover:**

```nginx
upstream discuz {
    server production-new-1.example.com;
    server production-new-2.example.com;
    # Remove old servers
}

server {
    location / {
        proxy_pass http://discuz;
    }
}
```

**Post-Deployment Tasks:**
- [ ] Blue servers stopped
- [ ] Blue servers backed up
- [ ] DNS updated (if needed)
- [ ] Monitoring alerts updated
- [ ] Documentation updated
- [ ] Team notified

---

## Phase 3: Post-Deployment (Week 22, Days 6-7)

### Monitoring Dashboard

**Key Metrics to Watch:**

```yaml
# config/grafana-dashboard.yml

dashboard:
  title: Discuz Production Metrics
  panels:
    - title: Request Rate
      metrics:
        - requests_per_second
      targets:
        - alert: <10/min
          severity: warning

    - title: Error Rate
      metrics:
        - error_percentage
      targets:
        - alert: >1%
          severity: critical

    - title: Response Time
      metrics:
        - p50_response_time
        - p95_response_time
        - p99_response_time
      targets:
        - alert: p95 > 1000ms
          severity: warning

    - title: Database Connections
      metrics:
        - active_connections
      targets:
        - alert: >80% max
          severity: warning

    - title: Cache Hit Rate
      metrics:
        - cache_hit_percentage
      targets:
        - alert: <80%
          severity: warning

    - title: Active Users
      metrics:
        - online_users
      targets:
        - alert: <50% baseline
          severity: warning
```

---

### Rollback Procedures

**Emergency Rollback (< 1 hour):**

```bash
#!/bin/bash
# scripts/emergency-rollback.sh

echo "=== EMERGENCY ROLLBACK ==="

# 1. Switch load balancer to 100% blue
cat > /etc/nginx/conf.d/discuz.conf << EOF
upstream discuz {
    server production-1.example.com;
    server production-2.example.com;
}
EOF

nginx -s reload

# 2. Stop green servers
for host in production-new-1 production-new-2; do
    ssh $host 'systemctl stop php8.3-fpm'
done

# 3. Notify team
curl -X POST $SLACK_WEBHOOK -d '{"text":"Emergency rollback performed!"}'

echo "=== ROLLBACK COMPLETE ==="
```

**Data Rollback (if needed):**

```bash
#!/bin/bash
# scripts/rollback-database.sh

echo "Rolling back database..."

# Stop application
ssh production-new-1 'systemctl stop php8.3-fpm'

# Restore database backup
zcat backup_$(date +%Y%m%d).sql.gz | mysql -u root -p discuz

# Verify restore
mysql -u root -p discuz -e "SELECT COUNT(*) FROM cdb_members"

# Start application
ssh production-new-1 'systemctl start php8.3-fpm'

echo "Database rolled back"
```

---

### Post-Deployment Validation

**User Acceptance Test:**

```
URL: https://forum.example.com

Checklist:
- [ ] Homepage loads in <2 seconds
- [ ] Can login with existing credentials
- [ ] Chinese characters display correctly
- [ ] Can view forums
- [ ] Can read threads
- [ ] Can post new thread
- [ ] Can reply to thread
- [ ] Search works
- [ ] Profile page loads
- [ ] Settings save
- [ ] Uploads work
- [ ] Private messages work
- [ ] Admin panel accessible
- [ ] Moderation works
```

**Performance Validation:**

```
Test: Apache Bench (1000 requests, 10 concurrent)

Homepage:
  Requests per second: >100
  Time per request: <100ms
  Failed requests: 0

Forum Page:
  Requests per second: >80
  Time per request: <125ms
  Failed requests: 0

Thread Page:
  Requests per second: >60
  Time per request: <166ms
  Failed requests: 0
```

---

## Phase 4: Cleanup (Week 22, Day 7)

### Remove Old Servers

```bash
# After 7 days of stable operation

# Stop old servers
systemctl stop php7.4-fpm  # Old PHP version

# Remove from load balancer
# Update nginx config to remove blue servers

# Decommission servers
# (Keep backups for 30 days)
```

### Documentation Updates

**Update Documentation:**
- [ ] Architecture diagrams updated
- [ ] Deployment guide updated
- [ ] Runbooks updated
- [ ] Troubleshooting guides updated
- [ ] API documentation updated

---

## Success Criteria

✅ 100% traffic on new system
✅ <1% error rate
✅ <500ms average response time
✅ All user journeys working
✅ No data loss
✅ No security issues
✅ 7 days stable operation

---

## Communication Plan

**Pre-Deployment:**
- Announcement: 1 week before
- Email to all users
- Banner on forum
- Scheduled maintenance window

**During Deployment:**
- Status page updates
- Slack notifications
- Email to stakeholders
- Progress updates

**Post-Deployment:**
- Success announcement
- Feature highlights
- Known issues list
- Support contact info

---

## Timeline Summary

**Week 21: Preparation**
- Days 1-2: Infrastructure setup
- Days 3-4: Staging deployment & testing
- Day 5: Load testing & fixes

**Week 22: Production Rollout**
- Day 1: Deploy to green (0% traffic)
- Day 2: 5% traffic
- Day 3: 25% traffic
- Day 4: 50% traffic
- Day 5: 100% cutover
- Day 6-7: Monitoring & validation

**Total:** 12 days from prep to complete rollout

---

## Emergency Contacts

| Role | Name | Phone | Email |
|------|------|-------|-------|
| Tech Lead | | | |
| DevOps | | | |
| DBA | | | |
| Product Owner | | | |

---

**Document Status:** ✅ Complete
**All Documents Complete!** ✅
