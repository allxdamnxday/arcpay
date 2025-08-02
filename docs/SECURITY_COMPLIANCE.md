# Security and Compliance Requirements

## Overview

This document outlines the security measures and compliance requirements for the Ironworkers Payroll Application. Given the sensitive nature of payroll data, including PII (Personally Identifiable Information) and financial information, robust security measures are critical.

## 1. Data Classification

### 1.1 Highly Sensitive Data
- **Social Security Numbers (SSN)**
- **Bank Account Numbers**
- **Bank Routing Numbers**
- **Driver's License Numbers**
- **Medical Information** (if applicable)

### 1.2 Sensitive Data
- **Employee Names with Addresses**
- **Dates of Birth**
- **Wage Information**
- **Tax Withholding Details**
- **Hours Worked**
- **Union Membership Information**

### 1.3 Internal Use Data
- **Project Information**
- **Wage Rate Tables**
- **Tax Tables**
- **System Configuration**

## 2. Encryption Requirements

### 2.1 Data at Rest

#### Field-Level Encryption
```typescript
interface EncryptionConfig {
    algorithm: 'AES-256-GCM';
    keyDerivation: 'PBKDF2';
    iterations: 100000;
    saltLength: 32;
    tagLength: 16;
}

class FieldEncryption {
    private masterKey: Buffer;
    
    encryptField(plaintext: string, fieldName: string): EncryptedField {
        // Generate unique IV for each encryption
        const iv = crypto.randomBytes(16);
        
        // Derive field-specific key
        const fieldKey = this.deriveFieldKey(fieldName);
        
        // Encrypt
        const cipher = crypto.createCipheriv('aes-256-gcm', fieldKey, iv);
        const encrypted = Buffer.concat([
            cipher.update(plaintext, 'utf8'),
            cipher.final()
        ]);
        
        const tag = cipher.getAuthTag();
        
        return {
            encrypted: encrypted.toString('base64'),
            iv: iv.toString('base64'),
            tag: tag.toString('base64')
        };
    }
}
```

#### Database Encryption
- Entire database file encrypted using SQLCipher
- Transparent encryption/decryption
- Key stored in secure key store

#### Backup Encryption
- All backups encrypted before storage
- Different key from main database
- Key rotation every 90 days

### 2.2 Data in Transit

#### IPC Communication
```typescript
interface IPCSecurityConfig {
    encryption: boolean;
    signing: boolean;
    nonceValidation: boolean;
}

class SecureIPC {
    sendMessage(channel: string, data: any): void {
        const message = {
            data,
            timestamp: Date.now(),
            nonce: crypto.randomBytes(16).toString('hex')
        };
        
        const encrypted = this.encrypt(message);
        const signed = this.sign(encrypted);
        
        ipcRenderer.send(channel, signed);
    }
}
```

#### External Communications
- All external APIs use HTTPS only
- Certificate pinning for critical services
- No sensitive data in URLs

### 2.3 Key Management

#### Key Hierarchy
```
Master Key (Hardware Security Module)
    ├── Database Encryption Key
    ├── Field Encryption Key
    ├── Backup Encryption Key
    └── Session Keys
```

#### Key Storage
```typescript
class KeyStore {
    private readonly keychain: Keychain;
    
    async storeMasterKey(key: Buffer): Promise<void> {
        // Use OS keychain (macOS Keychain, Windows Credential Store)
        await this.keychain.setPassword({
            account: 'ironworkers-payroll-master',
            service: 'com.ironworkers.payroll',
            password: key.toString('base64')
        });
    }
    
    async retrieveMasterKey(): Promise<Buffer> {
        const stored = await this.keychain.getPassword({
            account: 'ironworkers-payroll-master',
            service: 'com.ironworkers.payroll'
        });
        
        return Buffer.from(stored, 'base64');
    }
}
```

## 3. Authentication and Authorization

### 3.1 User Authentication

#### Multi-Factor Authentication
```typescript
interface AuthenticationRequirements {
    passwordMinLength: 12;
    passwordComplexity: {
        requireUppercase: true;
        requireLowercase: true;
        requireNumbers: true;
        requireSpecialChars: true;
        prohibitCommonPasswords: true;
        prohibitUserInfo: true;
    };
    mfaRequired: boolean;
    mfaMethods: ['totp', 'sms', 'email'];
    sessionTimeout: 30; // minutes
    maxFailedAttempts: 5;
    lockoutDuration: 30; // minutes
}
```

