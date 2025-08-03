# Data Migration Guide

## Overview

This guide provides comprehensive procedures for migrating data from existing payroll systems to ArcPay. It covers data extraction, transformation, validation, and loading processes to ensure accurate and complete migration.

## Migration Planning

### Pre-Migration Assessment

#### 1. Source System Analysis
```typescript
interface SourceSystemAssessment {
  systemName: string;
  version: string;
  dataFormat: 'database' | 'files' | 'api';
  totalRecords: {
    employees: number;
    timecards: number;
    payrollRecords: number;
    projects: number;
  };
  dateRange: {
    oldest: Date;
    newest: Date;
  };
  customFields: string[];
  dataQualityIssues: string[];
}
```

#### 2. Data Mapping Document
Create a comprehensive mapping document:

| Source Field | Source Type | Target Table | Target Field | Transformation | Notes |
|--------------|-------------|--------------|--------------|----------------|--------|
| EMP_ID | VARCHAR(10) | employees | employee_number | Direct | Unique check |
| SSN | VARCHAR(11) | employees | ssn_encrypted | Encrypt | Format: XXX-XX-XXXX |
| HRLY_RATE | DECIMAL(5,2) | wage_rates | base_hourly_rate | Currency | Zone-based |
| FED_STATUS | CHAR(1) | employees | federal_filing_status | Lookup | S→single, M→married |

### Migration Strategy

#### 1. Phased Approach
```mermaid
graph LR
    A[Phase 1: Master Data] --> B[Phase 2: Historical Data]
    B --> C[Phase 3: Current Period]
    C --> D[Phase 4: Validation]
    D --> E[Phase 5: Cutover]
```

#### 2. Parallel Run Period
- Run both systems for one complete payroll cycle
- Compare results daily
- Reconcile any discrepancies
- Build user confidence

## Data Extraction

### 1. Employee Data Extraction

#### 1.1 SQL-Based Systems
```sql
-- Extract employee master data
SELECT 
    emp_id,
    ssn,
    first_name,
    last_name,
    middle_initial,
    date_of_birth,
    hire_date,
    termination_date,
    address_1,
    address_2,
    city,
    state,
    zip_code,
    phone,
    email,
    fed_filing_status,
    fed_allowances,
    state_filing_status,
    state_allowances,
    pay_rate,
    pay_type,
    union_local,
    classification,
    apprentice_level
FROM employee_master
WHERE status IN ('A', 'T') -- Active and Terminated
ORDER BY emp_id;
```

#### 1.2 File-Based Systems
```typescript
// CSV extraction parser
export class CSVEmployeeExtractor {
  async extractEmployees(filePath: string): Promise<SourceEmployee[]> {
    const employees: SourceEmployee[] = [];
    
    const stream = fs.createReadStream(filePath)
      .pipe(csv.parse({ headers: true }))
      .on('data', (row) => {
        employees.push(this.mapEmployee(row));
      });
    
    await finished(stream);
    return employees;
  }
  
  private mapEmployee(row: any): SourceEmployee {
    return {
      sourceId: row['Employee ID'],
      ssn: this.formatSSN(row['SSN']),
      firstName: this.cleanName(row['First Name']),
      lastName: this.cleanName(row['Last Name']),
      // ... map remaining fields
    };
  }
}
```

### 2. Project Data Extraction

```typescript
// Extract project and zone information
interface ProjectExtractor {
  async extractProjects(source: DataSource): Promise<SourceProject[]> {
    const projects = await source.query(`
      SELECT 
        project_code,
        project_name,
        contractor,
        address,
        city,
        county,
        state,
        zip,
        start_date,
        end_date,
        prevailing_wage,
        zone_override
      FROM projects
      WHERE start_date >= ?
    `, [this.cutoffDate]);
    
    return projects.map(p => this.mapProject(p));
  }
  
  private mapProject(source: any): SourceProject {
    return {
      projectNumber: source.project_code,
      name: source.project_name,
      contractorName: source.contractor,
      address: this.parseAddress(source),
      wageZone: this.determineZone(source),
      isPrevailingWage: source.prevailing_wage === 'Y',
      startDate: this.parseDate(source.start_date),
      endDate: this.parseDate(source.end_date)
    };
  }
}
```

