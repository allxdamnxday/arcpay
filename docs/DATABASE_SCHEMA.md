# Database Schema Specification

## Overview

This document provides the complete database schema for the Ironworkers Payroll Application. The schema is designed to handle complex union payroll requirements while maintaining data integrity, security, and performance.

## Database Design Principles

1. **Normalization**: 3NF to prevent data anomalies
2. **Audit Trail**: Every table includes audit fields
3. **Soft Deletes**: Records are marked inactive, not deleted
4. **Versioning**: Critical data maintains history
5. **Encryption**: Sensitive fields use field-level encryption

## Core Tables

### 1. Union Locals
```sql
CREATE TABLE union_locals (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    local_number VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    address_line1 VARCHAR(100) NOT NULL,
    address_line2 VARCHAR(100),
    city VARCHAR(50) NOT NULL,
    state CHAR(2) NOT NULL,
    zip_code VARCHAR(10) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(100),
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);

CREATE INDEX idx_union_locals_number ON union_locals(local_number);
```

### 2. Employees
```sql
CREATE TABLE employees (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    employee_number VARCHAR(20) NOT NULL UNIQUE,
    union_local_id INTEGER NOT NULL,
    -- Personal Information (Encrypted)
    ssn_encrypted VARCHAR(255) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    middle_name VARCHAR(50),
    date_of_birth DATE NOT NULL,
    -- Contact Information
    address_line1 VARCHAR(100) NOT NULL,
    address_line2 VARCHAR(100),
    city VARCHAR(50) NOT NULL,
    state CHAR(2) NOT NULL,
    zip_code VARCHAR(10) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(100),
    -- Employment Information
    hire_date DATE NOT NULL,
    classification_id INTEGER NOT NULL,
    apprentice_level INTEGER CHECK (apprentice_level BETWEEN 1 AND 8),
    journeyman_date DATE,
    -- Tax Information
    federal_filing_status VARCHAR(20) NOT NULL,
    federal_allowances INTEGER DEFAULT 0,
    state_filing_status VARCHAR(20) NOT NULL,
    state_allowances INTEGER DEFAULT 0,
    additional_federal_withholding DECIMAL(10,2) DEFAULT 0,
    additional_state_withholding DECIMAL(10,2) DEFAULT 0,
    -- Payment Information
    pay_method VARCHAR(20) NOT NULL CHECK (pay_method IN ('check', 'direct_deposit')),
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    termination_date DATE,
    FOREIGN KEY (union_local_id) REFERENCES union_locals(id),
    FOREIGN KEY (classification_id) REFERENCES classifications(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);

CREATE INDEX idx_employees_number ON employees(employee_number);
CREATE INDEX idx_employees_ssn ON employees(ssn_encrypted);
CREATE INDEX idx_employees_name ON employees(last_name, first_name);
CREATE INDEX idx_employees_active ON employees(is_active);
```

### 3. Employee Bank Accounts (Encrypted)
```sql
CREATE TABLE employee_bank_accounts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    employee_id INTEGER NOT NULL,
    account_type VARCHAR(20) NOT NULL CHECK (account_type IN ('checking', 'savings')),
    routing_number_encrypted VARCHAR(255) NOT NULL,
    account_number_encrypted VARCHAR(255) NOT NULL,
    bank_name VARCHAR(100) NOT NULL,
    is_primary BOOLEAN DEFAULT FALSE,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (employee_id) REFERENCES employees(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);

CREATE INDEX idx_bank_accounts_employee ON employee_bank_accounts(employee_id);
```

### 4. Classifications
```sql
CREATE TABLE classifications (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    code VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    is_apprentice BOOLEAN DEFAULT FALSE,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);
```

### 5. Wage Zones
```sql
CREATE TABLE wage_zones (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    zone_number INTEGER NOT NULL CHECK (zone_number BETWEEN 1 AND 5),
    name VARCHAR(50) NOT NULL,
    description TEXT,
    -- Geographic boundaries (simplified - in production use PostGIS)
    counties TEXT, -- JSON array of county names
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);
```

