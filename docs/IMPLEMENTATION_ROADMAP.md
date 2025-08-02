# Implementation Roadmap

## Overview

This document provides a detailed roadmap for implementing the Ironworkers Payroll Application, bridging the gap between technical specifications and actual code. It outlines the development phases, dependencies, and specific implementation steps.

**Last Updated**: December 2024  
**Estimated Timeline**: 16-20 weeks  
**Team Size**: 3-5 developers

## Prerequisites

Before starting implementation:

1. ✅ All specification documents reviewed and approved
2. ✅ Development environment setup complete (see DEVELOPMENT_SETUP.md)
3. ✅ Database engine decision made (SQLite vs PostgreSQL)
4. ✅ UI framework and state management approach chosen
5. ✅ Security architecture approved
6. ✅ Testing strategy defined

## Phase 1: Foundation (Weeks 1-3)

### 1.1 Project Setup
```bash
# Initialize project structure
arcpay/
├── src/
│   ├── main/           # Electron main process
│   ├── renderer/       # React application
│   ├── shared/         # Shared types and utilities
│   └── database/       # Database layer
├── tests/
├── docs/
└── scripts/
```

### 1.2 Core Infrastructure

#### Database Layer
1. Implement database connection manager
2. Create migration system
3. Implement tables from [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md):
   - `users` and `user_sessions` (Table 12)
   - `audit_log` (Table 13)
   - `union_locals` (Table 1)
   - `roles` and permissions

#### Security Foundation
1. Implement encryption utilities per [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md) Section 2
2. Create authentication service
3. Implement session management
4. Set up audit logging

#### IPC Architecture
1. Create IPC channel definitions from [API_SPECIFICATION.md](./API_SPECIFICATION.md)
2. Implement secure IPC wrapper
3. Create request/response handlers
4. Add error handling middleware

### 1.3 TypeScript Interfaces

Create TypeScript interfaces mapping to database schema:

```typescript
// src/shared/types/employee.ts
interface Employee {
  id: number;
  employeeNumber: string;
  unionLocalId: number;
  // ... map all fields from DATABASE_SCHEMA.md Table 2
}

// src/shared/types/timecard.ts
interface TimeCard {
  id: number;
  employeeId: number;
  projectId: number;
  workDate: Date;
  // ... map all fields from DATABASE_SCHEMA.md Table 8
}
```

## Phase 2: Employee Management (Weeks 4-5)

### 2.1 Database Implementation
1. Create `employees` table (Table 2)
2. Create `employee_bank_accounts` table (Table 3)
3. Create `classifications` table (Table 4)
4. Implement field-level encryption for SSN and bank details

### 2.2 Business Logic Layer
```typescript
// src/main/services/EmployeeService.ts
class EmployeeService {
  async createEmployee(data: CreateEmployeeDTO): Promise<Employee>
  async updateEmployee(id: number, data: UpdateEmployeeDTO): Promise<Employee>
  async getEmployee(id: number): Promise<Employee>
  async searchEmployees(criteria: SearchCriteria): Promise<Employee[]>
}
```

### 2.3 API Implementation
Implement IPC handlers from [API_SPECIFICATION.md](./API_SPECIFICATION.md) Section 5:
- `employee:list`
- `employee:get`
- `employee:create`
- `employee:update`
- `employee:deactivate`

### 2.4 UI Components
```typescript
// src/renderer/components/employees/
├── EmployeeList.tsx
├── EmployeeForm.tsx
├── EmployeeDetail.tsx
└── EmployeeSearch.tsx
```

## Phase 3: Project & Wage Setup (Weeks 6-7)

### 3.1 Database Implementation
1. Create `wage_zones` table (Table 5)
2. Create `wage_rates` table (Table 6)
3. Create `projects` table (Table 7)
4. Implement zone boundary logic

### 3.2 Wage Calculation Foundation
Based on [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) Section 1:

```typescript
// src/shared/calculations/wages.ts
export class WageCalculator {
  getJourneymanRate(params: WageParams): number
  getApprenticeRate(journeymanRate: number, level: number): number
  applyShiftDifferential(rate: number, shift: number): number
}
```

### 3.3 Zone Detection Service
```typescript
// src/main/services/ZoneService.ts
class ZoneService {
  async detectZoneByAddress(address: Address): Promise<WageZone>
  async getZoneByCounty(county: string): Promise<WageZone>
}
```

## Phase 4: Time Tracking (Weeks 8-9)

### 4.1 Database Implementation
1. Create `time_cards` table (Table 8)
2. Implement unique constraints for date/employee/project
3. Create time card status workflow

### 4.2 Overtime Calculation Engine
Implement from [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) Section 2:

