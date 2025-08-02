# Calculation Engine Implementation Guide

## Overview

This guide provides practical implementation details for building the payroll calculation engine, connecting the specifications in [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) with the tax guides ([CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md) and [FED 2025 TAX RETURN GUIDE.md](./FED%202025%20TAX%20RETURN%20GUIDE.md)) to create a working implementation.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Calculation Engine                      │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │    Wage     │  │  Overtime   │  │  Special    │    │
│  │ Calculator  │  │ Calculator  │  │    Pay      │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Federal   │  │ California  │  │    FICA     │    │
│  │     Tax     │  │     Tax     │  │   & SDI     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Fringe    │  │ Compliance  │  │  Rounding   │    │
│  │  Benefits   │  │ Validator   │  │   Rules     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## Core Implementation Structure

### 1. Base Types and Interfaces

```typescript
// src/shared/types/calculations.ts

export interface Money {
  amount: number;
  currency: 'USD';
}

export interface Hours {
  regular: number;
  overtime: number;
  doubleTime: number;
  total: number;
}

export interface WageRate {
  baseRate: Money;
  overtimeRate: Money;
  doubleTimeRate: Money;
  shift: 1 | 2 | 3;
  effectiveDate: Date;
}

export interface CalculationContext {
  employee: Employee;
  payPeriod: PayrollPeriod;
  timecards: TimeCard[];
  wageRates: WageRate[];
  taxTables: TaxTables;
  ytdAmounts: YearToDateAmounts;
}

export interface CalculationResult {
  gross: GrossWages;
  taxes: TaxCalculations;
  deductions: Deductions;
  net: Money;
  fringes: FringeBenefits;
  metadata: CalculationMetadata;
}
```

### 2. Calculation Engine Core

```typescript
// src/shared/calculations/engine/CalculationEngine.ts

export class CalculationEngine {
  private wageCalculator: WageCalculator;
  private overtimeCalculator: OvertimeCalculator;
  private specialPayCalculator: SpecialPayCalculator;
  private federalTaxCalculator: FederalTaxCalculator;
  private californiaTaxCalculator: CaliforniaTaxCalculator;
  private ficaCalculator: FICACalculator;
  private complianceValidator: ComplianceValidator;

  constructor(private config: CalculationConfig) {
    this.initializeCalculators();
  }

  async calculatePayroll(context: CalculationContext): Promise<CalculationResult> {
    // Step 1: Validate input data
    await this.validateContext(context);

    // Step 2: Calculate hours breakdown
    const hoursBreakdown = await this.calculateHours(context.timecards);

    // Step 3: Calculate base wages
    const baseWages = await this.calculateBaseWages(hoursBreakdown, context);

    // Step 4: Calculate special pays
    const specialPays = await this.calculateSpecialPays(context.timecards, context);

    // Step 5: Calculate gross wages
    const grossWages = this.sumGrossWages(baseWages, specialPays);

    // Step 6: Calculate taxes
    const taxes = await this.calculateTaxes(grossWages, context);

    // Step 7: Calculate deductions
    const deductions = await this.calculateDeductions(grossWages, context);

    // Step 8: Calculate net pay
    const netPay = this.calculateNetPay(grossWages, taxes, deductions);

    // Step 9: Calculate fringe benefits
    const fringes = await this.calculateFringeBenefits(hoursBreakdown, context);

    // Step 10: Validate results
    await this.validateResults(context, { grossWages, taxes, netPay });

    return this.buildResult(grossWages, taxes, deductions, netPay, fringes);
  }
}
```

## Detailed Implementation Guides

### 1. Wage Calculation Implementation

