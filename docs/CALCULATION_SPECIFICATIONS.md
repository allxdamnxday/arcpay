# Payroll Calculation Specifications

## Overview

This document provides detailed specifications for all payroll calculations in the Ironworkers Payroll Application. Each calculation includes formulas, edge cases, rounding rules, and compliance requirements.

## 1. Wage Calculations

### 1.1 Base Hourly Rate Determination

#### 1.1.1 Journeyman Rate
```typescript
function getJourneymanRate(
    unionLocalId: number,
    classificationId: number,
    zoneId: number,
    workDate: Date
): number {
    // Find applicable wage rate based on effective date
    const wageRate = findWageRate(unionLocalId, classificationId, zoneId, workDate);
    return wageRate.baseHourlyRate;
}
```

#### 1.1.2 Apprentice Rate
```typescript
function getApprenticeRate(
    journeymanRate: number,
    apprenticeLevel: number,
    wageRate: WageRate
): number {
    const percentageField = `apprentice_percent_level${apprenticeLevel}`;
    const percentage = wageRate[percentageField];
    
    if (!percentage) {
        throw new Error(`No apprentice percentage for level ${apprenticeLevel}`);
    }
    
    return roundToCents(journeymanRate * percentage);
}
```

#### 1.1.3 Shift Differential
```typescript
function applyShiftDifferential(
    baseRate: number,
    shiftNumber: number,
    wageRate: WageRate
): number {
    switch (shiftNumber) {
        case 1:
            return baseRate;
        case 2:
            return roundToCents(baseRate * (1 + wageRate.shift2Differential));
        case 3:
            return roundToCents(baseRate * (1 + wageRate.shift3Differential));
        default:
            throw new Error(`Invalid shift number: ${shiftNumber}`);
    }
}
```

### 1.2 Federal Premium
```typescript
function calculateFederalPremium(
    hours: number,
    isFederalProject: boolean
): number {
    const FEDERAL_PREMIUM_RATE = 9.00; // $9/hour
    
    if (!isFederalProject) {
        return 0;
    }
    
    return roundToCents(hours * FEDERAL_PREMIUM_RATE);
}
```

## 2. Overtime Calculations

### 2.1 Daily Overtime Rules

#### 2.1.1 Monday-Friday Rules
```typescript
function calculateWeekdayOvertime(hoursWorked: number): OvertimeBreakdown {
    const REGULAR_HOURS = 8;
    const OVERTIME_HOURS = 2;
    
    let regular = Math.min(hoursWorked, REGULAR_HOURS);
    let overtime = 0;
    let doubleTime = 0;
    
    if (hoursWorked > REGULAR_HOURS) {
        overtime = Math.min(hoursWorked - REGULAR_HOURS, OVERTIME_HOURS);
        
        if (hoursWorked > REGULAR_HOURS + OVERTIME_HOURS) {
            doubleTime = hoursWorked - REGULAR_HOURS - OVERTIME_HOURS;
        }
    }
    
    return {
        regular: roundToQuarterHour(regular),
        overtime: roundToQuarterHour(overtime),
        doubleTime: roundToQuarterHour(doubleTime)
    };
}
```

#### 2.1.2 Saturday Rules
```typescript
function calculateSaturdayOvertime(hoursWorked: number): OvertimeBreakdown {
    const OVERTIME_HOURS = 8;
    
    let overtime = Math.min(hoursWorked, OVERTIME_HOURS);
    let doubleTime = Math.max(hoursWorked - OVERTIME_HOURS, 0);
    
    return {
        regular: 0,
        overtime: roundToQuarterHour(overtime),
        doubleTime: roundToQuarterHour(doubleTime)
    };
}
```

#### 2.1.3 Sunday/Holiday Rules
```typescript
function calculateSundayHolidayOvertime(hoursWorked: number): OvertimeBreakdown {
    return {
        regular: 0,
        overtime: 0,
        doubleTime: roundToQuarterHour(hoursWorked)
    };
}
```

### 2.2 Special Schedules