```typescript
// src/shared/calculations/overtime.ts
export class OvertimeCalculator {
  calculateWeekdayOvertime(hours: number): OvertimeBreakdown
  calculateSaturdayOvertime(hours: number): OvertimeBreakdown
  calculateSundayHolidayOvertime(hours: number): OvertimeBreakdown
  calculateFourTenOvertime(params: FourTenParams): OvertimeBreakdown
  calculateWeeklyOvertime(timecards: TimeCard[]): WeeklyOvertimeSummary
}
```

### 4.3 Time Entry UI
```typescript
// src/renderer/components/timecards/
├── TimeCardEntry.tsx      // Daily time entry form
├── TimeCardCalendar.tsx   // Weekly view
├── TimeCardApproval.tsx   // Supervisor approval
└── OvertimeWarning.tsx    // Real-time overtime alerts
```

## Phase 5: Tax Implementation (Weeks 10-12)

### 5.1 Tax Table Import System
1. Create `federal_tax_brackets` table (Table 11)
2. Create `california_tax_brackets` table (Table 11)
3. Implement PDF parser for tax tables
4. Create tax table versioning system

### 5.2 Federal Tax Calculator
Implement [FED 2025 TAX RETURN GUIDE.md](./FED%202025%20TAX%20RETURN%20GUIDE.md):

```typescript
// src/shared/calculations/federalTax.ts
export class FederalTaxCalculator {
  // Implement Worksheet 1A logic
  calculateFederalWithholding(params: FederalTaxParams): TaxResult {
    // Step 1: Adjust employee's wage amount
    // Step 2: Figure tentative withholding amount
    // Step 3: Account for tax credits
    // Step 4: Figure final amount to withhold
  }
}
```

### 5.3 California Tax Calculator
Implement [CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md):

```typescript
// src/shared/calculations/californiaTax.ts
export class CaliforniaTaxCalculator {
  // Implement Method B - Exact Calculation
  calculateCaliforniaWithholding(params: CaliforniaTaxParams): TaxResult {
    // Step 1: Check low-income exemption
    // Step 2: Subtract estimated deduction allowance
    // Step 3: Subtract standard deduction
    // Step 4: Compute tax liability
    // Step 5: Subtract exemption allowance credit
  }
}
```

### 5.4 FICA and SDI Calculations
From [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) Section 6.3-6.4:

```typescript
// src/shared/calculations/ficaTax.ts
export class FICATaxCalculator {
  calculateSocialSecurity(wages: number, ytdWages: number): FICATaxResult
  calculateMedicare(wages: number, ytdWages: number): MedicareTaxResult
  calculateCaliforniaSDI(wages: number, ytdWages: number): SDIResult
}
```

## Phase 6: Payroll Processing (Weeks 13-15)

### 6.1 Database Implementation
1. Create `payroll_periods` table (Table 9)
2. Create `payroll_calculations` table (Table 10)
3. Implement payroll status workflow

### 6.2 Payroll Calculation Engine
Integrate all calculation components:

```typescript
// src/main/services/PayrollCalculationService.ts
class PayrollCalculationService {
  async calculatePayroll(periodId: number): Promise<PayrollResults> {
    // 1. Gather approved time cards
    // 2. Apply wage rates and differentials
    // 3. Calculate overtime
    // 4. Apply special pays (show-up, meal, travel)
    // 5. Calculate gross wages
    // 6. Calculate all taxes
    // 7. Calculate deductions
    // 8. Calculate net pay
    // 9. Calculate fringe benefits
    // 10. Store results
  }
}
```

### 6.3 Special Pay Calculations
From [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) Sections 3-5:

```typescript
// src/shared/calculations/specialPay.ts
export class SpecialPayCalculator {
  calculateShowUpPay(params: ShowUpPayParams): ShowUpPayResult
  calculateMealPenalty(params: MealPenaltyParams): MealPenaltyResult
  calculateTravelSubsistence(params: TravelParams): TravelResult
}
```

### 6.4 Payroll Processing UI
```typescript
// src/renderer/components/payroll/
├── PayrollPeriodList.tsx
├── PayrollPreview.tsx
├── PayrollCalculation.tsx
├── PayrollApproval.tsx
└── PayStubGenerator.tsx
```

## Phase 7: Reporting & Compliance (Weeks 16-17)

### 7.1 Report Generation Engine
From [API_SPECIFICATION.md](./API_SPECIFICATION.md) Section 8:

```typescript
// src/main/services/ReportService.ts
class ReportService {
  async generateCertifiedPayroll(params: ReportParams): Promise<Report>
  async generateUnionRemittance(params: ReportParams): Promise<Report>
  async generateQuarterly941(params: ReportParams): Promise<Report>
  async generateAnnualW2(params: ReportParams): Promise<Report>
}
```