```typescript
// src/shared/calculations/wages/WageCalculator.ts

export class WageCalculator {
  calculateBaseWages(
    hours: Hours,
    wageRate: WageRate,
    context: CalculationContext
  ): BaseWages {
    // Get the applicable rate based on classification and zone
    const baseRate = this.getApplicableRate(context);
    
    // Apply shift differential
    const shiftAdjustedRate = this.applyShiftDifferential(
      baseRate,
      context.shift
    );

    // Calculate wages for each hour type
    const regularWages = this.roundMoney(
      hours.regular * shiftAdjustedRate
    );
    
    const overtimeWages = this.roundMoney(
      hours.overtime * shiftAdjustedRate * 1.5
    );
    
    const doubleTimeWages = this.roundMoney(
      hours.doubleTime * shiftAdjustedRate * 2
    );

    return {
      regular: regularWages,
      overtime: overtimeWages,
      doubleTime: doubleTimeWages,
      total: regularWages + overtimeWages + doubleTimeWages
    };
  }

  private getApplicableRate(context: CalculationContext): number {
    const { employee, project, workDate } = context;
    
    // Find the wage rate based on:
    // 1. Union local
    // 2. Classification (journeyman/apprentice)
    // 3. Zone
    // 4. Effective date
    
    const wageRate = this.wageRateRepository.findApplicableRate({
      unionLocalId: employee.unionLocalId,
      classificationId: employee.classificationId,
      zoneId: project.wageZoneId,
      effectiveDate: workDate
    });

    if (!wageRate) {
      throw new Error(`No wage rate found for employee ${employee.id}`);
    }

    // Apply apprentice percentage if applicable
    if (employee.isApprentice) {
      return this.applyApprenticePercentage(
        wageRate.baseHourlyRate,
        employee.apprenticeLevel,
        wageRate
      );
    }

    return wageRate.baseHourlyRate;
  }
}
```

### 2. Overtime Calculation Implementation

```typescript
// src/shared/calculations/overtime/OvertimeCalculator.ts

export class OvertimeCalculator {
  calculateDailyOvertime(
    timecard: TimeCard,
    scheduleType: ScheduleType
  ): DailyOvertimeResult {
    const dayOfWeek = timecard.workDate.getDay();
    const hoursWorked = timecard.totalHours;

    // Sunday or Holiday - all double time
    if (dayOfWeek === 0 || timecard.isHoliday) {
      return {
        regular: 0,
        overtime: 0,
        doubleTime: this.roundToQuarterHour(hoursWorked)
      };
    }

    // Saturday - first 8 hours overtime, rest double time
    if (dayOfWeek === 6) {
      return this.calculateSaturdayOvertime(hoursWorked);
    }

    // Special handling for 4-10 schedule
    if (scheduleType === ScheduleType.FOUR_TEN) {
      return this.calculateFourTenOvertime(dayOfWeek, hoursWorked);
    }

    // Regular weekday (Monday-Friday)
    return this.calculateWeekdayOvertime(hoursWorked);
  }

  private calculateWeekdayOvertime(hoursWorked: number): DailyOvertimeResult {
    const REGULAR_THRESHOLD = 8;
    const OVERTIME_THRESHOLD = 10;

    let regular = 0;
    let overtime = 0;
    let doubleTime = 0;

    if (hoursWorked <= REGULAR_THRESHOLD) {
      regular = hoursWorked;
    } else if (hoursWorked <= OVERTIME_THRESHOLD) {
      regular = REGULAR_THRESHOLD;
      overtime = hoursWorked - REGULAR_THRESHOLD;
    } else {
      regular = REGULAR_THRESHOLD;
      overtime = 2; // Max 2 hours at 1.5x
      doubleTime = hoursWorked - OVERTIME_THRESHOLD;
    }

    return {
      regular: this.roundToQuarterHour(regular),
      overtime: this.roundToQuarterHour(overtime),
      doubleTime: this.roundToQuarterHour(doubleTime)
    };
  }

  calculateWeeklyOvertime(
    weeklyTimecards: TimeCard[],
    scheduleType: ScheduleType
  ): WeeklyOvertimeResult {
    // First calculate daily overtime for each day
    const dailyResults = weeklyTimecards.map(tc => 
      this.calculateDailyOvertime(tc, scheduleType)
    );

    // Sum up regular hours
    const totalRegularHours = dailyResults.reduce(
      (sum, day) => sum + day.regular, 
      0
    );

    // Check if weekly overtime applies (over 40 hours)
    const WEEKLY_THRESHOLD = 40;
    
    if (totalRegularHours > WEEKLY_THRESHOLD) {
      const weeklyOvertimeHours = totalRegularHours - WEEKLY_THRESHOLD;
      
      // Convert excess regular hours to overtime
      // Starting from the last day of the week, working backwards
      return this.applyWeeklyOvertimeAdjustment(
        dailyResults,
        weeklyOvertimeHours
      );
    }

    return {
      dailyBreakdowns: dailyResults,
      weeklyAdjustment: 0,
      totalHours: this.sumHours(dailyResults)
    };
  }
}
```