#### 2.2.1 Four-Ten Schedule (4 days × 10 hours)
```typescript
function calculateFourTenOvertime(
    dayOfWeek: number,
    hoursWorked: number,
    isScheduledDay: boolean
): OvertimeBreakdown {
    const REGULAR_HOURS = 10;
    const OVERTIME_HOURS = 2;
    
    // If not a scheduled workday, treat as regular overtime
    if (!isScheduledDay) {
        return calculateWeekdayOvertime(hoursWorked);
    }
    
    // Monday-Thursday on 4-10 schedule
    if (dayOfWeek >= 1 && dayOfWeek <= 4) {
        let regular = Math.min(hoursWorked, REGULAR_HOURS);
        let overtime = 0;
        let doubleTime = 0;
        
        if (hoursWorked > REGULAR_HOURS) {
            overtime = Math.min(hoursWorked - REGULAR_HOURS, OVERTIME_HOURS);
            
            if (hoursWorked > REGULAR_HOURS + OVERTIME_HOURS) {
                doubleTime = hoursWorked - REGULAR_HOURS - OVERTIME_HOURS;
            }
        }
        
        return {
            regular: roundToQuarterHour(regular),
            overtime: roundToQuarterHour(overtime),
            doubleTime: roundToQuarterHour(doubleTime)
        };
    }
    
    // Friday on 4-10 schedule is overtime
    return calculateSaturdayOvertime(hoursWorked);
}
```

### 2.3 Weekly Overtime Calculation
```typescript
function calculateWeeklyOvertime(
    weeklyTimeCards: TimeCard[],
    scheduleType: 'standard' | 'four-ten'
): WeeklyOvertimeSummary {
    const WEEKLY_THRESHOLD = scheduleType === 'four-ten' ? 40 : 40;
    let totalRegularHours = 0;
    let weeklyOvertimePool = 0;
    
    // First pass: calculate daily overtime
    const dailyCalculations = weeklyTimeCards.map(tc => {
        const daily = calculateDailyOvertime(tc);
        totalRegularHours += daily.regular;
        return daily;
    });
    
    // Second pass: check for weekly overtime
    if (totalRegularHours > WEEKLY_THRESHOLD) {
        weeklyOvertimePool = totalRegularHours - WEEKLY_THRESHOLD;
        
        // Convert regular hours to overtime, starting from last day
        for (let i = dailyCalculations.length - 1; i >= 0 && weeklyOvertimePool > 0; i--) {
            const conversion = Math.min(dailyCalculations[i].regular, weeklyOvertimePool);
            dailyCalculations[i].regular -= conversion;
            dailyCalculations[i].overtime += conversion;
            weeklyOvertimePool -= conversion;
        }
    }
    
    return {
        dailyCalculations,
        weeklyOvertimeApplied: totalRegularHours > WEEKLY_THRESHOLD
    };
}
```

## 3. Show-Up Pay

### 3.1 Show-Up Pay Rules
```typescript
interface ShowUpPayRules {
    baseAmount: number;
    minimumHoursIfWorked: number;
    extendedMinimumThreshold: number;
    extendedMinimumHours: number;
}

const SHOW_UP_PAY_RULES: ShowUpPayRules = {
    baseAmount: 60.00,
    minimumHoursIfWorked: 4,
    extendedMinimumThreshold: 4,
    extendedMinimumHours: 6
};

function calculateShowUpPay(
    hoursWorked: number,
    hourlyRate: number,
    dayType: 'regular' | 'overtime' | 'sunday'
): ShowUpPayResult {
    // No show-up pay if didn't show up
    if (hoursWorked === 0) {
        return { applicable: false, amount: 0, hours: 0 };
    }
    
    // Calculate base show-up pay amount
    let baseShowUpPay = SHOW_UP_PAY_RULES.baseAmount;
    
    // Adjust for overtime/Sunday days
    if (dayType === 'overtime') {
        baseShowUpPay = baseShowUpPay * 1.5;
    } else if (dayType === 'sunday') {
        baseShowUpPay = baseShowUpPay * 2;
    }
    
    // Determine minimum hours
    let minimumHours = SHOW_UP_PAY_RULES.minimumHoursIfWorked;
    if (hoursWorked >= SHOW_UP_PAY_RULES.extendedMinimumThreshold) {
        minimumHours = SHOW_UP_PAY_RULES.extendedMinimumHours;
    }
    
    // Calculate potential earnings
    const actualEarnings = hoursWorked * hourlyRate;
    const minimumEarnings = Math.max(baseShowUpPay, minimumHours * hourlyRate);
    
    if (actualEarnings < minimumEarnings) {
        return {
            applicable: true,
            amount: roundToCents(minimumEarnings - actualEarnings),
            hours: minimumHours - hoursWorked
        };
    }
    
    return { applicable: false, amount: 0, hours: 0 };
}
```

