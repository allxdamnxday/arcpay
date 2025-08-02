# Testing Guide

## Overview

This guide provides a comprehensive testing strategy for the Ironworkers Payroll Application. Given the critical nature of payroll calculations and compliance requirements, thorough testing is essential. This document outlines test strategies, test data setup, and specific test scenarios.

## Testing Philosophy

- **Accuracy is Non-Negotiable**: Payroll calculations must be 100% accurate
- **Test Everything**: From unit tests to end-to-end scenarios
- **Real-World Data**: Use actual union agreements and tax examples
- **Compliance Focus**: Ensure all legal requirements are met
- **Performance Matters**: Test with realistic data volumes

## Test Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Test Suite                         │
├─────────────────────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │   Unit     │  │Integration │  │    E2E     │   │
│  │   Tests    │  │   Tests    │  │   Tests    │   │
│  └────────────┘  └────────────┘  └────────────┘   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │Compliance  │  │Performance │  │  Security  │   │
│  │   Tests    │  │   Tests    │  │   Tests    │   │
│  └────────────┘  └────────────┘  └────────────┘   │
└─────────────────────────────────────────────────────┘
```

## 1. Unit Testing

### 1.1 Calculation Tests

#### Wage Calculations
```typescript
// tests/unit/calculations/wages.test.ts

describe('WageCalculator', () => {
  let calculator: WageCalculator;
  
  beforeEach(() => {
    calculator = new WageCalculator();
  });

  describe('base wage calculations', () => {
    test('calculates journeyman rate correctly', () => {
      const result = calculator.getJourneymanRate({
        baseRate: 45.50,
        hours: 8,
        shift: 1
      });
      expect(result).toBe(364.00); // 45.50 * 8
    });

    test('applies 2nd shift differential', () => {
      const result = calculator.applyShiftDifferential(45.50, 2, 0.06);
      expect(result).toBe(48.23); // 45.50 * 1.06
    });

    test('applies 3rd shift differential', () => {
      const result = calculator.applyShiftDifferential(45.50, 3, 0.13);
      expect(result).toBe(51.415);
      expect(calculator.roundToCents(result)).toBe(51.42);
    });

    test('calculates apprentice rates', () => {
      const journeymanRate = 45.50;
      const testCases = [
        { level: 1, percentage: 0.50, expected: 22.75 },
        { level: 2, percentage: 0.55, expected: 25.025 },
        { level: 3, percentage: 0.60, expected: 27.30 },
        { level: 4, percentage: 0.65, expected: 29.575 },
        { level: 5, percentage: 0.75, expected: 34.125 },
        { level: 6, percentage: 0.80, expected: 36.40 },
        { level: 7, percentage: 0.85, expected: 38.675 },
        { level: 8, percentage: 0.95, expected: 43.225 }
      ];

      testCases.forEach(({ level, percentage, expected }) => {
        const result = calculator.getApprenticeRate(journeymanRate, level, percentage);
        expect(result).toBeCloseTo(expected, 2);
      });
    });
  });

  describe('zone-based rates', () => {
    test('returns correct rate for each zone', () => {
      const zones = [
        { zone: 1, baseRate: 45.50 },
        { zone: 2, baseRate: 44.75 },
        { zone: 3, baseRate: 43.25 },
        { zone: 4, baseRate: 42.00 },
        { zone: 5, baseRate: 41.50 }
      ];

      zones.forEach(({ zone, baseRate }) => {
        const result = calculator.getZoneRate(zone, 'journeyman');
        expect(result).toBe(baseRate);
      });
    });
  });
});
```

#### Overtime Calculations
```typescript
// tests/unit/calculations/overtime.test.ts