### 7.2 Export Functionality
1. Implement CSV export for all reports
2. Implement PDF generation for pay stubs
3. Create ACH file generator for direct deposits
4. Implement government reporting formats

### 7.3 Compliance Validation
```typescript
// src/shared/validation/compliance.ts
export class ComplianceValidator {
  validateMinimumWage(rate: number, location: string): ValidationResult
  validatePrevailingWage(project: Project, rate: number): ValidationResult
  validateOvertimeCompliance(timecard: TimeCard): ValidationResult
  validateTaxCalculations(calculation: PayrollCalculation): ValidationResult
}
```

## Phase 8: Testing & Security Hardening (Weeks 18-19)

### 8.1 Test Implementation
Following [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) Section 12:

```typescript
// tests/calculations/
├── wages.test.ts
├── overtime.test.ts
├── federalTax.test.ts
├── californiaTax.test.ts
├── specialPay.test.ts
└── compliance.test.ts
```

### 8.2 Security Implementation
Per [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md):

1. Implement penetration testing fixes
2. Add rate limiting to API endpoints
3. Implement data masking for PII
4. Add security headers to all responses
5. Implement secure backup system

### 8.3 Performance Optimization
1. Add database indexes per [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md)
2. Implement caching for tax calculations
3. Optimize large payroll batch processing
4. Add progress indicators for long operations

## Phase 9: Deployment & Training (Week 20)

### 9.1 Deployment Preparation
1. Create installer packages for Windows/Mac
2. Implement auto-update mechanism
3. Create backup and restore procedures
4. Document deployment process

### 9.2 Data Migration
1. Create import scripts for existing data
2. Validate imported data integrity
3. Create rollback procedures
4. Document migration process

### 9.3 User Training
1. Create user training materials
2. Conduct pilot testing with one union local
3. Gather feedback and iterate
4. Create support documentation

## Critical Success Factors

### Code Quality Standards
- 100% TypeScript with strict mode
- ESLint and Prettier configuration
- Minimum 80% test coverage
- All calculations tested against real examples

### Architecture Principles
- Separation of concerns (main/renderer/shared)
- Immutable state management
- Error boundaries for UI stability
- Comprehensive logging

### Security Requirements
- All PII encrypted at rest
- Secure IPC communication
- Regular security audits
- Compliance with all regulations

## Risk Mitigation

### Technical Risks
1. **Tax Calculation Accuracy**: Extensive testing with IRS/CA examples
2. **Performance at Scale**: Load testing with 5000+ employees
3. **Data Migration**: Phased migration with validation
4. **Security Vulnerabilities**: Regular security scanning

### Business Risks
1. **Union Agreement Changes**: Flexible rule engine
2. **Tax Law Updates**: Modular tax calculation system
3. **User Adoption**: Comprehensive training program
4. **Compliance Violations**: Regular compliance audits

## Deliverables by Phase

### Phase 1 Deliverables
- ✅ Working authentication system
- ✅ Basic project structure
- ✅ Database connection layer
- ✅ IPC communication framework

### Phase 2 Deliverables
- ✅ Complete employee management
- ✅ Encrypted PII storage
- ✅ Employee CRUD operations

### Phase 3 Deliverables
- ✅ Project management system
- ✅ Wage rate configuration
- ✅ Zone detection logic

### Phase 4 Deliverables
- ✅ Time card entry system
- ✅ Overtime calculations
- ✅ Approval workflow

### Phase 5 Deliverables
- ✅ Federal tax calculations
- ✅ California tax calculations
- ✅ FICA and SDI calculations

### Phase 6 Deliverables
- ✅ Complete payroll processing
- ✅ Pay stub generation
- ✅ Direct deposit files

### Phase 7 Deliverables
- ✅ All required reports
- ✅ Government compliance files
- ✅ Export functionality

### Phase 8 Deliverables
- ✅ Security hardening complete
- ✅ Performance optimization
- ✅ Comprehensive test suite

### Phase 9 Deliverables
- ✅ Production deployment
- ✅ User training complete
- ✅ Support documentation

## Next Steps

1. Review and approve this roadmap
2. Assign team members to phases
3. Set up development environment
4. Begin Phase 1 implementation
5. Establish weekly progress reviews

## Success Metrics

- **Accuracy**: 100% calculation accuracy vs current system
- **Performance**: < 5 minutes for 1000 employee payroll
- **Reliability**: 99.9% uptime
- **Compliance**: Zero compliance violations
- **User Satisfaction**: > 90% satisfaction score