### 3. Historical Timecard Extraction

```typescript
// Extract historical timecards with batching
export class TimecardExtractor {
  private batchSize = 10000;
  
  async extractTimecards(
    startDate: Date,
    endDate: Date,
    onProgress?: (progress: number) => void
  ): Promise<void> {
    const totalCount = await this.getTimecardCount(startDate, endDate);
    let processed = 0;
    
    for (let offset = 0; offset < totalCount; offset += this.batchSize) {
      const batch = await this.extractBatch(startDate, endDate, offset);
      await this.processBatch(batch);
      
      processed += batch.length;
      onProgress?.(processed / totalCount * 100);
    }
  }
  
  private async extractBatch(
    startDate: Date,
    endDate: Date,
    offset: number
  ): Promise<SourceTimecard[]> {
    return await this.source.query(`
      SELECT 
        tc.*,
        p.project_code,
        e.emp_id
      FROM timecards tc
      JOIN projects p ON tc.project_id = p.id
      JOIN employees e ON tc.employee_id = e.id
      WHERE tc.work_date BETWEEN ? AND ?
      ORDER BY tc.work_date, tc.employee_id
      LIMIT ? OFFSET ?
    `, [startDate, endDate, this.batchSize, offset]);
  }
}
```

## Data Transformation

### 1. Employee Transformation

#### 1.1 Data Cleansing
```typescript
export class EmployeeTransformer {
  transform(source: SourceEmployee): TargetEmployee {
    return {
      employeeNumber: this.generateEmployeeNumber(source),
      ssn: this.validateAndFormatSSN(source.ssn),
      firstName: this.cleanName(source.firstName),
      lastName: this.cleanName(source.lastName),
      middleName: this.expandMiddleInitial(source.middleInitial),
      dateOfBirth: this.validateDate(source.dateOfBirth),
      unionLocalId: this.mapUnionLocal(source.unionLocal),
      classificationId: this.mapClassification(source.classification),
      apprenticeLevel: this.validateApprenticeLevel(source),
      // Tax information
      federalFilingStatus: this.mapFederalStatus(source.fedStatus),
      federalAllowances: this.validateAllowances(source.fedAllowances),
      stateFilingStatus: this.mapStateStatus(source.stateStatus),
      stateAllowances: this.validateAllowances(source.stateAllowances),
      // Address
      address: this.standardizeAddress(source),
      // Contact
      phone: this.formatPhone(source.phone),
      email: this.validateEmail(source.email)
    };
  }
  
  private validateAndFormatSSN(ssn: string): string {
    // Remove all non-digits
    const digits = ssn.replace(/\D/g, '');
    
    if (digits.length !== 9) {
      throw new ValidationError(`Invalid SSN length: ${ssn}`);
    }
    
    // Validate SSN rules
    if (digits.startsWith('000') || digits.startsWith('666')) {
      throw new ValidationError(`Invalid SSN pattern: ${ssn}`);
    }
    
    // Format as XXX-XX-XXXX
    return `${digits.slice(0, 3)}-${digits.slice(3, 5)}-${digits.slice(5)}`;
  }
  
  private cleanName(name: string): string {
    return name
      .trim()
      .replace(/\s+/g, ' ')
      .split(' ')
      .map(part => 
        part.charAt(0).toUpperCase() + part.slice(1).toLowerCase()
      )
      .join(' ');
  }
  
  private mapFederalStatus(code: string): FederalFilingStatus {
    const mapping: Record<string, FederalFilingStatus> = {
      'S': 'single',
      'M': 'married_filing_jointly',
      'H': 'head_of_household',
      'MS': 'married_filing_separately',
      '1': 'single', // Legacy codes
      '2': 'married_filing_jointly'
    };
    
    const status = mapping[code.toUpperCase()];
    if (!status) {
      throw new ValidationError(`Unknown federal status code: ${code}`);
    }
    
    return status;
  }
}
```