describe('OvertimeCalculator', () => {
  let calculator: OvertimeCalculator;

  beforeEach(() => {
    calculator = new OvertimeCalculator();
  });

  describe('daily overtime - Monday through Friday', () => {
    const testCases = [
      { hours: 6, expected: { regular: 6, overtime: 0, doubleTime: 0 } },
      { hours: 8, expected: { regular: 8, overtime: 0, doubleTime: 0 } },
      { hours: 9, expected: { regular: 8, overtime: 1, doubleTime: 0 } },
      { hours: 10, expected: { regular: 8, overtime: 2, doubleTime: 0 } },
      { hours: 11, expected: { regular: 8, overtime: 2, doubleTime: 1 } },
      { hours: 12, expected: { regular: 8, overtime: 2, doubleTime: 2 } },
      { hours: 14, expected: { regular: 8, overtime: 2, doubleTime: 4 } }
    ];

    testCases.forEach(({ hours, expected }) => {
      test(`${hours} hours worked`, () => {
        const result = calculator.calculateWeekdayOvertime(hours);
        expect(result).toEqual(expected);
      });
    });
  });

  describe('Saturday overtime', () => {
    const testCases = [
      { hours: 4, expected: { regular: 0, overtime: 4, doubleTime: 0 } },
      { hours: 8, expected: { regular: 0, overtime: 8, doubleTime: 0 } },
      { hours: 10, expected: { regular: 0, overtime: 8, doubleTime: 2 } },
      { hours: 12, expected: { regular: 0, overtime: 8, doubleTime: 4 } }
    ];

    testCases.forEach(({ hours, expected }) => {
      test(`${hours} hours on Saturday`, () => {
        const result = calculator.calculateSaturdayOvertime(hours);
        expect(result).toEqual(expected);
      });
    });
  });

  describe('Sunday/Holiday overtime', () => {
    test('all hours are double time', () => {
      const testHours = [4, 8, 10, 12];
      
      testHours.forEach(hours => {
        const result = calculator.calculateSundayHolidayOvertime(hours);
        expect(result).toEqual({
          regular: 0,
          overtime: 0,
          doubleTime: hours
        });
      });
    });
  });

  describe('4-10 schedule', () => {
    test('Monday-Thursday allows 10 regular hours', () => {
      const result = calculator.calculateFourTenOvertime(1, 10, true); // Monday
      expect(result).toEqual({
        regular: 10,
        overtime: 0,
        doubleTime: 0
      });
    });

    test('Friday on 4-10 schedule is all overtime', () => {
      const result = calculator.calculateFourTenOvertime(5, 8, true); // Friday
      expect(result).toEqual({
        regular: 0,
        overtime: 8,
        doubleTime: 0
      });
    });
  });

  describe('weekly overtime', () => {
    test('converts regular hours to overtime after 40', () => {
      const timecards = Array(5).fill(null).map((_, i) => ({
        workDate: new Date(2024, 11, 9 + i), // Mon-Fri
        totalHours: 9, // 9 hours per day = 45 total
        dayOfWeek: i + 1
      }));

      const result = calculator.calculateWeeklyOvertime(timecards, 'standard');
      
      // Should have 40 regular hours and 5 overtime hours total
      const totalRegular = result.dailyBreakdowns.reduce((sum, d) => sum + d.regular, 0);
      const totalOvertime = result.dailyBreakdowns.reduce((sum, d) => sum + d.overtime, 0);
      
      expect(totalRegular).toBe(40);
      expect(totalOvertime).toBe(5);
    });
  });
});
```

#### Tax Calculations
```typescript
// tests/unit/calculations/federalTax.test.ts

describe('FederalTaxCalculator', () => {
  let calculator: FederalTaxCalculator;

  beforeEach(() => {
    calculator = new FederalTaxCalculator();
  });

  describe('IRS Publication 15-T Examples', () => {
    // Test against actual IRS examples
    test('Example 1: Single, biweekly, $1,000 wages', () => {
      const result = calculator.calculate({
        grossWages: 1000,
        payPeriod: 'biweekly',
        filingStatus: 'single',
        w4Version: 2020,
        step2Checked: false,
        allowances: 0,
        additionalWithholding: 0
      });

      expect(result.withholding).toBeCloseTo(65.00, 2);
    });

    test('Example 2: Married filing jointly, monthly, $5,000 wages', () => {
      const result = calculator.calculate({
        grossWages: 5000,
        payPeriod: 'monthly',
        filingStatus: 'married_filing_jointly',
        w4Version: 2020,
        step2Checked: false,
        allowances: 0,
        additionalWithholding: 0
      });

      expect(result.withholding).toBeCloseTo(392.00, 2);
    });
  });

  describe('2019 W-4 compatibility', () => {
    test('applies allowance deduction correctly', () => {
      const result = calculator.calculate({
        grossWages: 1000,
        payPeriod: 'biweekly',
        filingStatus: 'single',
        w4Version: 2019,
        allowances: 2,
        additionalWithholding: 0
      });

      // Each allowance = $4,300 annually / 26 biweekly periods = $165.38
      expect(result.adjustedWages).toBeCloseTo(669.24, 2); // 1000 - (2 * 165.38)
    });
  });
});
```

### 1.2 Database Tests

```typescript
// tests/unit/database/employee.test.ts