## 4. Meal Period Penalties

### 4.1 Meal Penalty Calculation
```typescript
function calculateMealPenalty(
    hoursWorked: number,
    mealPeriodTaken: boolean,
    hourlyRate: number,
    overtimeRate: number
): MealPenaltyResult {
    const MEAL_REQUIRED_AFTER = 4.5; // hours
    const PENALTY_RATE_MULTIPLIER = 1.5; // overtime rate
    
    // No penalty if meal period was taken
    if (mealPeriodTaken) {
        return { applicable: false, amount: 0 };
    }
    
    // No penalty if worked less than threshold
    if (hoursWorked <= MEAL_REQUIRED_AFTER) {
        return { applicable: false, amount: 0 };
    }
    
    // Calculate penalty hours (all hours after 4.5)
    const penaltyHours = hoursWorked - MEAL_REQUIRED_AFTER;
    const penaltyRate = hourlyRate * PENALTY_RATE_MULTIPLIER;
    const penaltyAmount = roundToCents(penaltyHours * (penaltyRate - hourlyRate));
    
    return {
        applicable: true,
        amount: penaltyAmount,
        hours: penaltyHours,
        rate: penaltyRate
    };
}
```

## 5. Travel and Subsistence

### 5.1 Travel Zones and Rates
```typescript
interface TravelZone {
    minMiles: number;
    maxMiles: number;
    dailySubsistence: number;
}

const TRAVEL_ZONES: TravelZone[] = [
    { minMiles: 0, maxMiles: 40, dailySubsistence: 0 },
    { minMiles: 40, maxMiles: 60, dailySubsistence: 35 },
    { minMiles: 60, maxMiles: 80, dailySubsistence: 50 },
    { minMiles: 80, maxMiles: 100, dailySubsistence: 65 },
    { minMiles: 100, maxMiles: null, dailySubsistence: 80 }
];

function calculateTravelSubsistence(
    projectLocation: Location,
    unionHallLocation: Location,
    daysWorked: number
): TravelSubsistenceResult {
    const distance = calculateDistance(projectLocation, unionHallLocation);
    
    const zone = TRAVEL_ZONES.find(z => 
        distance >= z.minMiles && (z.maxMiles === null || distance < z.maxMiles)
    );
    
    if (!zone || zone.dailySubsistence === 0) {
        return { applicable: false, distance, amount: 0 };
    }
    
    return {
        applicable: true,
        distance,
        dailyRate: zone.dailySubsistence,
        amount: zone.dailySubsistence * daysWorked
    };
}
```

## 6. Tax Calculations

> **Note**: For detailed implementation data and tables, see:
> - [FED 2025 TAX RETURN GUIDE.md](./FED%202025%20TAX%20RETURN%20GUIDE.md) - Complete federal tax tables and calculations
> - [CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md) - California Method B calculation tables

### 6.1 Federal Income Tax (IRS Publication 15-T)

#### 6.1.1 Method: Percentage Method
```typescript
function calculateFederalIncomeTax(
    grossWages: number,
    payPeriod: PayPeriod,
    filingStatus: FilingStatus,
    allowances: number,
    additionalWithholding: number = 0
): FederalTaxResult {
    // See FED 2025 TAX RETURN GUIDE.md for complete implementation
    // using Worksheet 1A and current tax tables
    // Step 1: Adjust gross wages for pre-tax deductions
    const adjustedWages = grossWages; // Add pre-tax deduction handling
    
    // Step 2: Calculate allowance deduction
    const allowanceAmount = getAllowanceAmount(payPeriod, allowances);
    const wagesAfterAllowances = Math.max(0, adjustedWages - allowanceAmount);
    
    // Step 3: Apply standard deduction
    const standardDeduction = getStandardDeduction(payPeriod, filingStatus);
    const taxableWages = Math.max(0, wagesAfterAllowances - standardDeduction);
    
    // Step 4: Calculate tax using brackets
    const brackets = getFederalTaxBrackets(payPeriod, filingStatus);
    let tax = 0;
    
    for (const bracket of brackets) {
        if (taxableWages > bracket.minIncome) {
            const taxableInBracket = bracket.maxIncome 
                ? Math.min(taxableWages - bracket.minIncome, bracket.maxIncome - bracket.minIncome)
                : taxableWages - bracket.minIncome;
            
            tax += taxableInBracket * bracket.taxRate;
        }
    }
    
    // Step 5: Add additional withholding
    tax += additionalWithholding;
    
    return {
        taxableWages,
        calculatedTax: roundToCents(tax),
        effectiveRate: tax / grossWages
    };
}
```