### 6. Wage Rates (Versioned)
```sql
CREATE TABLE wage_rates (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    union_local_id INTEGER NOT NULL,
    classification_id INTEGER NOT NULL,
    zone_id INTEGER NOT NULL,
    effective_date DATE NOT NULL,
    expiration_date DATE,
    -- Base rates
    base_hourly_rate DECIMAL(10,4) NOT NULL,
    -- Shift differentials (as percentage)
    shift2_differential DECIMAL(5,4) DEFAULT 0.06, -- 6%
    shift3_differential DECIMAL(5,4) DEFAULT 0.13, -- 13%
    -- Fringe benefits
    health_welfare_hourly DECIMAL(10,4) NOT NULL,
    pension_hourly DECIMAL(10,4) NOT NULL,
    vacation_hourly DECIMAL(10,4) NOT NULL,
    training_hourly DECIMAL(10,4) NOT NULL,
    -- Apprentice percentages (if applicable)
    apprentice_percent_level1 DECIMAL(5,4),
    apprentice_percent_level2 DECIMAL(5,4),
    apprentice_percent_level3 DECIMAL(5,4),
    apprentice_percent_level4 DECIMAL(5,4),
    apprentice_percent_level5 DECIMAL(5,4),
    apprentice_percent_level6 DECIMAL(5,4),
    apprentice_percent_level7 DECIMAL(5,4),
    apprentice_percent_level8 DECIMAL(5,4),
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (union_local_id) REFERENCES union_locals(id),
    FOREIGN KEY (classification_id) REFERENCES classifications(id),
    FOREIGN KEY (zone_id) REFERENCES wage_zones(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id),
    UNIQUE(union_local_id, classification_id, zone_id, effective_date)
);

CREATE INDEX idx_wage_rates_lookup ON wage_rates(union_local_id, classification_id, zone_id, effective_date);
```

### 7. Projects
```sql
CREATE TABLE projects (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_number VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    contractor_name VARCHAR(200) NOT NULL,
    -- Location
    address_line1 VARCHAR(100) NOT NULL,
    address_line2 VARCHAR(100),
    city VARCHAR(50) NOT NULL,
    state CHAR(2) NOT NULL,
    zip_code VARCHAR(10) NOT NULL,
    county VARCHAR(50) NOT NULL,
    wage_zone_id INTEGER NOT NULL,
    -- Project details
    project_type VARCHAR(50) NOT NULL,
    is_federal BOOLEAN DEFAULT FALSE,
    federal_wage_determination VARCHAR(50),
    prevailing_wage_required BOOLEAN DEFAULT FALSE,
    -- Dates
    start_date DATE NOT NULL,
    estimated_end_date DATE,
    actual_end_date DATE,
    -- Special rates
    has_special_rates BOOLEAN DEFAULT FALSE,
    special_overtime_rules TEXT, -- JSON
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (wage_zone_id) REFERENCES wage_zones(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);

CREATE INDEX idx_projects_number ON projects(project_number);
CREATE INDEX idx_projects_active ON projects(is_active);
CREATE INDEX idx_projects_zone ON projects(wage_zone_id);
```

### 8. Time Cards
```sql
CREATE TABLE time_cards (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    employee_id INTEGER NOT NULL,
    project_id INTEGER NOT NULL,
    work_date DATE NOT NULL,
    -- Time entries (stored as decimal hours)
    regular_hours DECIMAL(4,2) DEFAULT 0,
    overtime_hours DECIMAL(4,2) DEFAULT 0,
    double_time_hours DECIMAL(4,2) DEFAULT 0,
    -- Shift information
    shift_number INTEGER CHECK (shift_number IN (1, 2, 3)),
    -- Show up pay
    show_up_pay_applicable BOOLEAN DEFAULT FALSE,
    show_up_pay_hours DECIMAL(4,2) DEFAULT 0,
    -- Meal periods
    meal_period_taken BOOLEAN DEFAULT TRUE,
    meal_penalty_applicable BOOLEAN DEFAULT FALSE,
    -- Travel and subsistence
    travel_hours DECIMAL(4,2) DEFAULT 0,
    subsistence_applicable BOOLEAN DEFAULT FALSE,
    subsistence_amount DECIMAL(10,2) DEFAULT 0,
    -- Federal premium
    federal_premium_applicable BOOLEAN DEFAULT FALSE,
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'submitted', 'approved', 'processed', 'void')),
    approved_by INTEGER,
    approved_at TIMESTAMP,
    -- Notes
    employee_notes TEXT,
    supervisor_notes TEXT,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    FOREIGN KEY (employee_id) REFERENCES employees(id),
    FOREIGN KEY (project_id) REFERENCES projects(id),
    FOREIGN KEY (approved_by) REFERENCES users(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id),
    UNIQUE(employee_id, project_id, work_date)
);

CREATE INDEX idx_time_cards_employee_date ON time_cards(employee_id, work_date);
CREATE INDEX idx_time_cards_project_date ON time_cards(project_id, work_date);
CREATE INDEX idx_time_cards_status ON time_cards(status);
```