describe('EmployeeRepository', () => {
  let repo: EmployeeRepository;
  let db: TestDatabase;

  beforeEach(async () => {
    db = await createTestDatabase();
    repo = new EmployeeRepository(db);
  });

  afterEach(async () => {
    await db.cleanup();
  });

  describe('encryption', () => {
    test('encrypts SSN on save', async () => {
      const employee = await repo.create({
        ssn: '123-45-6789',
        firstName: 'John',
        lastName: 'Doe'
      });

      // Direct database query to verify encryption
      const raw = await db.query('SELECT ssn_encrypted FROM employees WHERE id = ?', [employee.id]);
      
      expect(raw[0].ssn_encrypted).not.toBe('123-45-6789');
      expect(raw[0].ssn_encrypted).toMatch(/^[A-Za-z0-9+/]+=*$/); // Base64 pattern
    });

    test('decrypts SSN on read', async () => {
      const created = await repo.create({
        ssn: '123-45-6789',
        firstName: 'John',
        lastName: 'Doe'
      });

      const retrieved = await repo.findById(created.id);
      expect(retrieved.ssn).toBe('123-45-6789');
    });
  });
});
```

## 2. Integration Testing

### 2.1 API Integration Tests

```typescript
// tests/integration/api/payroll.test.ts

describe('Payroll API Integration', () => {
  let app: Application;
  let testData: TestData;

  beforeAll(async () => {
    app = await createTestApplication();
    testData = await setupTestData();
  });

  describe('POST /api/payroll/calculate', () => {
    test('calculates payroll for period correctly', async () => {
      // Create test timecards
      const timecards = await createTestTimecards({
        employeeId: testData.employee.id,
        week: '2024-12-09',
        hours: [8, 10, 8, 12, 6] // Mon-Fri
      });

      const response = await request(app)
        .post('/api/payroll/calculate')
        .send({
          periodId: testData.payrollPeriod.id,
          employeeIds: [testData.employee.id]
        })
        .expect(200);

      const calculation = response.body.calculations[0];
      
      // Verify hours calculation
      expect(calculation.hours).toEqual({
        regular: 40,
        overtime: 4,
        doubleTime: 0
      });

      // Verify gross wages (assuming $45/hour base rate)
      expect(calculation.gross).toEqual({
        regular: 1800.00,    // 40 * 45
        overtime: 270.00,    // 4 * 45 * 1.5
        doubleTime: 0,
        total: 2070.00
      });

      // Verify taxes were calculated
      expect(calculation.taxes.federal).toBeGreaterThan(0);
      expect(calculation.taxes.california).toBeGreaterThan(0);
      expect(calculation.taxes.socialSecurity).toBeCloseTo(128.34, 2); // 2070 * 0.062
      expect(calculation.taxes.medicare).toBeCloseTo(30.02, 2); // 2070 * 0.0145
    });
  });
});
```

### 2.2 Workflow Integration Tests

```typescript
// tests/integration/workflows/timecard-to-payroll.test.ts

describe('Timecard to Payroll Workflow', () => {
  test('complete workflow from timecard entry to pay stub', async () => {
    // Step 1: Create timecard
    const timecard = await timecardService.create({
      employeeId: testEmployee.id,
      projectId: testProject.id,
      workDate: '2024-12-09',
      hours: 10,
      shift: 1
    });

    // Step 2: Approve timecard
    await timecardService.approve(timecard.id, supervisor.id);

    // Step 3: Process payroll
    const payroll = await payrollService.processWeek({
      weekEnding: '2024-12-15',
      employeeIds: [testEmployee.id]
    });

    // Step 4: Generate pay stub
    const payStub = await reportService.generatePayStub(
      payroll.calculations[0].id
    );

    // Verify complete flow
    expect(timecard.status).toBe('approved');
    expect(payroll.status).toBe('processed');
    expect(payStub).toBeDefined();
    expect(payStub.netPay).toBeGreaterThan(0);
  });
});
```

## 3. End-to-End Testing

### 3.1 UI Testing with Playwright

```typescript
// tests/e2e/payroll-entry.test.ts