### 3. Federal Tax Calculation Implementation

Based on [FED 2025 TAX RETURN GUIDE.md](./FED%202025%20TAX%20RETURN%20GUIDE.md):

```typescript
// src/shared/calculations/tax/FederalTaxCalculator.ts

export class FederalTaxCalculator {
  calculateFederalWithholding(
    grossWages: number,
    payPeriod: PayPeriod,
    w4Info: W4Information,
    ytdWages: number
  ): FederalTaxResult {
    // Implement Worksheet 1A from the Federal Tax Guide
    
    // Step 1: Adjust the employee's wage amount
    const step1 = this.adjustWageAmount(grossWages, payPeriod, w4Info);
    
    // Step 2: Figure the tentative withholding amount
    const step2 = this.calculateTentativeWithholding(
      step1.adjustedAnnualWage,
      w4Info
    );
    
    // Step 3: Account for tax credits
    const step3 = this.applyTaxCredits(step2.tentativeWithholding, w4Info);
    
    // Step 4: Figure the final amount to withhold
    const finalWithholding = this.calculateFinalWithholding(
      step3.afterCredits,
      w4Info.additionalWithholding,
      payPeriod
    );

    return {
      withholdingAmount: this.roundTax(finalWithholding),
      taxableWages: step1.adjustedAnnualWage / this.getPayPeriods(payPeriod),
      effectiveRate: finalWithholding / grossWages,
      calculationDetails: {
        step1,
        step2,
        step3,
        finalAmount: finalWithholding
      }
    };
  }

  private adjustWageAmount(
    grossWages: number,
    payPeriod: PayPeriod,
    w4Info: W4Information
  ): Step1Result {
    // Line 1a: Enter employee's total taxable wages
    const line1a = grossWages;
    
    // Line 1b: Number of pay periods per year
    const line1b = this.getPayPeriods(payPeriod);
    
    // Line 1c: Annual wage amount
    const line1c = line1a * line1b;

    if (w4Info.formVersion >= 2020) {
      // New Form W-4 (2020 or later)
      const line1d = w4Info.step4a || 0; // Other income
      const line1e = line1c + line1d;
      const line1f = w4Info.step4b || 0; // Deductions
      
      // Line 1g: Standard deduction
      const line1g = w4Info.step2Checked ? 0 : 
        (w4Info.filingStatus === 'married_filing_jointly' ? 12900 : 8600);
      
      const line1h = line1f + line1g;
      const line1i = Math.max(0, line1e - line1h);
      
      return {
        adjustedAnnualWage: line1i,
        details: { line1a, line1b, line1c, line1d, line1e, line1f, line1g, line1h, line1i }
      };
    } else {
      // Old Form W-4 (2019 or earlier)
      const line1j = w4Info.allowances || 0;
      const line1k = line1j * 4300;
      const line1l = Math.max(0, line1c - line1k);
      
      return {
        adjustedAnnualWage: line1l,
        details: { line1a, line1b, line1c, line1j, line1k, line1l }
      };
    }
  }

  private calculateTentativeWithholding(
    adjustedAnnualWage: number,
    w4Info: W4Information
  ): Step2Result {
    // Determine which tax table to use
    const useStandardTable = !w4Info.step2Checked || w4Info.formVersion < 2020;
    const taxTable = useStandardTable ? 
      this.getStandardWithholdingTable(w4Info.filingStatus) :
      this.getStep2CheckboxTable(w4Info.filingStatus);

    // Find applicable tax bracket
    const bracket = taxTable.find(b => 
      adjustedAnnualWage > b.min && 
      (b.max === null || adjustedAnnualWage <= b.max)
    );

    if (!bracket) {
      throw new Error('No applicable tax bracket found');
    }

    // Calculate tax
    const taxableOverMin = adjustedAnnualWage - bracket.min;
    const tax = bracket.baseTax + (taxableOverMin * bracket.rate);

    return {
      tentativeWithholding: tax,
      bracket,
      calculation: {
        adjustedWage: adjustedAnnualWage,
        bracketMin: bracket.min,
        taxableOverMin,
        baseTax: bracket.baseTax,
        marginalRate: bracket.rate,
        marginalTax: taxableOverMin * bracket.rate
      }
    };
  }
}
```

