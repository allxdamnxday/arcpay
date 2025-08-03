# Maintenance Guide

## Overview

This guide provides comprehensive maintenance procedures for the ArcPay Ironworkers Payroll System. Regular maintenance ensures system reliability, security, and compliance with changing regulations.

## Annual Maintenance Calendar

### January
- **Federal Tax Table Updates**
  - Download new IRS Publication 15-T
  - Update federal tax brackets and rates
  - Test with IRS provided examples
  - Deploy updates before first payroll

- **State Tax Updates**
  - Download California DE 44
  - Update state tax tables
  - Update SDI/UI rates
  - Verify low-income exemptions

### February
- **W-2 Processing**
  - Generate and distribute W-2s
  - Submit to SSA
  - Archive prior year data

### March
- **Q1 Maintenance**
  - Database optimization
  - Archive old audit logs
  - Security audit

### April
- **Quarterly Tax Filings**
  - Generate 941 reports
  - Generate DE 9 reports
  - Reconcile YTD amounts

### June
- **Mid-Year Updates**
  - Union wage rate updates (if applicable)
  - Zone boundary reviews
  - Fringe benefit rate changes

### July
- **Q2 Maintenance**
  - Performance analysis
  - Database integrity checks
  - Backup system verification

### September
- **Q3 Maintenance**
  - Security updates
  - Compliance review
  - Disaster recovery test

### October
- **Year-End Preparation**
  - Review tax law changes
  - Prepare for W-2 updates
  - Test year-end procedures

### December
- **Year-End Processing**
  - Final tax table updates
  - Archive current year data
  - Prepare for new year

## Tax Table Update Procedures

### 1. Federal Tax Tables

#### 1.1 Download Source Documents
```bash
# Create backup of current tables
npm run backup:tax-tables

# Download IRS Publication 15-T
wget https://www.irs.gov/pub/irs-pdf/p15t.pdf -O /tmp/p15t_2025.pdf
```

#### 1.2 Parse and Import
```typescript
// src/scripts/import-federal-tax.ts
import { FederalTaxImporter } from '@main/services/tax/FederalTaxImporter';

async function importFederalTaxTables() {
  const importer = new FederalTaxImporter();
  
  // Parse PDF
  const tables = await importer.parsePDF('/tmp/p15t_2025.pdf');
  
  // Validate structure
  await importer.validateTables(tables);
  
  // Import to database
  await importer.importTables(tables, 2025);
  
  // Run validation tests
  await importer.runValidationTests();
}
```

#### 1.3 Validation Tests
```bash
# Run federal tax validation
npm run test:federal-tax -- --year=2025

# Compare with IRS examples
npm run validate:federal-examples
```

### 2. California Tax Tables

#### 2.1 Download and Parse
```bash
# Download DE 44
wget https://edd.ca.gov/pdf_pub_ctr/de44.pdf -O /tmp/de44_2025.pdf

# Run California importer
npm run import:ca-tax -- --file=/tmp/de44_2025.pdf --year=2025
```

#### 2.2 Update Configuration
```json
// config/tax-rates.json
{
  "california": {
    "2025": {
      "sdi_rate": 0.009,
      "sdi_wage_limit": 153164,
      "sui_rate": 0.031,
      "ett_rate": 0.001,
      "low_income_exemptions": {
        "single": {
          "weekly": 342,
          "biweekly": 683,
          "monthly": 1481
        }
      }
    }
  }
}
```

## Database Maintenance

### 1. Regular Optimization

#### 1.1 Weekly Tasks
```sql
-- Analyze tables for query optimization
ANALYZE employees;
ANALYZE time_cards;
ANALYZE payroll_calculations;

-- Update statistics
UPDATE STATISTICS;
```

#### 1.2 Monthly Tasks
```sql
-- Rebuild indexes
REINDEX TABLE employees;
REINDEX TABLE time_cards;
REINDEX TABLE payroll_calculations;

-- Vacuum database (SQLite)
VACUUM;

-- Check integrity
PRAGMA integrity_check;
```

### 2. Archive Procedures

#### 2.1 Audit Log Archival
```typescript
// src/scripts/archive-audit-logs.ts
async function archiveAuditLogs() {
  const cutoffDate = subMonths(new Date(), 3);
  
  // Export old logs
  const oldLogs = await db.query(
    'SELECT * FROM audit_log WHERE timestamp < ?',
    [cutoffDate]
  );
  
  // Write to archive
  await writeToArchive('audit_logs', oldLogs);
  
  // Delete from active database
  await db.query(
    'DELETE FROM audit_log WHERE timestamp < ?',
    [cutoffDate]
  );
}
```