import { test, expect } from '@playwright/test';

test.describe('Payroll Entry Flow', () => {
  test('payroll clerk can enter and submit timecards', async ({ page }) => {
    // Login
    await page.goto('/login');
    await page.fill('#username', 'payroll.clerk@test.com');
    await page.fill('#password', 'TestPassword123!');
    await page.click('button[type="submit"]');

    // Navigate to timecard entry
    await page.click('nav >> text=Timecards');
    await page.click('button >> text=New Timecard');

    // Fill timecard form
    await page.selectOption('#employee', 'John Doe (E001)');
    await page.selectOption('#project', 'Golden Gate Bridge (P001)');
    await page.fill('#workDate', '2024-12-09');
    await page.fill('#hoursWorked', '10');
    await page.selectOption('#shift', '1');
    await page.check('#mealPeriodTaken');

    // Submit and verify
    await page.click('button >> text=Calculate');
    
    // Verify overtime warning appears
    await expect(page.locator('.overtime-warning')).toContainText('2 hours overtime');
    
    await page.click('button >> text=Save Timecard');
    await expect(page.locator('.success-message')).toContainText('Timecard saved');
  });
});
```

### 3.2 Full Payroll Cycle Test

```typescript
// tests/e2e/full-payroll-cycle.test.ts

test('complete payroll cycle for union local', async ({ page }) => {
  // Test data setup
  const weekEnding = '2024-12-15';
  const employeeCount = 50;

  // Step 1: Enter timecards for the week
  await enterWeeklyTimecards(page, weekEnding, employeeCount);

  // Step 2: Approve all timecards
  await approveTimecards(page, weekEnding);

  // Step 3: Run payroll calculation
  await page.goto('/payroll/process');
  await page.selectOption('#payPeriod', weekEnding);
  await page.click('button >> text=Calculate Payroll');

  // Wait for calculation to complete
  await page.waitForSelector('.calculation-complete', { timeout: 30000 });

  // Step 4: Review and approve payroll
  await page.click('button >> text=Review Results');
  
  // Verify totals
  const totalGross = await page.textContent('.total-gross');
  expect(parseFloat(totalGross.replace(/[^0-9.]/g, ''))).toBeGreaterThan(50000);

  await page.click('button >> text=Approve Payroll');

  // Step 5: Generate outputs
  await page.click('button >> text=Generate Pay Stubs');
  await page.click('button >> text=Create ACH File');
  await page.click('button >> text=Generate Reports');

  // Verify all outputs generated
  await expect(page.locator('.pay-stubs-status')).toContainText('Generated');
  await expect(page.locator('.ach-file-status')).toContainText('Created');
  await expect(page.locator('.reports-status')).toContainText('Complete');
});
```

## 4. Compliance Testing

### 4.1 Tax Calculation Compliance

```typescript
// tests/compliance/tax-calculations.test.ts

describe('Tax Calculation Compliance', () => {
  describe('Federal Tax - IRS Examples', () => {
    // Load official IRS test cases
    const irsTestCases = loadIRSPublicationExamples();

    irsTestCases.forEach((testCase) => {
      test(`IRS Example: ${testCase.description}`, async () => {
        const result = await federalTaxCalculator.calculate(testCase.input);
        
        expect(result.withholding).toBeCloseTo(
          testCase.expectedWithholding,
          0.01 // Within 1 cent
        );
      });
    });
  });

  describe('California Tax - EDD Examples', () => {
    const eddTestCases = loadCaliforniaDE44Examples();

    eddTestCases.forEach((testCase) => {
      test(`EDD Example: ${testCase.description}`, async () => {
        const result = await californiaTaxCalculator.calculate(testCase.input);
        
        expect(result.withholding).toBeCloseTo(
          testCase.expectedWithholding,
          0.01
        );
      });
    });
  });
});
```

### 4.2 Overtime Compliance

```typescript
// tests/compliance/overtime-rules.test.ts