### 4. California Tax Calculation Implementation

Based on [CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md):

```typescript
// src/shared/calculations/tax/CaliforniaTaxCalculator.ts

export class CaliforniaTaxCalculator {
  calculateCaliforniaWithholding(
    grossWages: number,
    payPeriod: PayPeriod,
    caW4Info: CaliforniaW4Info,
    ytdWages: number
  ): CaliforniaTaxResult {
    // Implement Method B - Exact Calculation from CA Tax Guide
    
    // Step 1: Check low-income exemption
    const lowIncomeExempt = this.checkLowIncomeExemption(
      grossWages,
      payPeriod,
      caW4Info
    );
    
    if (lowIncomeExempt) {
      return {
        withholdingAmount: 0,
        taxableWages: 0,
        effectiveRate: 0,
        lowIncomeExempt: true
      };
    }

    // Step 2: Subtract estimated deduction allowance
    const afterEstimatedDeduction = this.applyEstimatedDeduction(
      grossWages,
      payPeriod,
      caW4Info.estimatedDeductions
    );

    // Step 3: Subtract standard deduction
    const afterStandardDeduction = this.applyStandardDeduction(
      afterEstimatedDeduction,
      payPeriod,
      caW4Info.filingStatus
    );

    // Step 4: Compute tax liability
    const taxLiability = this.computeTaxLiability(
      afterStandardDeduction,
      payPeriod,
      caW4Info.filingStatus
    );

    // Step 5: Subtract exemption allowance credit
    const finalTax = this.applyExemptionCredit(
      taxLiability,
      payPeriod,
      caW4Info.allowances
    );

    return {
      withholdingAmount: this.roundTax(Math.max(0, finalTax)),
      taxableWages: afterStandardDeduction,
      effectiveRate: finalTax / grossWages,
      calculationSteps: {
        grossWages,
        afterEstimatedDeduction,
        afterStandardDeduction,
        taxLiability,
        exemptionCredit: taxLiability - finalTax,
        finalTax
      }
    };
  }

  private checkLowIncomeExemption(
    grossWages: number,
    payPeriod: PayPeriod,
    caW4Info: CaliforniaW4Info
  ): boolean {
    const exemptionTable = this.getLowIncomeExemptionTable();
    const threshold = exemptionTable[payPeriod][caW4Info.filingStatus];
    
    return grossWages <= threshold;
  }

  private computeTaxLiability(
    taxableIncome: number,
    payPeriod: PayPeriod,
    filingStatus: CaliforniaFilingStatus
  ): number {
    const taxTable = this.getCaliforniaTaxTable(payPeriod, filingStatus);
    
    // Find applicable bracket
    const bracket = taxTable.find(b => 
      taxableIncome > b.min && 
      (b.max === null || taxableIncome <= b.max)
    );

    if (!bracket) {
      return 0; // No tax if below minimum
    }

    // Calculate tax using bracket information
    const taxableOverMin = taxableIncome - bracket.min;
    const tax = bracket.baseTax + (taxableOverMin * bracket.rate);

    return tax;
  }
}
```