#### 1.2 Wage Rate Mapping
```typescript
export class WageRateTransformer {
  async transformWageRates(
    employees: SourceEmployee[],
    effectiveDate: Date
  ): Promise<WageRate[]> {
    const wageRates: WageRate[] = [];
    
    // Group by union local and classification
    const groups = this.groupByRateCategory(employees);
    
    for (const group of groups) {
      const rate = await this.createWageRate(group, effectiveDate);
      wageRates.push(rate);
    }
    
    return wageRates;
  }
  
  private async createWageRate(
    group: EmployeeGroup,
    effectiveDate: Date
  ): Promise<WageRate> {
    // Determine zone based on majority
    const zone = await this.determineZone(group.employees);
    
    // Calculate base rate (highest in group for safety)
    const baseRate = Math.max(...group.employees.map(e => e.hourlyRate));
    
    return {
      unionLocalId: group.unionLocalId,
      classificationId: group.classificationId,
      zoneId: zone.id,
      effectiveDate,
      baseHourlyRate: baseRate,
      // Default shift differentials
      shift2Differential: 0.06,
      shift3Differential: 0.13,
      // Fringe benefits (need manual verification)
      healthWelfareHourly: 0, // TO BE UPDATED
      pensionHourly: 0,       // TO BE UPDATED
      vacationHourly: 0,      // TO BE UPDATED
      trainingHourly: 0,      // TO BE UPDATED
      // Apprentice percentages
      apprenticePercentages: this.getApprenticePercentages()
    };
  }
}
```

### 2. Timecard Transformation

#### 2.1 Overtime Calculation Verification
```typescript
export class TimecardTransformer {
  async transformTimecard(
    source: SourceTimecard,
    employee: Employee,
    project: Project
  ): Promise<TimeCard> {
    // Parse hours
    const hours = this.parseHours(source);
    
    // Verify overtime calculation
    const calculatedOT = this.calculateOvertime(
      hours.total,
      source.workDate,
      project
    );
    
    // Compare with source system
    if (source.overtimeHours && 
        Math.abs(source.overtimeHours - calculatedOT.overtime) > 0.01) {
      this.logDiscrepancy({
        type: 'OVERTIME_MISMATCH',
        sourceId: source.id,
        sourceOT: source.overtimeHours,
        calculatedOT: calculatedOT.overtime,
        date: source.workDate
      });
    }
    
    return {
      employeeId: employee.id,
      projectId: project.id,
      workDate: this.parseDate(source.workDate),
      regularHours: calculatedOT.regular,
      overtimeHours: calculatedOT.overtime,
      doubleTimeHours: calculatedOT.doubleTime,
      shiftNumber: this.determineShift(source),
      mealPeriodTaken: this.parseMealPeriod(source),
      // Special pay indicators
      showUpPayApplicable: this.checkShowUpPay(source),
      federalPremiumApplicable: project.isFederal
    };
  }
}
```

### 3. Tax Information Migration

#### 3.1 W-4 Data Conversion
```typescript
export class TaxDataTransformer {
  transformW4Data(source: SourceEmployee): W4Information {
    // Determine W-4 version based on data
    const w4Version = this.determineW4Version(source);
    
    if (w4Version >= 2020) {
      return this.transformNewW4(source);
    } else {
      return this.transformLegacyW4(source);
    }
  }
  
  private transformLegacyW4(source: SourceEmployee): W4Information {
    return {
      version: 2019,
      filingStatus: this.mapFilingStatus(source.fedStatus),
      allowances: source.fedAllowances || 0,
      additionalWithholding: source.additionalFedWithholding || 0,
      exempt: source.fedExempt === 'Y'
    };
  }
  
  private transformNewW4(source: SourceEmployee): W4Information {
    // Map old allowances to new system
    const allowances = source.fedAllowances || 0;
    
    return {
      version: 2020,
      filingStatus: this.mapFilingStatus(source.fedStatus),
      multipleJobs: false, // Default, needs review
      dependentCredit: allowances > 2 ? (allowances - 2) * 2000 : 0,
      otherIncome: 0, // Not available in old system
      deductions: 0,   // Not available in old system
      additionalWithholding: source.additionalFedWithholding || 0
    };
  }
}
```

## Data Validation