describe('Union Agreement Overtime Compliance', () => {
  describe('Local 377 Agreement', () => {
    test('Saturday makeup day is straight time if under 40 hours', () => {
      const timecards = [
        { date: '2024-12-09', hours: 0 },    // Mon - off
        { date: '2024-12-10', hours: 8 },    // Tue
        { date: '2024-12-11', hours: 8 },    // Wed
        { date: '2024-12-12', hours: 8 },    // Thu
        { date: '2024-12-13', hours: 8 },    // Fri
        { date: '2024-12-14', hours: 8 },    // Sat - makeup day
      ];

      const result = calculator.calculateWeekWithMakeupDay(timecards);
      
      // Total 40 hours, all at straight time
      expect(result.regular).toBe(40);
      expect(result.overtime).toBe(0);
    });
  });
});
```

## 5. Performance Testing

### 5.1 Load Testing

```typescript
// tests/performance/payroll-load.test.ts

describe('Payroll Performance', () => {
  test('processes 1000 employee payroll in under 5 minutes', async () => {
    // Create test data
    const employees = await createTestEmployees(1000);
    const timecards = await createTestTimecards(employees, '2024-12-15');

    const startTime = Date.now();
    
    const result = await payrollService.processPeriod({
      periodId: testPeriod.id,
      employeeIds: employees.map(e => e.id)
    });

    const duration = Date.now() - startTime;

    expect(result.processed).toBe(1000);
    expect(duration).toBeLessThan(5 * 60 * 1000); // 5 minutes
    
    // Log performance metrics
    console.log(`Processed ${result.processed} employees in ${duration}ms`);
    console.log(`Average time per employee: ${duration / result.processed}ms`);
  });

  test('handles concurrent payroll requests', async () => {
    const periods = await createTestPeriods(5);
    
    // Process 5 payroll periods concurrently
    const results = await Promise.all(
      periods.map(period => 
        payrollService.processPeriod({ periodId: period.id })
      )
    );

    // All should complete successfully
    results.forEach(result => {
      expect(result.status).toBe('completed');
      expect(result.errors).toHaveLength(0);
    });
  });
});
```

### 5.2 Database Performance

```typescript
// tests/performance/database-queries.test.ts

describe('Database Query Performance', () => {
  test('employee search completes in under 100ms', async () => {
    // Create 10,000 test employees
    await createTestEmployees(10000);

    const queries = [
      { search: 'Smith' },
      { search: 'John' },
      { classificationId: 1 },
      { unionLocalId: 377 }
    ];

    for (const query of queries) {
      const start = Date.now();
      const results = await employeeRepo.search(query);
      const duration = Date.now() - start;

      expect(duration).toBeLessThan(100);
      console.log(`Query ${JSON.stringify(query)} took ${duration}ms`);
    }
  });
});
```

## 6. Security Testing

### 6.1 Authentication Tests

```typescript
// tests/security/authentication.test.ts

describe('Authentication Security', () => {
  test('prevents brute force attacks', async () => {
    const attempts = [];
    
    // Try 6 failed login attempts
    for (let i = 0; i < 6; i++) {
      attempts.push(
        authService.login('test@example.com', 'wrongpassword')
      );
    }

    const results = await Promise.allSettled(attempts);
    
    // First 5 should fail with invalid credentials
    results.slice(0, 5).forEach(result => {
      expect(result.status).toBe('rejected');
      expect(result.reason.code).toBe('INVALID_CREDENTIALS');
    });

    // 6th should fail with account locked
    expect(results[5].status).toBe('rejected');
    expect(results[5].reason.code).toBe('ACCOUNT_LOCKED');
  });

  test('session expires after idle timeout', async () => {
    const session = await authService.login('test@example.com', 'password');
    
    // Fast-forward time by 16 minutes
    jest.advanceTimersByTime(16 * 60 * 1000);
    
    const isValid = await authService.validateSession(session.token);
    expect(isValid).toBe(false);
  });
});
```

### 6.2 Data Access Tests

```typescript
// tests/security/data-access.test.ts