### 5. Special Pay Calculations

```typescript
// src/shared/calculations/special/SpecialPayCalculator.ts

export class SpecialPayCalculator {
  calculateShowUpPay(
    hoursWorked: number,
    hourlyRate: number,
    dayType: DayType
  ): ShowUpPayResult {
    const rules = this.getShowUpPayRules();
    
    // No show-up pay if didn't show up
    if (hoursWorked === 0) {
      return { applicable: false, amount: 0, hours: 0 };
    }

    // Calculate base show-up pay
    let baseShowUpPay = rules.baseAmount; // $60
    
    // Adjust for overtime/Sunday
    if (dayType === DayType.SATURDAY) {
      baseShowUpPay *= 1.5;
    } else if (dayType === DayType.SUNDAY || dayType === DayType.HOLIDAY) {
      baseShowUpPay *= 2;
    }

    // Determine minimum hours guarantee
    let minimumHours = rules.minimumHoursIfWorked; // 4 hours
    if (hoursWorked >= rules.extendedThreshold) { // >= 4 hours
      minimumHours = rules.extendedMinimumHours; // 6 hours
    }

    // Calculate earnings
    const actualEarnings = hoursWorked * hourlyRate;
    const minimumEarnings = Math.max(baseShowUpPay, minimumHours * hourlyRate);

    if (actualEarnings < minimumEarnings) {
      return {
        applicable: true,
        amount: this.roundMoney(minimumEarnings - actualEarnings),
        hours: minimumHours - hoursWorked,
        reason: `Minimum ${minimumHours} hours guaranteed`
      };
    }

    return { applicable: false, amount: 0, hours: 0 };
  }

  calculateMealPenalty(
    timecard: TimeCard,
    hourlyRate: number
  ): MealPenaltyResult {
    const MEAL_REQUIRED_AFTER = 4.5; // hours
    const PENALTY_MULTIPLIER = 1.5; // overtime rate
    
    // No penalty if meal period was taken
    if (timecard.mealPeriodTaken) {
      return { applicable: false, amount: 0, hours: 0 };
    }

    // No penalty if worked less than threshold
    if (timecard.totalHours <= MEAL_REQUIRED_AFTER) {
      return { applicable: false, amount: 0, hours: 0 };
    }

    // Calculate penalty
    const penaltyHours = timecard.totalHours - MEAL_REQUIRED_AFTER;
    const regularPay = penaltyHours * hourlyRate;
    const penaltyPay = penaltyHours * hourlyRate * PENALTY_MULTIPLIER;
    const penaltyAmount = penaltyPay - regularPay;

    return {
      applicable: true,
      amount: this.roundMoney(penaltyAmount),
      hours: penaltyHours,
      rate: hourlyRate * PENALTY_MULTIPLIER,
      reason: 'No meal period after 4.5 hours'
    };
  }

  calculateTravelSubsistence(
    project: Project,
    employee: Employee,
    daysWorked: number
  ): TravelSubsistenceResult {
    // Calculate distance from union hall to project
    const distance = this.calculateDistance(
      employee.unionHallLocation,
      project.location
    );

    // Find applicable travel zone
    const zone = this.getTravelZone(distance);
    
    if (!zone || zone.dailyRate === 0) {
      return {
        applicable: false,
        distance,
        dailyRate: 0,
        totalAmount: 0
      };
    }

    return {
      applicable: true,
      distance,
      zone: zone.name,
      dailyRate: zone.dailyRate,
      daysWorked,
      totalAmount: this.roundMoney(zone.dailyRate * daysWorked)
    };
  }
}
```

### 6. Calculation Orchestration