### 6.2 California State Income Tax (Method B)

> **Implementation Guide**: See [CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md) for complete Method B tables including:
> - Low Income Exemption Table (Table 1)
> - Estimated Deduction Table (Table 2)
> - Standard Deduction Table (Table 3)
> - Exemption Allowance Table (Table 4)
> - Tax Rate Tables (Table 5)

#### 6.2.1 California Withholding
```typescript
function calculateCaliforniaIncomeTax(
    grossWages: number,
    payPeriod: PayPeriod,
    filingStatus: CaliforniaFilingStatus,
    allowances: number,
    additionalWithholding: number = 0
): CaliforniaTaxResult {
    // Step 1: Calculate estimated annual wages
    const annualizedWages = grossWages * getPayPeriodsPerYear(payPeriod);
    
    // Step 2: Apply allowances
    const CALIFORNIA_ALLOWANCE_ANNUAL = 154.00; // 2024 amount
    const allowanceDeduction = allowances * CALIFORNIA_ALLOWANCE_ANNUAL;
    const wagesAfterAllowances = Math.max(0, annualizedWages - allowanceDeduction);
    
    // Step 3: Apply standard deduction
    const standardDeduction = getCaliforniaStandardDeduction(filingStatus);
    const taxableIncome = Math.max(0, wagesAfterAllowances - standardDeduction);
    
    // Step 4: Calculate annual tax
    const brackets = getCaliforniaTaxBrackets(filingStatus);
    let annualTax = 0;
    
    for (const bracket of brackets) {
        if (taxableIncome > bracket.minIncome) {
            const taxableInBracket = bracket.maxIncome 
                ? Math.min(taxableIncome - bracket.minIncome, bracket.maxIncome - bracket.minIncome)
                : taxableIncome - bracket.minIncome;
            
            annualTax += bracket.baseTax + (taxableInBracket * bracket.taxRate);
        }
    }
    
    // Step 5: Convert to pay period
    const periodTax = annualTax / getPayPeriodsPerYear(payPeriod);
    
    // Step 6: Add additional withholding
    const totalTax = periodTax + additionalWithholding;
    
    return {
        taxableIncome: taxableIncome / getPayPeriodsPerYear(payPeriod),
        calculatedTax: roundToCents(totalTax),
        effectiveRate: totalTax / grossWages
    };
}
```

### 6.3 FICA Taxes

#### 6.3.1 Social Security Tax
```typescript
function calculateSocialSecurityTax(
    grossWages: number,
    yearToDateWages: number,
    year: number
): FICATaxResult {
    const SOCIAL_SECURITY_RATE = 0.062; // 6.2%
    const WAGE_BASE_LIMIT = getSocialSecurityWageBase(year); // e.g., $160,200 for 2024
    
    // Calculate remaining wages subject to SS tax
    const remainingWageBase = Math.max(0, WAGE_BASE_LIMIT - yearToDateWages);
    const taxableWages = Math.min(grossWages, remainingWageBase);
    
    const tax = roundToCents(taxableWages * SOCIAL_SECURITY_RATE);
    
    return {
        taxableWages,
        tax,
        rate: SOCIAL_SECURITY_RATE,
        atLimit: yearToDateWages + grossWages >= WAGE_BASE_LIMIT
    };
}
```