### 9. Payroll Periods
```sql
CREATE TABLE payroll_periods (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    union_local_id INTEGER NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    pay_date DATE NOT NULL,
    period_number INTEGER NOT NULL,
    period_year INTEGER NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'processing', 'approved', 'paid', 'closed')),
    -- Processing information
    processed_by INTEGER,
    processed_at TIMESTAMP,
    approved_by INTEGER,
    approved_at TIMESTAMP,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    FOREIGN KEY (union_local_id) REFERENCES union_locals(id),
    FOREIGN KEY (processed_by) REFERENCES users(id),
    FOREIGN KEY (approved_by) REFERENCES users(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id),
    UNIQUE(union_local_id, period_start, period_end)
);

CREATE INDEX idx_payroll_periods_dates ON payroll_periods(period_start, period_end);
CREATE INDEX idx_payroll_periods_status ON payroll_periods(status);
```

### 10. Payroll Calculations
```sql
CREATE TABLE payroll_calculations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    payroll_period_id INTEGER NOT NULL,
    employee_id INTEGER NOT NULL,
    -- Hours summary
    total_regular_hours DECIMAL(6,2) NOT NULL,
    total_overtime_hours DECIMAL(6,2) NOT NULL,
    total_double_time_hours DECIMAL(6,2) NOT NULL,
    -- Gross wages
    regular_wages DECIMAL(10,2) NOT NULL,
    overtime_wages DECIMAL(10,2) NOT NULL,
    double_time_wages DECIMAL(10,2) NOT NULL,
    show_up_pay DECIMAL(10,2) DEFAULT 0,
    meal_penalty_pay DECIMAL(10,2) DEFAULT 0,
    travel_pay DECIMAL(10,2) DEFAULT 0,
    subsistence_pay DECIMAL(10,2) DEFAULT 0,
    federal_premium_pay DECIMAL(10,2) DEFAULT 0,
    gross_wages DECIMAL(10,2) NOT NULL,
    -- Deductions
    federal_tax DECIMAL(10,2) NOT NULL,
    state_tax DECIMAL(10,2) NOT NULL,
    social_security_tax DECIMAL(10,2) NOT NULL,
    medicare_tax DECIMAL(10,2) NOT NULL,
    state_disability DECIMAL(10,2) NOT NULL,
    -- Union deductions
    union_dues DECIMAL(10,2) DEFAULT 0,
    working_dues DECIMAL(10,2) DEFAULT 0,
    -- Other deductions
    other_deductions DECIMAL(10,2) DEFAULT 0,
    total_deductions DECIMAL(10,2) NOT NULL,
    -- Net pay
    net_pay DECIMAL(10,2) NOT NULL,
    -- Fringe benefits (employer paid)
    health_welfare_contribution DECIMAL(10,2) NOT NULL,
    pension_contribution DECIMAL(10,2) NOT NULL,
    vacation_contribution DECIMAL(10,2) NOT NULL,
    training_contribution DECIMAL(10,2) NOT NULL,
    total_fringe_benefits DECIMAL(10,2) NOT NULL,
    -- Payment information
    payment_method VARCHAR(20) NOT NULL,
    check_number VARCHAR(20),
    direct_deposit_confirmation VARCHAR(50),
    -- Calculation details (JSON)
    calculation_details TEXT NOT NULL, -- JSON with full breakdown
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'calculated' CHECK (status IN ('calculated', 'approved', 'paid', 'void')),
    void_reason TEXT,
    voided_by INTEGER,
    voided_at TIMESTAMP,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    FOREIGN KEY (payroll_period_id) REFERENCES payroll_periods(id),
    FOREIGN KEY (employee_id) REFERENCES employees(id),
    FOREIGN KEY (voided_by) REFERENCES users(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id),
    UNIQUE(payroll_period_id, employee_id)
);

CREATE INDEX idx_payroll_calculations_period ON payroll_calculations(payroll_period_id);
CREATE INDEX idx_payroll_calculations_employee ON payroll_calculations(employee_id);
CREATE INDEX idx_payroll_calculations_status ON payroll_calculations(status);
```