### 1. Pre-Migration Validation

#### 1.1 Data Quality Checks
```typescript
export class DataQualityValidator {
  async validateSourceData(): Promise<ValidationReport> {
    const report = new ValidationReport();
    
    // Check for duplicates
    report.addCheck(await this.checkDuplicateSSNs());
    report.addCheck(await this.checkDuplicateEmployeeNumbers());
    
    // Check for missing required fields
    report.addCheck(await this.checkMissingRequiredFields());
    
    // Check for invalid data
    report.addCheck(await this.checkInvalidDates());
    report.addCheck(await this.checkInvalidSSNs());
    report.addCheck(await this.checkInvalidWageRates());
    
    // Check referential integrity
    report.addCheck(await this.checkOrphanedTimecards());
    report.addCheck(await this.checkInvalidProjectReferences());
    
    return report;
  }
  
  private async checkDuplicateSSNs(): Promise<CheckResult> {
    const duplicates = await this.source.query(`
      SELECT ssn, COUNT(*) as count
      FROM employees
      WHERE ssn IS NOT NULL
      GROUP BY ssn
      HAVING COUNT(*) > 1
    `);
    
    return {
      name: 'Duplicate SSNs',
      passed: duplicates.length === 0,
      errors: duplicates.map(d => 
        `SSN ${this.maskSSN(d.ssn)} appears ${d.count} times`
      )
    };
  }
  
  private async checkInvalidWageRates(): Promise<CheckResult> {
    const invalid = await this.source.query(`
      SELECT emp_id, hourly_rate
      FROM employees
      WHERE hourly_rate <= 0 
         OR hourly_rate > 200
         OR hourly_rate IS NULL
    `);
    
    return {
      name: 'Invalid Wage Rates',
      passed: invalid.length === 0,
      errors: invalid.map(e => 
        `Employee ${e.emp_id}: $${e.hourly_rate}/hour`
      )
    };
  }
}
```

#### 1.2 Business Rule Validation
```typescript
export class BusinessRuleValidator {
  async validateBusinessRules(
    employees: Employee[],
    timecards: TimeCard[]
  ): Promise<ValidationReport> {
    const report = new ValidationReport();
    
    // Validate apprentice levels
    report.addCheck(this.validateApprenticeLevels(employees));
    
    // Validate overtime rules
    report.addCheck(this.validateOvertimeRules(timecards));
    
    // Validate zone assignments
    report.addCheck(this.validateZoneAssignments(employees));
    
    // Validate tax configurations
    report.addCheck(this.validateTaxSetup(employees));
    
    return report;
  }
  
  private validateApprenticeLevels(employees: Employee[]): CheckResult {
    const errors: string[] = [];
    
    employees
      .filter(e => e.classificationId === APPRENTICE_CLASS_ID)
      .forEach(e => {
        if (!e.apprenticeLevel || e.apprenticeLevel < 1 || e.apprenticeLevel > 8) {
          errors.push(`Employee ${e.employeeNumber}: Invalid apprentice level ${e.apprenticeLevel}`);
        }
      });
    
    return {
      name: 'Apprentice Levels',
      passed: errors.length === 0,
      errors
    };
  }
}
```

### 2. Post-Migration Validation

#### 2.1 Record Count Verification
```typescript
export class RecordCountValidator {
  async validateCounts(): Promise<CountValidation> {
    const sourceCounts = await this.getSourceCounts();
    const targetCounts = await this.getTargetCounts();
    
    return {
      employees: {
        source: sourceCounts.employees,
        target: targetCounts.employees,
        difference: targetCounts.employees - sourceCounts.employees,
        valid: Math.abs(targetCounts.employees - sourceCounts.employees) <= 
               sourceCounts.employees * 0.001 // 0.1% tolerance
      },
      timecards: {
        source: sourceCounts.timecards,
        target: targetCounts.timecards,
        difference: targetCounts.timecards - sourceCounts.timecards,
        valid: targetCounts.timecards >= sourceCounts.timecards * 0.999
      },
      // ... other record types
    };
  }
}
```