#### Password Storage
```typescript
class PasswordManager {
    async hashPassword(password: string): Promise<string> {
        // Use bcrypt with cost factor 12
        return bcrypt.hash(password, 12);
    }
    
    async verifyPassword(password: string, hash: string): Promise<boolean> {
        return bcrypt.compare(password, hash);
    }
}
```

### 3.2 Role-Based Access Control (RBAC)

#### User Roles
```typescript
enum UserRole {
    SUPER_ADMIN = 'super_admin',        // System-wide access
    UNION_ADMIN = 'union_admin',        // Union-specific admin
    PAYROLL_MANAGER = 'payroll_manager', // Process payroll
    PAYROLL_CLERK = 'payroll_clerk',    // Enter time, view limited data
    SUPERVISOR = 'supervisor',           // Approve time cards
    AUDITOR = 'auditor',                // Read-only access to all data
    EMPLOYEE = 'employee'               // Self-service access only
}
```

#### Permission Matrix
```typescript
interface PermissionMatrix {
    [UserRole.SUPER_ADMIN]: ['*'];
    [UserRole.UNION_ADMIN]: [
        'employees:*',
        'projects:*',
        'payroll:*',
        'reports:*',
        'settings:read',
        'settings:write:union'
    ];
    [UserRole.PAYROLL_MANAGER]: [
        'employees:read',
        'employees:write',
        'projects:read',
        'timecards:*',
        'payroll:*',
        'reports:*'
    ];
    [UserRole.PAYROLL_CLERK]: [
        'employees:read',
        'projects:read',
        'timecards:create',
        'timecards:read',
        'timecards:update:draft',
        'reports:read:basic'
    ];
    [UserRole.SUPERVISOR]: [
        'employees:read:team',
        'timecards:read:team',
        'timecards:approve:team',
        'reports:read:team'
    ];
    [UserRole.AUDITOR]: [
        'employees:read',
        'projects:read',
        'timecards:read',
        'payroll:read',
        'reports:read',
        'audit:read'
    ];
    [UserRole.EMPLOYEE]: [
        'employees:read:self',
        'timecards:read:self',
        'payroll:read:self',
        'reports:read:self'
    ];
}
```

### 3.3 Session Management

```typescript
class SessionManager {
    private readonly SESSION_DURATION = 30 * 60 * 1000; // 30 minutes
    private readonly IDLE_TIMEOUT = 15 * 60 * 1000;    // 15 minutes
    
    createSession(user: User): Session {
        const session = {
            id: crypto.randomBytes(32).toString('hex'),
            userId: user.id,
            createdAt: Date.now(),
            expiresAt: Date.now() + this.SESSION_DURATION,
            lastActivity: Date.now()
        };
        
        // Store in secure session store
        this.store.set(session.id, session);
        
        return session;
    }
    
    validateSession(sessionId: string): boolean {
        const session = this.store.get(sessionId);
        
        if (!session) return false;
        if (Date.now() > session.expiresAt) return false;
        if (Date.now() - session.lastActivity > this.IDLE_TIMEOUT) return false;
        
        // Update last activity
        session.lastActivity = Date.now();
        this.store.set(sessionId, session);
        
        return true;
    }
}
```

## 4. Audit Logging

### 4.1 Audit Requirements

#### Events to Audit
```typescript
enum AuditEvent {
    // Authentication
    LOGIN_SUCCESS = 'auth.login.success',
    LOGIN_FAILURE = 'auth.login.failure',
    LOGOUT = 'auth.logout',
    SESSION_EXPIRED = 'auth.session.expired',
    PASSWORD_CHANGED = 'auth.password.changed',
    
    // Data Access
    EMPLOYEE_VIEWED = 'data.employee.viewed',
    EMPLOYEE_MODIFIED = 'data.employee.modified',
    PAYROLL_VIEWED = 'data.payroll.viewed',
    PAYROLL_PROCESSED = 'data.payroll.processed',
    REPORT_GENERATED = 'data.report.generated',
    
    // Administrative
    USER_CREATED = 'admin.user.created',
    USER_MODIFIED = 'admin.user.modified',
    PERMISSION_CHANGED = 'admin.permission.changed',
    SETTINGS_CHANGED = 'admin.settings.changed',
    
    // Security
    ENCRYPTION_KEY_ROTATED = 'security.key.rotated',
    BACKUP_CREATED = 'security.backup.created',
    BACKUP_RESTORED = 'security.backup.restored',
    SUSPICIOUS_ACTIVITY = 'security.suspicious.activity'
}
```