#### 2.2 Timecard Archival
```bash
# Archive timecards older than 7 years
npm run archive:timecards -- --years=7

# Verify archive integrity
npm run verify:archive -- --type=timecards
```

### 3. Performance Monitoring

#### 3.1 Query Performance
```typescript
// Monitor slow queries
interface QueryMetrics {
  query: string;
  avgDuration: number;
  maxDuration: number;
  count: number;
}

async function analyzeQueryPerformance(): Promise<QueryMetrics[]> {
  const logs = await getQueryLogs();
  
  return logs
    .groupBy('query')
    .map(group => ({
      query: group.key,
      avgDuration: avg(group.values.map(v => v.duration)),
      maxDuration: max(group.values.map(v => v.duration)),
      count: group.values.length
    }))
    .filter(m => m.avgDuration > 100) // Over 100ms
    .sort((a, b) => b.avgDuration - a.avgDuration);
}
```

#### 3.2 Database Size Management
```bash
# Check database size
sqlite3 payroll.db "SELECT page_count * page_size as size FROM pragma_page_count(), pragma_page_size();"

# Check table sizes
sqlite3 payroll.db ".dbinfo"

# Identify large tables
npm run analyze:db-size
```

## Backup Procedures

### 1. Automated Backups

#### 1.1 Daily Backup Script
```typescript
// src/scripts/daily-backup.ts
export async function performDailyBackup() {
  const timestamp = format(new Date(), 'yyyy-MM-dd-HHmmss');
  const backupPath = path.join(BACKUP_DIR, `payroll_${timestamp}.db`);
  
  try {
    // Create backup
    await db.backup(backupPath);
    
    // Encrypt backup
    await encryptFile(backupPath, BACKUP_ENCRYPTION_KEY);
    
    // Upload to cloud storage
    await uploadToS3(backupPath + '.enc');
    
    // Verify backup
    await verifyBackup(backupPath + '.enc');
    
    // Clean old backups (keep 30 days)
    await cleanOldBackups(30);
    
    // Log success
    await auditLog('BACKUP_SUCCESS', { path: backupPath });
  } catch (error) {
    await alertAdministrators('Backup failed', error);
    throw error;
  }
}
```

#### 1.2 Backup Verification
```bash
# Verify latest backup
npm run verify:backup -- --latest

# Test restore procedure
npm run test:restore -- --dry-run

# List all backups
npm run list:backups
```

### 2. Restore Procedures

#### 2.1 Full Restore
```bash
# Stop application
npm run stop

# Restore from backup
npm run restore:backup -- --date=2024-12-15 --verify

# Run integrity checks
npm run db:integrity-check

# Restart application
npm run start
```

#### 2.2 Partial Restore
```typescript
// Restore specific tables
async function restoreTable(tableName: string, backupDate: Date) {
  const backup = await loadBackup(backupDate);
  
  await db.transaction(async (trx) => {
    // Clear current data
    await trx.query(`DELETE FROM ${tableName}`);
    
    // Restore from backup
    const data = await backup.query(`SELECT * FROM ${tableName}`);
    await trx.batchInsert(tableName, data);
    
    // Verify counts
    const originalCount = await backup.count(tableName);
    const restoredCount = await trx.count(tableName);
    
    if (originalCount !== restoredCount) {
      throw new Error(`Restore verification failed for ${tableName}`);
    }
  });
}
```

## Security Updates

### 1. Dependency Updates

#### 1.1 Regular Updates
```bash
# Check for security vulnerabilities
npm audit

# Update dependencies
npm update

# Check for major updates
npm outdated

# Update specific security patches
npm audit fix
```

#### 1.2 Electron Updates
```bash
# Update Electron to latest stable
npm install electron@latest --save-dev

# Test application thoroughly
npm run test:all

# Update electron-builder
npm install electron-builder@latest --save-dev
```

### 2. Certificate Management

#### 2.1 Code Signing Certificates
```bash
# Check certificate expiration
openssl x509 -enddate -noout -in codesign.crt

# Renew certificate (90 days before expiration)
# Follow platform-specific procedures
```

#### 2.2 SSL Certificates
```bash
# Update SSL certificates for API endpoints
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365
```

## Performance Optimization

### 1. Database Optimization

#### 1.1 Index Analysis
```sql
-- Find missing indexes
SELECT 
  'CREATE INDEX idx_' || table_name || '_' || column_name || 
  ' ON ' || table_name || '(' || column_name || ');' as suggested_index
FROM query_stats
WHERE full_scan = 1
AND rows_examined > 1000;
```