#### 2.2 Calculation Verification
```typescript
export class CalculationValidator {
  async validateCalculations(
    sampleSize: number = 100
  ): Promise<CalculationValidation[]> {
    // Select random sample of employees
    const sample = await this.selectRandomSample(sampleSize);
    const validations: CalculationValidation[] = [];
    
    for (const employee of sample) {
      // Get source system calculation
      const sourceCalc = await this.getSourceCalculation(employee);
      
      // Run calculation in new system
      const targetCalc = await this.runTargetCalculation(employee);
      
      // Compare results
      const validation = this.compareCalculations(
        sourceCalc,
        targetCalc,
        employee
      );
      
      validations.push(validation);
    }
    
    return validations;
  }
  
  private compareCalculations(
    source: PayrollCalculation,
    target: PayrollCalculation,
    employee: Employee
  ): CalculationValidation {
    const tolerance = 0.01; // 1 cent
    
    return {
      employeeId: employee.id,
      employeeNumber: employee.employeeNumber,
      period: source.period,
      matches: {
        grossWages: Math.abs(source.grossWages - target.grossWages) <= tolerance,
        federalTax: Math.abs(source.federalTax - target.federalTax) <= tolerance,
        stateTax: Math.abs(source.stateTax - target.stateTax) <= tolerance,
        netPay: Math.abs(source.netPay - target.netPay) <= tolerance
      },
      differences: {
        grossWages: target.grossWages - source.grossWages,
        federalTax: target.federalTax - source.federalTax,
        stateTax: target.stateTax - source.stateTax,
        netPay: target.netPay - source.netPay
      }
    };
  }
}
```

## Migration Execution

### 1. Migration Scripts

#### 1.1 Main Migration Orchestrator
```typescript
export class MigrationOrchestrator {
  async executeMigration(config: MigrationConfig): Promise<MigrationResult> {
    const result = new MigrationResult();
    
    try {
      // Phase 1: Setup
      await this.setupMigration(config);
      result.logPhase('setup', 'completed');
      
      // Phase 2: Extract
      const sourceData = await this.extractData(config);
      result.logPhase('extraction', 'completed', {
        recordCount: sourceData.totalRecords
      });
      
      // Phase 3: Transform
      const transformedData = await this.transformData(sourceData);
      result.logPhase('transformation', 'completed');
      
      // Phase 4: Validate
      const validation = await this.validateData(transformedData);
      if (!validation.passed) {
        throw new MigrationError('Validation failed', validation.errors);
      }
      result.logPhase('validation', 'completed');
      
      // Phase 5: Load
      await this.loadData(transformedData);
      result.logPhase('loading', 'completed');
      
      // Phase 6: Post-validation
      await this.postValidation();
      result.logPhase('post-validation', 'completed');
      
      // Phase 7: Cleanup
      await this.cleanup();
      result.setStatus('success');
      
    } catch (error) {
      result.setStatus('failed', error);
      await this.rollback();
      throw error;
    }
    
    return result;
  }
}
```

#### 1.2 Batch Loading
```typescript
export class BatchLoader {
  private batchSize = 1000;
  private maxRetries = 3;
  
  async loadEmployees(employees: Employee[]): Promise<LoadResult> {
    const total = employees.length;
    let loaded = 0;
    const errors: LoadError[] = [];
    
    for (let i = 0; i < total; i += this.batchSize) {
      const batch = employees.slice(i, i + this.batchSize);
      
      try {
        await this.loadBatch(batch);
        loaded += batch.length;
        
        // Report progress
        this.reportProgress(loaded, total);
        
      } catch (error) {
        // Retry failed batch
        const retryResult = await this.retryBatch(batch);
        loaded += retryResult.loaded;
        errors.push(...retryResult.errors);
      }
    }
    
    return {
      total,
      loaded,
      failed: total - loaded,
      errors
    };
  }
  
  private async loadBatch(batch: Employee[]): Promise<void> {
    await this.db.transaction(async (trx) => {
      for (const employee of batch) {
        // Encrypt sensitive fields
        const encrypted = await this.encryptSensitiveData(employee);
        
        // Insert employee
        await trx.insert('employees', encrypted);
        
        // Insert related data
        await this.insertRelatedData(trx, employee);
      }
    });
  }
}
```