### 11. Tax Tables
```sql
-- Federal Tax Brackets
CREATE TABLE federal_tax_brackets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    year INTEGER NOT NULL,
    filing_status VARCHAR(20) NOT NULL,
    payroll_frequency VARCHAR(20) NOT NULL,
    min_income DECIMAL(10,2) NOT NULL,
    max_income DECIMAL(10,2),
    tax_rate DECIMAL(5,4) NOT NULL,
    base_tax DECIMAL(10,2) NOT NULL,
    effective_date DATE NOT NULL,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    FOREIGN KEY (created_by) REFERENCES users(id)
);

CREATE INDEX idx_federal_tax_lookup ON federal_tax_brackets(year, filing_status, payroll_frequency, min_income);

-- California Tax Brackets
CREATE TABLE california_tax_brackets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    year INTEGER NOT NULL,
    filing_status VARCHAR(20) NOT NULL,
    payroll_frequency VARCHAR(20) NOT NULL,
    min_income DECIMAL(10,2) NOT NULL,
    max_income DECIMAL(10,2),
    tax_rate DECIMAL(5,4) NOT NULL,
    base_tax DECIMAL(10,2) NOT NULL,
    effective_date DATE NOT NULL,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    FOREIGN KEY (created_by) REFERENCES users(id)
);

CREATE INDEX idx_california_tax_lookup ON california_tax_brackets(year, filing_status, payroll_frequency, min_income);

-- Standard Deductions
CREATE TABLE standard_deductions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    year INTEGER NOT NULL,
    tax_type VARCHAR(20) NOT NULL CHECK (tax_type IN ('federal', 'california')),
    filing_status VARCHAR(20) NOT NULL,
    payroll_frequency VARCHAR(20) NOT NULL,
    deduction_amount DECIMAL(10,2) NOT NULL,
    allowance_amount DECIMAL(10,2) NOT NULL,
    effective_date DATE NOT NULL,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    FOREIGN KEY (created_by) REFERENCES users(id),
    UNIQUE(year, tax_type, filing_status, payroll_frequency)
);
```

### 12. Users and Security
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    role VARCHAR(50) NOT NULL,
    union_local_id INTEGER,
    -- Security
    last_login TIMESTAMP,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,
    password_reset_token VARCHAR(255),
    password_reset_expires TIMESTAMP,
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (union_local_id) REFERENCES union_locals(id),
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);

-- User Roles
CREATE TABLE roles (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,
    permissions TEXT NOT NULL, -- JSON array of permissions
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (updated_by) REFERENCES users(id)
);

-- User Sessions
CREATE TABLE user_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    session_token VARCHAR(255) NOT NULL UNIQUE,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_sessions_token ON user_sessions(session_token);
CREATE INDEX idx_sessions_user ON user_sessions(user_id);
```

### 13. Audit Trail
```sql
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    table_name VARCHAR(50) NOT NULL,
    record_id INTEGER NOT NULL,
    action VARCHAR(20) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE', 'VIEW')),
    user_id INTEGER NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT,
    old_values TEXT, -- JSON
    new_values TEXT, -- JSON
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp);
```

## Data Integrity Rules

### 1. Business Rules
- Employee cannot have overlapping time cards for the same date
- Payroll period dates cannot overlap for the same union local
- Wage rates must have non-overlapping effective dates
- Apprentice level must be between 1-8 if employee is apprentice
- Project end date must be after start date

### 2. Referential Integrity
- All foreign keys must reference existing records
- Cascade updates for audit fields
- Prevent deletion of referenced records

### 3. Data Validation
- SSN must be valid format (implemented in application layer)
- Email addresses must be valid format
- Phone numbers must be valid format
- Zip codes must be valid format
- Dates must be reasonable (not in future for historical data)

## Security Considerations

### 1. Encrypted Fields
- `employees.ssn_encrypted`
- `employee_bank_accounts.routing_number_encrypted`
- `employee_bank_accounts.account_number_encrypted`

### 2. Access Control
- Row-level security based on union_local_id
- Role-based permissions for all operations
- Audit trail for all data access

### 3. Data Retention
- Active employee data: Indefinite
- Terminated employee data: 7 years
- Payroll records: 7 years
- Audit logs: 3 years

## Performance Optimization

### 1. Indexes
- All foreign keys are indexed
- Common query patterns have covering indexes
- Date range queries are optimized

### 2. Partitioning Strategy
- Consider partitioning large tables by date (time_cards, payroll_calculations)
- Archive old data to separate tables

### 3. Query Optimization
- Use prepared statements
- Implement query result caching
- Batch operations where possible

## Migration Strategy

### 1. Version Control
- Each schema change is a numbered migration
- Migrations are reversible
- Migration history is tracked

### 2. Data Migration
- ETL scripts for importing existing data
- Data validation before and after migration
- Rollback procedures

### 3. Zero-Downtime Deployment
- Add new columns as nullable
- Backfill data
- Add constraints after backfill
- Remove old columns in separate migration