#### 1.2 Query Optimization
```typescript
// Profile slow queries
class QueryProfiler {
  async profilePayrollCalculation(periodId: number) {
    const profiler = new Profiler();
    
    profiler.start('total');
    
    profiler.start('fetch_employees');
    const employees = await this.fetchEmployees(periodId);
    profiler.end('fetch_employees');
    
    profiler.start('fetch_timecards');
    const timecards = await this.fetchTimecards(periodId);
    profiler.end('fetch_timecards');
    
    profiler.start('calculations');
    const results = await this.performCalculations(employees, timecards);
    profiler.end('calculations');
    
    profiler.end('total');
    
    return profiler.getReport();
  }
}
```

### 2. Application Performance

#### 2.1 Memory Usage
```typescript
// Monitor memory usage
setInterval(() => {
  const usage = process.memoryUsage();
  
  if (usage.heapUsed > 500 * 1024 * 1024) { // 500MB
    logger.warn('High memory usage detected', {
      heapUsed: formatBytes(usage.heapUsed),
      heapTotal: formatBytes(usage.heapTotal),
      rss: formatBytes(usage.rss)
    });
  }
}, 60000); // Check every minute
```

#### 2.2 Cache Management
```typescript
// Clear caches periodically
class CacheManager {
  async performMaintenance() {
    // Clear expired entries
    await this.clearExpiredEntries();
    
    // Check cache size
    const size = await this.getCacheSize();
    if (size > MAX_CACHE_SIZE) {
      await this.evictLRU(size - MAX_CACHE_SIZE);
    }
    
    // Rebuild frequently accessed items
    await this.warmCache();
  }
}
```

## Monitoring and Alerts

### 1. System Health Checks

#### 1.1 Automated Health Checks
```typescript
// src/monitoring/health-check.ts
export class HealthChecker {
  async performHealthCheck(): Promise<HealthStatus> {
    const checks = await Promise.all([
      this.checkDatabase(),
      this.checkDiskSpace(),
      this.checkMemory(),
      this.checkBackups(),
      this.checkTaxTables(),
      this.checkEncryption()
    ]);
    
    const status = {
      healthy: checks.every(c => c.passed),
      timestamp: new Date(),
      checks
    };
    
    if (!status.healthy) {
      await this.sendAlert(status);
    }
    
    return status;
  }
  
  private async checkDatabase(): Promise<CheckResult> {
    try {
      await db.query('SELECT 1');
      const integrityCheck = await db.query('PRAGMA integrity_check');
      
      return {
        name: 'database',
        passed: integrityCheck[0].integrity_check === 'ok',
        message: 'Database integrity verified'
      };
    } catch (error) {
      return {
        name: 'database',
        passed: false,
        message: error.message
      };
    }
  }
}
```

#### 1.2 Alert Configuration
```json
// config/alerts.json
{
  "alerts": {
    "database_error": {
      "severity": "critical",
      "channels": ["email", "sms"],
      "recipients": ["admin@company.com", "+1234567890"]
    },
    "backup_failure": {
      "severity": "high",
      "channels": ["email"],
      "recipients": ["admin@company.com"]
    },
    "high_memory": {
      "severity": "warning",
      "channels": ["email"],
      "recipients": ["ops@company.com"]
    }
  }
}
```

### 2. Performance Metrics

#### 2.1 Metrics Collection
```typescript
// Collect and store metrics
interface MetricCollector {
  // Response times
  trackResponseTime(operation: string, duration: number): void;
  
  // Resource usage
  trackResourceUsage(metrics: ResourceMetrics): void;
  
  // Business metrics
  trackPayrollProcessing(metrics: PayrollMetrics): void;
}

// Implementation
class PrometheusCollector implements MetricCollector {
  private responseTime = new Histogram({
    name: 'arcpay_response_time',
    help: 'Response time in milliseconds',
    labelNames: ['operation']
  });
  
  trackResponseTime(operation: string, duration: number): void {
    this.responseTime.labels(operation).observe(duration);
  }
}
```

## Common Issues and Solutions

### 1. Performance Issues

#### Issue: Slow Payroll Processing
```bash
# Diagnose
npm run diagnose:performance -- --operation=payroll

# Common solutions:
# 1. Rebuild indexes
sqlite3 payroll.db "REINDEX;"

# 2. Clear calculation cache
npm run clear:cache -- --type=calculations

# 3. Optimize queries
npm run optimize:queries
```

#### Issue: High Memory Usage
```bash
# Check memory usage
npm run monitor:memory

# Solutions:
# 1. Increase Node.js memory limit
node --max-old-space-size=4096 dist/main.js

# 2. Clear caches
npm run clear:all-caches

# 3. Restart application
npm run restart
```