### 2. Rollback Procedures

#### 2.1 Transaction Rollback
```typescript
export class MigrationRollback {
  async rollback(migrationId: string): Promise<void> {
    const migration = await this.getMigration(migrationId);
    
    // Rollback in reverse order
    const phases = migration.completedPhases.reverse();
    
    for (const phase of phases) {
      await this.rollbackPhase(phase);
    }
    
    // Mark migration as rolled back
    await this.updateMigrationStatus(migrationId, 'rolled_back');
  }
  
  private async rollbackPhase(phase: MigrationPhase): Promise<void> {
    switch (phase.name) {
      case 'loading':
        await this.rollbackDataLoad(phase.metadata);
        break;
      case 'transformation':
        await this.rollbackTransformation(phase.metadata);
        break;
      // ... other phases
    }
  }
  
  private async rollbackDataLoad(metadata: any): Promise<void> {
    // Delete migrated records
    await this.db.transaction(async (trx) => {
      // Delete in reverse dependency order
      await trx.delete('payroll_calculations')
        .where('migration_id', metadata.migrationId);
      
      await trx.delete('time_cards')
        .where('migration_id', metadata.migrationId);
      
      await trx.delete('employees')
        .where('migration_id', metadata.migrationId);
    });
  }
}
```

## Common Migration Scenarios

### 1. QuickBooks to ArcPay

#### 1.1 Export Configuration
```typescript
// QuickBooks IIF export mapping
const quickBooksMapping = {
  employee: {
    'NAME': 'fullName',
    'SSN': 'ssn',
    'ADDR1': 'addressLine1',
    'ADDR2': 'addressLine2',
    'CITY': 'city',
    'STATE': 'state',
    'ZIP': 'zipCode',
    'PHONE1': 'phone',
    'EMAIL': 'email',
    'PAYPERIOD': 'payFrequency',
    'STDRATE': 'hourlyRate'
  },
  timecard: {
    'DATE': 'workDate',
    'JOB': 'projectCode',
    'DURATION': 'hours',
    'PAYITEM': 'payType'
  }
};
```

#### 1.2 Special Handling
```typescript
export class QuickBooksMigrator extends BaseMigrator {
  protected parseEmployee(row: any): SourceEmployee {
    // QuickBooks stores name as single field
    const nameParts = this.parseFullName(row.NAME);
    
    // QuickBooks uses different tax codes
    const taxStatus = this.mapQuickBooksTaxStatus(row.FILINGSTATUS);
    
    return {
      ...nameParts,
      ssn: row.SSN,
      address: {
        line1: row.ADDR1,
        line2: row.ADDR2,
        city: row.CITY,
        state: row.STATE,
        zipCode: row.ZIP
      },
      hourlyRate: parseFloat(row.STDRATE),
      federalFilingStatus: taxStatus.federal,
      stateFilingStatus: taxStatus.state,
      // QuickBooks specific
      quickBooksId: row.REFNUM
    };
  }
}
```

### 2. ADP to ArcPay

#### 2.1 ADP API Integration
```typescript
export class ADPMigrator extends BaseMigrator {
  private adpClient: ADPClient;
  
  async extractEmployees(): Promise<SourceEmployee[]> {
    const employees = await this.adpClient.getWorkers({
      includeInactive: true,
      includeDemographics: true,
      includePayroll: true
    });
    
    return employees.map(e => this.mapADPEmployee(e));
  }
  
  private mapADPEmployee(adpEmployee: ADPWorker): SourceEmployee {
    return {
      sourceId: adpEmployee.aoid,
      employeeNumber: adpEmployee.workerID?.idValue,
      ssn: adpEmployee.governmentIDs?.find(
        id => id.nameCode?.codeValue === 'SSN'
      )?.idValue,
      firstName: adpEmployee.person?.legalName?.givenName,
      lastName: adpEmployee.person?.legalName?.familyName1,
      // Map ADP specific fields
      adpPositionId: adpEmployee.workAssignments?.[0]?.positionID
    };
  }
}
```

### 3. Excel/CSV Migration