```typescript
// src/shared/calculations/orchestrator/PayrollOrchestrator.ts

export class PayrollOrchestrator {
  async processPayrollPeriod(
    periodId: number,
    options: ProcessingOptions
  ): Promise<PayrollPeriodResult> {
    const period = await this.payrollPeriodRepo.findById(periodId);
    const employees = await this.getEmployeesForPeriod(period);
    
    const results: EmployeePayrollResult[] = [];
    const errors: ProcessingError[] = [];

    // Process each employee
    for (const employee of employees) {
      try {
        const result = await this.processEmployee(employee, period);
        results.push(result);
      } catch (error) {
        errors.push({
          employeeId: employee.id,
          employeeName: `${employee.firstName} ${employee.lastName}`,
          error: error.message,
          stack: error.stack
        });
      }
    }

    // Generate summary
    const summary = this.generatePeriodSummary(results);

    // Save results to database
    await this.savePayrollResults(period, results);

    return {
      periodId,
      processedCount: results.length,
      errorCount: errors.length,
      summary,
      results: options.includeDetails ? results : undefined,
      errors
    };
  }

  private async processEmployee(
    employee: Employee,
    period: PayrollPeriod
  ): Promise<EmployeePayrollResult> {
    // 1. Get approved timecards
    const timecards = await this.getApprovedTimecards(employee.id, period);
    
    if (timecards.length === 0) {
      return this.createZeroPayrollResult(employee, period);
    }

    // 2. Get wage rates
    const wageRates = await this.getWageRates(employee, timecards);

    // 3. Get YTD amounts for tax calculations
    const ytdAmounts = await this.getYTDAmounts(employee.id, period.year);

    // 4. Build calculation context
    const context: CalculationContext = {
      employee,
      payPeriod: period,
      timecards,
      wageRates,
      taxTables: await this.taxTableService.getActiveTables(),
      ytdAmounts
    };

    // 5. Run calculation engine
    const calculation = await this.calculationEngine.calculatePayroll(context);

    // 6. Create payroll record
    return {
      employeeId: employee.id,
      periodId: period.id,
      calculation,
      status: 'calculated',
      calculatedAt: new Date()
    };
  }
}
```

## Testing Strategy

### 1. Unit Tests for Each Calculator

```typescript
// tests/calculations/overtime.test.ts

describe('OvertimeCalculator', () => {
  let calculator: OvertimeCalculator;

  beforeEach(() => {
    calculator = new OvertimeCalculator();
  });

  describe('weekday overtime', () => {
    test('regular 8 hour day has no overtime', () => {
      const result = calculator.calculateWeekdayOvertime(8);
      expect(result).toEqual({
        regular: 8,
        overtime: 0,
        doubleTime: 0
      });
    });

    test('10 hour day has 2 hours overtime', () => {
      const result = calculator.calculateWeekdayOvertime(10);
      expect(result).toEqual({
        regular: 8,
        overtime: 2,
        doubleTime: 0
      });
    });

    test('12 hour day has 2 hours OT and 2 hours DT', () => {
      const result = calculator.calculateWeekdayOvertime(12);
      expect(result).toEqual({
        regular: 8,
        overtime: 2,
        doubleTime: 2
      });
    });
  });

  describe('weekly overtime', () => {
    test('applies weekly OT after 40 regular hours', () => {
      const timecards = [
        { totalHours: 10, workDate: new Date('2024-12-09') }, // Mon: 8R + 2OT
        { totalHours: 10, workDate: new Date('2024-12-10') }, // Tue: 8R + 2OT
        { totalHours: 10, workDate: new Date('2024-12-11') }, // Wed: 8R + 2OT
        { totalHours: 10, workDate: new Date('2024-12-12') }, // Thu: 8R + 2OT
        { totalHours: 10, workDate: new Date('2024-12-13') }, // Fri: 8R + 2OT
      ];

      const result = calculator.calculateWeeklyOvertime(
        timecards,
        ScheduleType.STANDARD
      );

      // 40 regular hours total, but 8 should convert to OT
      expect(result.totalHours.regular).toBe(32);
      expect(result.totalHours.overtime).toBe(18); // 10 daily + 8 weekly
    });
  });
});
```