#### Audit Log Structure
```typescript
interface AuditLogEntry {
    id: string;
    timestamp: number;
    event: AuditEvent;
    userId: string;
    userName: string;
    userRole: UserRole;
    ipAddress: string;
    userAgent: string;
    targetType?: string;    // e.g., 'employee', 'payroll'
    targetId?: string;      // ID of affected record
    oldValues?: any;        // For modifications
    newValues?: any;        // For modifications
    result: 'success' | 'failure';
    errorMessage?: string;
    additionalInfo?: Record<string, any>;
}
```

### 4.2 Audit Log Security

```typescript
class SecureAuditLogger {
    // Audit logs are write-only for all users except auditors
    // Logs cannot be modified or deleted
    // Logs are encrypted and signed
    
    async writeLog(entry: AuditLogEntry): Promise<void> {
        // Sign the entry
        const signature = this.signEntry(entry);
        
        // Encrypt sensitive fields
        const encrypted = this.encryptSensitiveFields(entry);
        
        // Write to append-only log
        await this.appendToLog({
            ...encrypted,
            signature,
            hash: this.hashEntry(entry)
        });
    }
    
    private hashEntry(entry: AuditLogEntry): string {
        // Include hash of previous entry for tamper detection
        const previousHash = this.getPreviousHash();
        const content = JSON.stringify(entry) + previousHash;
        return crypto.createHash('sha256').update(content).digest('hex');
    }
}
```

## 5. Compliance Requirements

### 5.1 Labor Law Compliance

#### Fair Labor Standards Act (FLSA)
- Accurate time tracking to the minute
- Proper overtime calculations
- Record retention for 3 years
- Employee access to their records

#### California Labor Code
- Meal and rest period tracking
- Accurate wage statements
- Timely payment of wages
- Penalty calculations for violations

#### Davis-Bacon Act (Federal Projects)
- Prevailing wage compliance
- Certified payroll reports
- Weekly submission requirements
- Fringe benefit tracking

### 5.2 Tax Compliance

#### IRS Requirements
- Accurate withholding calculations
- Timely deposit of withholdings
- Annual W-2 generation
- Quarterly 941 reporting

#### California EDD Requirements
- State tax withholding accuracy
- DE 9 quarterly reporting
- Annual reconciliation
- New hire reporting

### 5.3 Data Privacy Compliance

#### General Requirements
```typescript
interface PrivacyRequirements {
    dataMinimization: true;          // Collect only necessary data
    purposeLimitation: true;         // Use data only for stated purposes
    consentRequired: boolean;        // For certain data uses
    dataPortability: true;          // Export employee data
    rightToErasure: 'limited';      // Subject to retention requirements
    breachNotification: {
        internal: '24 hours';
        regulatory: '72 hours';
        affected: 'without undue delay';
    };
}
```

#### California Consumer Privacy Act (CCPA)
- Employee notice requirements
- Data inventory maintenance
- Access request handling
- Opt-out mechanisms (where applicable)

### 5.4 Industry Standards

#### PCI DSS (if storing card data)
- Not recommended for this application
- Use third-party payment processors

#### SOC 2 Type II (if required by unions)
- Security controls
- Availability monitoring
- Processing integrity
- Confidentiality measures

## 6. Security Controls

### 6.1 Application Security

#### Input Validation
```typescript
class InputValidator {
    validateSSN(ssn: string): ValidationResult {
        // Remove formatting
        const cleaned = ssn.replace(/\D/g, '');
        
        // Check length
        if (cleaned.length !== 9) {
            return { valid: false, error: 'SSN must be 9 digits' };
        }
        
        // Check for invalid patterns
        if (cleaned.startsWith('000') || cleaned.startsWith('666')) {
            return { valid: false, error: 'Invalid SSN pattern' };
        }
        
        // Check area number
        const area = parseInt(cleaned.substring(0, 3));
        if (area === 0 || area > 899) {
            return { valid: false, error: 'Invalid SSN area number' };
        }
        
        return { valid: true };
    }
    
    validateBankAccount(account: string): ValidationResult {
        // Implement ABA routing number checksum validation
        // Validate account number format
        return { valid: true };
    }
}
```

#### SQL Injection Prevention
```typescript
class SecureDatabase {
    // Always use parameterized queries
    async getEmployee(id: number): Promise<Employee> {
        const query = 'SELECT * FROM employees WHERE id = ? AND is_active = ?';
        return this.db.get(query, [id, true]);
    }
    
    // Never concatenate user input
    async searchEmployees(search: string): Promise<Employee[]> {
        const query = 'SELECT * FROM employees WHERE last_name LIKE ?';
        return this.db.all(query, [`%${search}%`]);
    }
}
```