#### 3.1 Template Generation
```typescript
export class ExcelMigrationHelper {
  generateTemplate(): void {
    const workbook = new ExcelJS.Workbook();
    
    // Employee sheet
    const employeeSheet = workbook.addWorksheet('Employees');
    employeeSheet.columns = [
      { header: 'Employee Number*', key: 'employeeNumber', width: 20 },
      { header: 'SSN*', key: 'ssn', width: 15 },
      { header: 'First Name*', key: 'firstName', width: 20 },
      { header: 'Last Name*', key: 'lastName', width: 20 },
      { header: 'Middle Name', key: 'middleName', width: 20 },
      { header: 'Date of Birth*', key: 'dateOfBirth', width: 15 },
      { header: 'Hire Date*', key: 'hireDate', width: 15 },
      { header: 'Union Local*', key: 'unionLocal', width: 15 },
      { header: 'Classification*', key: 'classification', width: 20 },
      { header: 'Hourly Rate*', key: 'hourlyRate', width: 15 },
      // ... additional columns
    ];
    
    // Add validation
    this.addDataValidation(employeeSheet);
    
    // Add instructions sheet
    this.addInstructionsSheet(workbook);
    
    // Save template
    workbook.xlsx.writeFile('migration_template.xlsx');
  }
  
  private addDataValidation(sheet: ExcelJS.Worksheet): void {
    // SSN format validation
    sheet.getColumn('ssn').eachCell((cell) => {
      cell.dataValidation = {
        type: 'textLength',
        operator: 'equal',
        formula1: 11,
        showErrorMessage: true,
        errorTitle: 'Invalid SSN',
        error: 'SSN must be in format XXX-XX-XXXX'
      };
    });
    
    // Date validation
    ['dateOfBirth', 'hireDate'].forEach(col => {
      sheet.getColumn(col).eachCell((cell) => {
        cell.dataValidation = {
          type: 'date',
          operator: 'between',
          formula1: new Date(1900, 0, 1),
          formula2: new Date(),
          showErrorMessage: true,
          errorTitle: 'Invalid Date',
          error: 'Date must be in MM/DD/YYYY format'
        };
      });
    });
  }
}
```

## Post-Migration Tasks

### 1. Data Reconciliation

#### 1.1 Payroll Reconciliation
```typescript
export class PayrollReconciliation {
  async reconcilePeriod(
    periodEndDate: Date
  ): Promise<ReconciliationReport> {
    // Get source system totals
    const sourceTotals = await this.getSourceTotals(periodEndDate);
    
    // Get target system totals
    const targetTotals = await this.getTargetTotals(periodEndDate);
    
    // Compare totals
    const differences = this.compareTotals(sourceTotals, targetTotals);
    
    // If differences exist, drill down
    if (differences.length > 0) {
      const details = await this.analyzeD

      
    }
    
    return {
      period: periodEndDate,
      sourceTotals,
      targetTotals,
      differences,
      details,
      reconciled: differences.length === 0
    };
  }
}
```

#### 1.2 YTD Verification
```typescript
export class YTDVerification {
  async verifyYTDAmounts(
    year: number
  ): Promise<YTDVerificationResult> {
    const employees = await this.getActiveEmployees();
    const results: EmployeeYTDResult[] = [];
    
    for (const employee of employees) {
      const sourceYTD = await this.getSourceYTD(employee, year);
      const targetYTD = await this.calculateTargetYTD(employee, year);
      
      const comparison = this.compareYTD(sourceYTD, targetYTD);
      
      if (!comparison.matches) {
        results.push({
          employeeId: employee.id,
          employeeNumber: employee.employeeNumber,
          differences: comparison.differences
        });
      }
    }
    
    return {
      year,
      totalEmployees: employees.length,
      discrepancies: results.length,
      details: results
    };
  }
}
```

### 2. User Training

#### 2.1 Migration Documentation
Create comprehensive documentation:
- Data mapping document
- Field transformation rules
- Known differences between systems
- Workaround procedures