describe('Data Access Security', () => {
  test('payroll clerk cannot access other union data', async () => {
    const clerk = await createUser({
      role: 'payroll_clerk',
      unionLocalId: 377
    });

    const otherUnionEmployee = await createEmployee({
      unionLocalId: 416 // Different union
    });

    await expect(
      employeeService.getEmployee(otherUnionEmployee.id, clerk)
    ).rejects.toThrow('Access denied');
  });

  test('employee can only see own payroll data', async () => {
    const employee1 = await createEmployee();
    const employee2 = await createEmployee();

    const user1 = await createUser({
      role: 'employee',
      employeeId: employee1.id
    });

    // Can access own data
    const ownPayroll = await payrollService.getPayrollHistory(employee1.id, user1);
    expect(ownPayroll).toBeDefined();

    // Cannot access other employee's data
    await expect(
      payrollService.getPayrollHistory(employee2.id, user1)
    ).rejects.toThrow('Access denied');
  });
});
```

## 7. Test Data Management

### 7.1 Test Data Factory

```typescript
// tests/factories/employee.factory.ts

export class EmployeeFactory {
  static create(overrides: Partial<Employee> = {}): Employee {
    return {
      id: faker.number.int(),
      employeeNumber: faker.string.alphanumeric(6).toUpperCase(),
      firstName: faker.person.firstName(),
      lastName: faker.person.lastName(),
      ssn: this.generateSSN(),
      unionLocalId: 377,
      classificationId: 1,
      hireDate: faker.date.past({ years: 5 }),
      hourlyRate: faker.number.float({ min: 35, max: 55, precision: 0.01 }),
      federalFilingStatus: 'single',
      federalAllowances: faker.number.int({ min: 0, max: 3 }),
      stateFilingStatus: 'single',
      stateAllowances: faker.number.int({ min: 0, max: 2 }),
      ...overrides
    };
  }

  static createBatch(count: number, overrides: Partial<Employee> = {}): Employee[] {
    return Array(count).fill(null).map(() => this.create(overrides));
  }

  private static generateSSN(): string {
    // Generate valid test SSN (not real)
    const area = faker.number.int({ min: 700, max: 728 }); // Test range
    const group = faker.number.int({ min: 10, max: 99 });
    const serial = faker.number.int({ min: 1000, max: 9999 });
    return `${area}-${group}-${serial}`;
  }
}
```

### 7.2 Scenario-Based Test Data

```typescript
// tests/scenarios/overtime-scenarios.ts

export const overtimeScenarios = {
  standard40Hour: [
    { day: 'Mon', hours: 8 },
    { day: 'Tue', hours: 8 },
    { day: 'Wed', hours: 8 },
    { day: 'Thu', hours: 8 },
    { day: 'Fri', hours: 8 }
  ],
  
  dailyOvertime: [
    { day: 'Mon', hours: 10 }, // 2 OT
    { day: 'Tue', hours: 8 },
    { day: 'Wed', hours: 12 }, // 2 OT, 2 DT
    { day: 'Thu', hours: 8 },
    { day: 'Fri', hours: 8 }
  ],
  
  weeklyOvertime: [
    { day: 'Mon', hours: 9 },
    { day: 'Tue', hours: 9 },
    { day: 'Wed', hours: 9 },
    { day: 'Thu', hours: 9 },
    { day: 'Fri', hours: 9 } // 45 total, 5 weekly OT
  ],
  
  saturday: [
    { day: 'Mon', hours: 8 },
    { day: 'Tue', hours: 8 },
    { day: 'Wed', hours: 8 },
    { day: 'Thu', hours: 8 },
    { day: 'Fri', hours: 8 },
    { day: 'Sat', hours: 10 } // 8 OT, 2 DT
  ],
  
  fourTenSchedule: [
    { day: 'Mon', hours: 10 },
    { day: 'Tue', hours: 10 },
    { day: 'Wed', hours: 10 },
    { day: 'Thu', hours: 10 },
    { day: 'Fri', hours: 0 }
  ]
};
```

## 8. Test Environment Setup

### 8.1 Database Setup

```typescript
// tests/setup/database.ts