### 6.2 Infrastructure Security

#### Desktop Application Security
```typescript
interface ElectronSecurityConfig {
    webPreferences: {
        contextIsolation: true;
        nodeIntegration: false;
        webviewTag: false;
        enableRemoteModule: false;
        preload: string; // Path to preload script
    };
    contentSecurityPolicy: {
        'default-src': ["'self'"];
        'script-src': ["'self'"];
        'style-src': ["'self'", "'unsafe-inline'"];
        'img-src': ["'self'", 'data:'];
        'connect-src': ["'self'"];
    };
}
```

#### Update Security
```typescript
class SecureUpdater {
    async checkForUpdates(): Promise<UpdateInfo> {
        // Verify update server certificate
        // Download over HTTPS only
        // Verify update signature
        const update = await this.downloadUpdate();
        
        if (!this.verifySignature(update)) {
            throw new Error('Invalid update signature');
        }
        
        return update;
    }
}
```

### 6.3 Operational Security

#### Backup Security
```typescript
class SecureBackup {
    async createBackup(): Promise<BackupInfo> {
        // Encrypt backup
        const encrypted = await this.encryptDatabase();
        
        // Generate backup metadata
        const metadata = {
            timestamp: Date.now(),
            version: this.getAppVersion(),
            hash: this.hashBackup(encrypted),
            encryptionMethod: 'AES-256-GCM'
        };
        
        // Sign metadata
        const signature = this.signMetadata(metadata);
        
        // Store securely
        await this.storeBackup(encrypted, metadata, signature);
        
        return { id: metadata.hash, timestamp: metadata.timestamp };
    }
}
```

#### Incident Response
```typescript
interface IncidentResponsePlan {
    detection: {
        monitoring: ['failed_logins', 'data_access_patterns', 'system_errors'];
        alerting: ['email', 'sms', 'dashboard'];
    };
    response: {
        immediate: ['isolate_affected_systems', 'preserve_evidence'];
        shortTerm: ['investigate_scope', 'notify_stakeholders'];
        longTerm: ['remediate_vulnerabilities', 'update_procedures'];
    };
    recovery: {
        steps: ['verify_remediation', 'restore_operations', 'monitor_closely'];
    };
    postIncident: {
        review: ['root_cause_analysis', 'lessons_learned'];
        improvements: ['update_security_controls', 'staff_training'];
    };
}
```

## 7. Security Testing Requirements

### 7.1 Penetration Testing
- Annual third-party penetration test
- Quarterly automated vulnerability scans
- Pre-release security testing

### 7.2 Code Security
```typescript
interface CodeSecurityRequirements {
    staticAnalysis: {
        tools: ['ESLint', 'SonarQube', 'Snyk'];
        frequency: 'every_commit';
    };
    dependencyScanning: {
        tools: ['npm audit', 'Dependabot'];
        frequency: 'daily';
    };
    secretsScanning: {
        tools: ['GitLeaks', 'TruffleHog'];
        frequency: 'every_commit';
    };
}
```

### 7.3 Security Metrics
- Failed login attempts
- Unauthorized access attempts
- Average session duration
- Time to detect incidents
- Time to resolve vulnerabilities

## 8. Employee Training

### 8.1 Security Awareness
- Initial security training for all users
- Annual refresher training
- Role-specific security training
- Phishing simulation exercises

### 8.2 Incident Reporting
- Clear reporting procedures
- Anonymous reporting option
- No retaliation policy
- Reward program for security findings

## 9. Compliance Checklist

### 9.1 Pre-Launch
- [ ] Security assessment completed
- [ ] Encryption implemented and tested
- [ ] Access controls configured
- [ ] Audit logging operational
- [ ] Backup procedures tested
- [ ] Incident response plan documented
- [ ] Staff training completed

### 9.2 Ongoing
- [ ] Monthly security reviews
- [ ] Quarterly access audits
- [ ] Annual penetration testing
- [ ] Regular compliance updates
- [ ] Continuous monitoring
- [ ] Timely patching

## 10. Documentation Requirements

### 10.1 Security Documentation
- Security architecture diagram
- Data flow diagrams
- Encryption key procedures
- Incident response runbooks
- Recovery procedures

### 10.2 Compliance Documentation
- Privacy notices
- Data retention policies
- Access control matrices
- Audit procedures
- Training records