#### 2.2 Training Materials
```typescript
export class TrainingMaterialGenerator {
  generateEmployeeGuide(): void {
    const guide = {
      title: 'ArcPay Migration - What Changed',
      sections: [
        {
          title: 'New Employee Numbers',
          content: 'Your employee number may have changed...',
          examples: [
            { old: 'E001', new: 'IW-377-0001' }
          ]
        },
        {
          title: 'Tax Withholding',
          content: 'Review your tax settings...',
          action: 'Verify your W-4 information'
        },
        {
          title: 'Time Entry',
          content: 'New time entry process...',
          screenshots: ['timeentry1.png', 'timeentry2.png']
        }
      ]
    };
    
    this.generatePDF(guide);
  }
}
```

## Migration Checklist

### Pre-Migration
- [ ] Complete source system analysis
- [ ] Create data mapping document
- [ ] Identify data quality issues
- [ ] Develop transformation rules
- [ ] Create test migration plan
- [ ] Backup source system
- [ ] Notify users of migration schedule

### During Migration
- [ ] Run pre-migration validation
- [ ] Execute extraction scripts
- [ ] Transform data according to rules
- [ ] Load data in correct order
- [ ] Run post-migration validation
- [ ] Verify record counts
- [ ] Test sample calculations

### Post-Migration
- [ ] Reconcile payroll totals
- [ ] Verify YTD amounts
- [ ] Run parallel payroll test
- [ ] Train users on new system
- [ ] Document known issues
- [ ] Create support procedures
- [ ] Monitor system performance

### Go-Live
- [ ] Final data synchronization
- [ ] Disable old system access
- [ ] Enable new system for users
- [ ] Monitor first payroll run
- [ ] Address user questions
- [ ] Document lessons learned

## Troubleshooting

### Common Issues

#### 1. Duplicate Records
```sql
-- Find duplicate employees
SELECT ssn, COUNT(*) as count
FROM employees
GROUP BY ssn
HAVING COUNT(*) > 1;

-- Resolution: Merge or mark inactive
UPDATE employees
SET is_active = FALSE
WHERE id IN (
  SELECT MAX(id)
  FROM employees
  GROUP BY ssn
  HAVING COUNT(*) > 1
);
```

#### 2. Missing References
```typescript
// Find orphaned timecards
const orphaned = await db.query(`
  SELECT tc.*
  FROM time_cards tc
  LEFT JOIN employees e ON tc.employee_id = e.id
  WHERE e.id IS NULL
`);

// Resolution: Map to correct employee or delete
for (const timecard of orphaned) {
  const employee = await this.findEmployeeByLegacyId(timecard.legacy_emp_id);
  if (employee) {
    await this.updateTimecard(timecard.id, { employeeId: employee.id });
  } else {
    await this.deleteTimecard(timecard.id);
  }
}
```

#### 3. Calculation Differences
```typescript
// Identify calculation differences
export class CalculationDebugger {
  async debugCalculation(
    employeeId: number,
    periodId: number
  ): Promise<DebugResult> {
    // Get all inputs
    const inputs = await this.gatherInputs(employeeId, periodId);
    
    // Step through calculation
    const steps = await this.stepThroughCalculation(inputs);
    
    // Compare with expected
    const comparison = await this.compareWithExpected(steps);
    
    return {
      inputs,
      steps,
      comparison,
      recommendations: this.generateRecommendations(comparison)
    };
  }
}
```

## Support Resources

### Migration Scripts Repository
Store all migration scripts in version control:
```
migration/
├── extractors/
│   ├── quickbooks/
│   ├── adp/
│   └── generic/
├── transformers/
│   ├── employee.ts
│   ├── timecard.ts
│   └── tax.ts
├── validators/
│   ├── pre-migration.ts
│   └── post-migration.ts
└── loaders/
    ├── batch-loader.ts
    └── rollback.ts
```

### Migration Log Format
```typescript
interface MigrationLog {
  migrationId: string;
  startTime: Date;
  endTime?: Date;
  status: 'running' | 'completed' | 'failed' | 'rolled_back';
  phases: MigrationPhase[];
  statistics: {
    recordsProcessed: number;
    recordsFailed: number;
    warnings: string[];
    errors: string[];
  };
  configuration: MigrationConfig;
}
```