#### 6.3.2 Medicare Tax
```typescript
function calculateMedicareTax(
    grossWages: number,
    yearToDateWages: number,
    filingStatus: FilingStatus
): MedicareTaxResult {
    const MEDICARE_RATE = 0.0145; // 1.45%
    const ADDITIONAL_MEDICARE_RATE = 0.009; // 0.9% additional
    const ADDITIONAL_MEDICARE_THRESHOLD = getAdditionalMedicareThreshold(filingStatus);
    
    let tax = grossWages * MEDICARE_RATE;
    
    // Check for additional Medicare tax
    const totalWages = yearToDateWages + grossWages;
    if (totalWages > ADDITIONAL_MEDICARE_THRESHOLD) {
        const additionalTaxableWages = Math.min(
            grossWages,
            totalWages - ADDITIONAL_MEDICARE_THRESHOLD
        );
        tax += additionalTaxableWages * ADDITIONAL_MEDICARE_RATE;
    }
    
    return {
        tax: roundToCents(tax),
        rate: MEDICARE_RATE,
        additionalTaxApplied: totalWages > ADDITIONAL_MEDICARE_THRESHOLD
    };
}
```

### 6.4 California SDI and UI

#### 6.4.1 State Disability Insurance (SDI)
```typescript
function calculateCaliforniaSDI(
    grossWages: number,
    yearToDateWages: number,
    year: number
): SDIResult {
    const SDI_RATE = getSDIRate(year); // e.g., 0.009 for 2024
    const SDI_WAGE_LIMIT = getSDIWageLimit(year); // e.g., $153,164 for 2024
    
    const remainingWageBase = Math.max(0, SDI_WAGE_LIMIT - yearToDateWages);
    const taxableWages = Math.min(grossWages, remainingWageBase);
    
    const tax = roundToCents(taxableWages * SDI_RATE);
    
    return {
        taxableWages,
        tax,
        rate: SDI_RATE,
        atLimit: yearToDateWages + grossWages >= SDI_WAGE_LIMIT
    };
}
```

## 7. Fringe Benefit Calculations

### 7.1 Hourly Fringe Benefits
```typescript
function calculateFringeBenefits(
    hoursWorked: HoursBreakdown,
    wageRate: WageRate
): FringeBenefitResult {
    const totalHours = hoursWorked.regular + hoursWorked.overtime + hoursWorked.doubleTime;
    
    const benefits = {
        healthWelfare: roundToCents(totalHours * wageRate.healthWelfareHourly),
        pension: roundToCents(totalHours * wageRate.pensionHourly),
        vacation: roundToCents(totalHours * wageRate.vacationHourly),
        training: roundToCents(totalHours * wageRate.trainingHourly)
    };
    
    return {
        ...benefits,
        total: roundToCents(
            benefits.healthWelfare + 
            benefits.pension + 
            benefits.vacation + 
            benefits.training
        )
    };
}
```

## 8. Rounding Rules

### 8.1 Time Rounding
```typescript
function roundToQuarterHour(hours: number): number {
    // Round to nearest quarter hour (0.25)
    return Math.round(hours * 4) / 4;
}
```

### 8.2 Money Rounding
```typescript
function roundToCents(amount: number): number {
    // Round to nearest cent using banker's rounding
    return Math.round(amount * 100) / 100;
}
```

### 8.3 Tax Rounding
```typescript
function roundTax(amount: number): number {
    // Always round up for tax withholding (conservative approach)
    return Math.ceil(amount * 100) / 100;
}
```

## 9. Edge Cases and Special Scenarios

### 9.1 Retroactive Pay Adjustments
```typescript
function calculateRetroactivePay(
    originalRate: number,
    newRate: number,
    hoursWorked: number,
    startDate: Date,
    endDate: Date
): RetroactivePayResult {
    const rateDifference = newRate - originalRate;
    const retroPay = roundToCents(hoursWorked * rateDifference);
    
    // Retroactive pay is typically not subject to overtime multipliers
    // but is subject to all taxes
    
    return {
        amount: retroPay,
        taxable: true,
        includeInRegularPay: false
    };
}
```

### 9.2 Holiday Pay on Overtime Day
```typescript
function calculateHolidayPay(
    isHoliday: boolean,
    dayOfWeek: number,
    hoursWorked: number,
    hourlyRate: number
): HolidayPayResult {
    if (!isHoliday) {
        return { holidayPay: 0, overtimeMultiplier: 1 };
    }
    
    // Holidays are always double time
    const holidayPay = roundToCents(hoursWorked * hourlyRate * 2);
    
    // Some contracts pay holiday pay PLUS overtime
    // This would need to be configured per union local
    
    return {
        holidayPay,
        overtimeMultiplier: 2,
        includeRegularPay: false
    };
}
```