### 2. Integration Tests

```typescript
// tests/integration/payroll-calculation.test.ts

describe('Payroll Calculation Integration', () => {
  test('complete payroll calculation matches expected result', async () => {
    // Setup test data
    const employee = createTestEmployee({
      hourlyRate: 45.00,
      federalAllowances: 2,
      stateAllowances: 1,
      filingStatus: 'single'
    });

    const timecards = [
      createTimecard({ hours: 8, date: '2024-12-09' }),
      createTimecard({ hours: 10, date: '2024-12-10' }),
      createTimecard({ hours: 8, date: '2024-12-11' }),
      createTimecard({ hours: 12, date: '2024-12-12' }),
      createTimecard({ hours: 6, date: '2024-12-13' })
    ];

    // Run calculation
    const result = await calculationEngine.calculatePayroll({
      employee,
      timecards,
      // ... other context
    });

    // Verify results
    expect(result.gross.total).toBeCloseTo(2115.00); // 40R + 4OT
    expect(result.taxes.federal).toBeCloseTo(234.56);
    expect(result.taxes.california).toBeCloseTo(123.45);
    expect(result.net).toBeCloseTo(1523.45);
  });
});
```

### 3. Compliance Tests

```typescript
// tests/compliance/tax-calculations.test.ts

describe('Tax Calculation Compliance', () => {
  test('federal tax matches IRS examples', async () => {
    // Test against actual IRS Publication 15-T examples
    const irsExamples = loadIRSTestCases();
    
    for (const example of irsExamples) {
      const result = await federalCalculator.calculate(example.input);
      expect(result.withholding).toBeCloseTo(
        example.expectedWithholding,
        2 // cents precision
      );
    }
  });

  test('california tax matches EDD examples', async () => {
    // Test against California DE 44 examples
    const eddExamples = loadCaliforniaTestCases();
    
    for (const example of eddExamples) {
      const result = await californiaCalculator.calculate(example.input);
      expect(result.withholding).toBeCloseTo(
        example.expectedWithholding,
        2
      );
    }
  });
});
```

## Performance Considerations

### 1. Caching Strategy

```typescript
// src/shared/calculations/cache/CalculationCache.ts

export class CalculationCache {
  private taxBracketCache = new Map<string, TaxBracket[]>();
  private wageRateCache = new Map<string, WageRate>();
  
  getCachedTaxBrackets(
    year: number,
    filingStatus: string,
    payPeriod: PayPeriod
  ): TaxBracket[] | undefined {
    const key = `${year}-${filingStatus}-${payPeriod}`;
    return this.taxBracketCache.get(key);
  }

  setCachedTaxBrackets(
    year: number,
    filingStatus: string,
    payPeriod: PayPeriod,
    brackets: TaxBracket[]
  ): void {
    const key = `${year}-${filingStatus}-${payPeriod}`;
    this.taxBracketCache.set(key, brackets);
  }
}
```

### 2. Batch Processing

```typescript
// src/shared/calculations/batch/BatchProcessor.ts

export class BatchProcessor {
  async processBatch(
    employees: Employee[],
    period: PayrollPeriod,
    options: BatchOptions
  ): Promise<BatchResult> {
    const batchSize = options.batchSize || 100;
    const results: CalculationResult[] = [];
    
    // Process in batches to avoid memory issues
    for (let i = 0; i < employees.length; i += batchSize) {
      const batch = employees.slice(i, i + batchSize);
      
      const batchResults = await Promise.all(
        batch.map(emp => this.processEmployee(emp, period))
      );
      
      results.push(...batchResults);
      
      // Report progress
      if (options.onProgress) {
        options.onProgress({
          processed: results.length,
          total: employees.length,
          percentage: (results.length / employees.length) * 100
        });
      }
    }

    return {
      results,
      totalProcessed: results.length,
      duration: Date.now() - startTime
    };
  }
}
```