export async function setupTestDatabase(): Promise<TestDatabase> {
  const db = new TestDatabase({
    type: 'sqlite',
    database: ':memory:',
    synchronize: false,
    logging: false
  });

  // Run migrations
  await db.runMigrations();

  // Load base data
  await loadTaxTables(db);
  await loadWageZones(db);
  await loadUnionLocals(db);

  return db;
}

export async function cleanupDatabase(db: TestDatabase): Promise<void> {
  // Clear all data except static tables
  await db.truncate([
    'employees',
    'time_cards',
    'payroll_calculations',
    'audit_log'
  ]);
}
```

### 8.2 Mock Services

```typescript
// tests/mocks/TaxTableService.mock.ts

export class MockTaxTableService implements ITaxTableService {
  private tables: Map<string, TaxTable> = new Map();

  constructor() {
    this.loadDefaultTables();
  }

  async getFederalTaxTable(year: number, filingStatus: string): Promise<TaxTable> {
    const key = `federal-${year}-${filingStatus}`;
    return this.tables.get(key) || this.createDefaultTable();
  }

  private loadDefaultTables(): void {
    // Load simplified tax tables for testing
    this.tables.set('federal-2025-single', {
      brackets: [
        { min: 0, max: 11000, rate: 0.10, base: 0 },
        { min: 11000, max: 44725, rate: 0.12, base: 1100 },
        // ... more brackets
      ]
    });
  }
}
```

## 9. Continuous Integration

### 9.1 GitHub Actions Configuration

```yaml
# .github/workflows/test.yml

name: Test Suite

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:unit
      - uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: coverage/

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost/test

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npx playwright install
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/

  compliance-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:compliance
```

## 10. Test Reporting

### 10.1 Coverage Requirements

```json
// jest.config.js
module.exports = {
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './src/calculations/': {
      branches: 95,
      functions: 95,
      lines: 95,
      statements: 95
    }
  }
};
```

### 10.2 Test Result Dashboard

```typescript
// tests/reporting/dashboard.ts

export class TestDashboard {
  async generateReport(): Promise<TestReport> {
    const results = await this.collectResults();
    
    return {
      summary: {
        total: results.total,
        passed: results.passed,
        failed: results.failed,
        skipped: results.skipped,
        duration: results.duration
      },
      coverage: {
        lines: results.coverage.lines,
        branches: results.coverage.branches,
        functions: results.coverage.functions,
        statements: results.coverage.statements
      },
      criticalPaths: {
        overtimeCalculation: results.criticalPaths.overtime,
        taxCalculation: results.criticalPaths.tax,
        payrollProcessing: results.criticalPaths.payroll
      },
      complianceStatus: {
        irsExamples: results.compliance.irs,
        californiaExamples: results.compliance.california,
        unionRules: results.compliance.union
      }
    };
  }
}
```

## Best Practices

### 1. Test Organization
- Group tests by feature/module
- Use descriptive test names
- Follow AAA pattern (Arrange, Act, Assert)
- Keep tests independent and idempotent

### 2. Test Data
- Use factories for consistent test data
- Never use production data
- Clean up after each test
- Use realistic data volumes

### 3. Assertions
- Test one thing per test
- Use specific assertions
- Include helpful error messages
- Test both success and failure cases

### 4. Performance
- Run unit tests in parallel
- Use in-memory databases for speed
- Mock external services
- Profile slow tests

### 5. Maintenance
- Keep tests up to date with code
- Remove obsolete tests
- Refactor test code like production code
- Document complex test scenarios

## Troubleshooting

### Common Issues

1. **Flaky Tests**
   - Check for race conditions
   - Ensure proper async/await usage
   - Verify test isolation
   - Add retry logic for network calls

2. **Slow Tests**
   - Profile test execution
   - Use test doubles appropriately
   - Optimize database queries
   - Parallelize where possible

3. **False Positives**
   - Verify assertions are correct
   - Check test data validity
   - Ensure proper cleanup
   - Review mock implementations

4. **Coverage Gaps**
   - Identify untested code paths
   - Add edge case tests
   - Test error conditions
   - Cover all calculation scenarios