### 9.3 Multi-Project Days
```typescript
function calculateMultiProjectDay(
    timeCards: TimeCard[]
): MultiProjectResult {
    // Sort by start time
    const sortedCards = timeCards.sort((a, b) => a.startTime - b.startTime);
    
    let cumulativeHours = 0;
    const results = [];
    
    for (const card of sortedCards) {
        const startHours = cumulativeHours;
        const endHours = cumulativeHours + card.hoursWorked;
        
        // Determine which hours fall into which overtime category
        const breakdown = calculateOvertimeForHourRange(startHours, endHours);
        
        results.push({
            projectId: card.projectId,
            ...breakdown
        });
        
        cumulativeHours = endHours;
    }
    
    return results;
}
```

## 10. Compliance and Validation

### 10.1 Minimum Wage Validation
```typescript
function validateMinimumWage(
    effectiveRate: number,
    location: string,
    date: Date
): ValidationResult {
    const minimumWage = getMinimumWage(location, date);
    
    if (effectiveRate < minimumWage) {
        return {
            valid: false,
            message: `Rate ${effectiveRate} is below minimum wage ${minimumWage}`
        };
    }
    
    return { valid: true };
}
```

### 10.2 Prevailing Wage Compliance
```typescript
function validatePrevailingWage(
    project: Project,
    classification: Classification,
    rate: number
): ValidationResult {
    if (!project.prevailingWageRequired) {
        return { valid: true };
    }
    
    const prevailingRate = getPrevailingWage(
        project.county,
        classification.code,
        project.workDate
    );
    
    if (rate < prevailingRate) {
        return {
            valid: false,
            message: `Rate ${rate} is below prevailing wage ${prevailingRate}`
        };
    }
    
    return { valid: true };
}
```

## 11. Calculation Order and Dependencies

### 11.1 Correct Calculation Sequence
1. Determine applicable wage rate
2. Apply shift differentials
3. Calculate hours breakdown (regular/OT/DT)
4. Calculate gross wages
5. Calculate show-up pay
6. Calculate meal penalties
7. Calculate travel/subsistence
8. Calculate federal premium
9. Sum total gross wages
10. Calculate pre-tax deductions
11. Calculate federal tax
12. Calculate state tax
13. Calculate FICA taxes
14. Calculate SDI
15. Calculate post-tax deductions
16. Calculate net pay
17. Calculate employer-paid fringes

### 11.2 Year-to-Date Tracking
Critical YTD values that must be tracked:
- Gross wages (for tax brackets)
- Social Security wages (for wage base limit)
- Medicare wages (for additional Medicare tax)
- SDI wages (for SDI limit)
- Federal tax withheld
- State tax withheld
- Each fringe benefit contribution

## 12. Testing Requirements

### 12.1 Test Scenarios
Each calculation must be tested with:
1. Minimum values (0 hours, minimum wage)
2. Maximum values (max overtime, max tax brackets)
3. Boundary conditions (exactly 8 hours, exactly at tax bracket)
4. Common scenarios (typical 40-hour week)
5. Edge cases (multiple projects, retroactive pay)

### 12.2 Validation Against Examples
- Use IRS Publication 15-T examples for federal tax
- Use California DE 44 examples for state tax
- Compare against current payroll system
- Validate with union-provided examples

### 12.3 Precision Requirements
- Time: Accurate to quarter hour
- Money: Accurate to the cent
- Percentages: Accurate to 4 decimal places
- No cumulative rounding errors over pay period

## Related Documentation

- [CALCULATION_ENGINE_GUIDE.md](./CALCULATION_ENGINE_GUIDE.md) - Implementation guide for calculation engine
- [FED 2025 TAX RETURN GUIDE.md](./FED%202025%20TAX%20RETURN%20GUIDE.md) - Federal tax withholding tables and calculations
- [CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md) - California tax withholding tables and calculations
- [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) - Database tables for storing calculation data
- [API_SPECIFICATION.md](./API_SPECIFICATION.md) - API endpoints for calculation services