### 2. Data Issues

#### Issue: Calculation Discrepancies
```typescript
// Reconciliation script
async function reconcileCalculations(periodId: number) {
  const calculations = await getCalculations(periodId);
  
  for (const calc of calculations) {
    // Recalculate
    const fresh = await recalculate(calc.employeeId, periodId);
    
    // Compare
    const differences = compareCalculations(calc, fresh);
    
    if (differences.length > 0) {
      await logDiscrepancy(calc.id, differences);
    }
  }
}
```

#### Issue: Missing Tax Tables
```bash
# Verify tax tables
npm run verify:tax-tables -- --year=2025

# Re-import if missing
npm run import:tax-tables -- --year=2025 --force
```

## Maintenance Scripts

### 1. Daily Maintenance
```bash
#!/bin/bash
# /scripts/daily-maintenance.sh

echo "Starting daily maintenance..."

# Backup database
npm run backup:daily

# Clear old logs
npm run clean:logs -- --days=30

# Update statistics
npm run db:analyze

# Check system health
npm run health:check

echo "Daily maintenance completed"
```

### 2. Weekly Maintenance
```bash
#!/bin/bash
# /scripts/weekly-maintenance.sh

echo "Starting weekly maintenance..."

# Full system backup
npm run backup:full

# Database optimization
npm run db:optimize

# Security scan
npm run security:scan

# Performance analysis
npm run analyze:performance

echo "Weekly maintenance completed"
```

### 3. Monthly Maintenance
```bash
#!/bin/bash
# /scripts/monthly-maintenance.sh

echo "Starting monthly maintenance..."

# Archive old data
npm run archive:old-data

# Full database integrity check
npm run db:full-check

# Update documentation
npm run docs:generate

# Review security logs
npm run security:review

echo "Monthly maintenance completed"
```

## Disaster Recovery

### 1. Recovery Procedures

#### 1.1 Complete System Failure
1. Deploy new server/workstation
2. Install application from backup
3. Restore latest database backup
4. Verify data integrity
5. Test critical functions
6. Resume operations

#### 1.2 Data Corruption
1. Stop application immediately
2. Identify extent of corruption
3. Restore from last known good backup
4. Replay transactions from audit log
5. Verify data consistency
6. Resume operations

### 2. Business Continuity

#### 2.1 Minimum Requirements
- Backup workstation with application installed
- Access to cloud backups
- Printed emergency procedures
- Contact list for key personnel

#### 2.2 Recovery Time Objectives
- Critical functions: 4 hours
- Full restoration: 24 hours
- Historical data: 48 hours

## Compliance Audits

### 1. Annual Compliance Review

#### 1.1 Tax Compliance
- Verify tax calculations against examples
- Review tax table update procedures
- Audit tax filing records
- Check retention policies

#### 1.2 Labor Law Compliance
- Review overtime calculations
- Verify meal penalty logic
- Check prevailing wage compliance
- Audit certified payroll reports

### 2. Security Audits

#### 2.1 Access Control Review
```sql
-- Review user permissions
SELECT 
  u.username,
  u.role,
  u.last_login,
  COUNT(al.id) as actions_last_30_days
FROM users u
LEFT JOIN audit_log al ON u.id = al.user_id 
  AND al.timestamp > date('now', '-30 days')
GROUP BY u.id
ORDER BY u.last_login DESC;
```

#### 2.2 Encryption Verification
```bash
# Verify encryption status
npm run verify:encryption -- --check-all

# Rotate encryption keys
npm run rotate:keys -- --type=database
```

## Documentation Updates

### 1. Keep Documentation Current
- Update after major changes
- Review quarterly
- Version control all changes
- Include change logs

### 2. User Training
- Document new features
- Update training materials
- Conduct refresher sessions
- Maintain FAQ

## Maintenance Checklist

### Daily
- [ ] Verify automated backups completed
- [ ] Check system health dashboard
- [ ] Review error logs
- [ ] Monitor disk space

### Weekly
- [ ] Run database optimization
- [ ] Review security logs
- [ ] Check backup integrity
- [ ] Update system statistics

### Monthly
- [ ] Archive old data
- [ ] Full database integrity check
- [ ] Security vulnerability scan
- [ ] Performance analysis

### Quarterly
- [ ] Review and update documentation
- [ ] Compliance audit
- [ ] Disaster recovery test
- [ ] User access review

### Annually
- [ ] Update tax tables
- [ ] Renew certificates
- [ ] Full security audit
- [ ] Review retention policies