## Error Handling

### 1. Calculation Errors

```typescript
// src/shared/calculations/errors/CalculationError.ts

export class CalculationError extends Error {
  constructor(
    message: string,
    public code: string,
    public context: any,
    public recoverable: boolean = false
  ) {
    super(message);
    this.name = 'CalculationError';
  }
}

export class WageRateNotFoundError extends CalculationError {
  constructor(employee: Employee, date: Date) {
    super(
      `No wage rate found for employee ${employee.id} on ${date}`,
      'WAGE_RATE_NOT_FOUND',
      { employeeId: employee.id, date },
      false
    );
  }
}

export class InvalidOvertimeError extends CalculationError {
  constructor(hours: number, reason: string) {
    super(
      `Invalid overtime calculation: ${reason}`,
      'INVALID_OVERTIME',
      { hours, reason },
      true
    );
  }
}
```

### 2. Recovery Strategies

```typescript
// src/shared/calculations/recovery/RecoveryHandler.ts

export class RecoveryHandler {
  async handleCalculationError(
    error: CalculationError,
    context: CalculationContext
  ): Promise<RecoveryResult> {
    switch (error.code) {
      case 'WAGE_RATE_NOT_FOUND':
        // Try to find the most recent wage rate
        const fallbackRate = await this.findFallbackWageRate(context);
        if (fallbackRate) {
          return {
            recovered: true,
            action: 'USED_FALLBACK_RATE',
            data: fallbackRate
          };
        }
        break;

      case 'TAX_TABLE_MISSING':
        // Use previous year's tax table with warning
        const previousTable = await this.getPreviousYearTaxTable(context);
        if (previousTable) {
          return {
            recovered: true,
            action: 'USED_PREVIOUS_YEAR_TABLE',
            warning: 'Using previous year tax table',
            data: previousTable
          };
        }
        break;
    }

    return {
      recovered: false,
      error: error.message
    };
  }
}
```

## Debugging and Logging

```typescript
// src/shared/calculations/logging/CalculationLogger.ts

export class CalculationLogger {
  logCalculation(
    employeeId: number,
    calculation: CalculationResult,
    context: CalculationContext
  ): void {
    const log: CalculationLog = {
      timestamp: new Date(),
      employeeId,
      periodId: context.payPeriod.id,
      input: {
        timecards: context.timecards.map(tc => ({
          date: tc.workDate,
          hours: tc.totalHours,
          project: tc.projectId
        })),
        wageRate: context.wageRates[0]?.baseRate,
        ytdAmounts: context.ytdAmounts
      },
      calculations: {
        hours: calculation.gross.hoursBreakdown,
        gross: calculation.gross.total,
        taxes: calculation.taxes,
        deductions: calculation.deductions,
        net: calculation.net
      },
      metadata: {
        calculationVersion: '1.0.0',
        duration: calculation.metadata.duration,
        warnings: calculation.metadata.warnings
      }
    };

    // Store for debugging and audit
    this.auditLogger.log('PAYROLL_CALCULATION', log);
  }
}
```

## Common Pitfalls and Solutions

### 1. Rounding Errors
- Always round at the last step
- Use consistent rounding rules (banker's rounding)
- Test with edge cases (e.g., $0.005)

### 2. Date/Time Issues
- Always use UTC for calculations
- Handle timezone conversions at UI layer only
- Test with daylight saving time transitions

### 3. YTD Tracking
- Update YTD amounts after each payroll
- Handle mid-year employee starts
- Track separately for each tax type

### 4. Concurrent Modifications
- Lock employee records during calculation
- Version wage rates and tax tables
- Handle race conditions in batch processing

## Next Steps

1. Implement base calculation classes
2. Create comprehensive test suite
3. Build performance benchmarks
4. Document calculation audit trail
5. Create debugging tools