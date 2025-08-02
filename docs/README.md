# Ironworkers Payroll Application - Documentation Index

## Overview

This documentation provides comprehensive specifications and analysis for building a desktop payroll application for California ironworkers unions. The application handles complex wage calculations, zone-based pay rates, union-specific overtime rules, and full tax compliance.

**Last Updated**: December 2024
**Version**: 1.1

## Documentation Dependency Graph

```
┌─────────────────────┐
│   PROJECT_ANALYSIS  │ ← Start Here
└──────────┬──────────┘
           │
     ┌─────▼─────┐
     │  README   │ ← This Document
     └─────┬─────┘
           │
    ┌──────┴────────────┬────────────┬─────────────┐
    │                   │            │             │
┌───▼────┐      ┌──────▼────┐  ┌────▼────┐  ┌────▼────┐
│DATABASE│      │CALCULATION │  │   API   │  │SECURITY │
│ SCHEMA │      │   SPECS    │  │  SPEC   │  │COMPLIANCE│
└────────┘      └─────┬──────┘  └─────────┘  └──────────┘
                      │
              ┌───────┴────────┐
              │   TAX GUIDES   │
              │ • CA TAX GUIDE │
              │ • FED TAX GUIDE│
              └────────────────┘
```

## Core Documentation

### 1. [PROJECT_ANALYSIS.md](./PROJECT_ANALYSIS.md)
**Purpose**: High-level project analysis and feasibility assessment

**Key Sections**:
- Executive summary with revised timeline (16-20 weeks)
- Technical architecture assessment
- Critical missing specifications identified
- Risk assessment and mitigation strategies
- Phased development approach
- Cost considerations

**When to Read**: Start here for overall project understanding

### 2. [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md)
**Purpose**: Complete database design with all tables, relationships, and constraints

**Key Sections**:
- 13 core tables with full specifications
- Encryption strategy for sensitive data
- Audit trail implementation
- Performance optimization indexes
- Data integrity rules
- Migration strategy

**When to Read**: Before implementing database layer

### 3. [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md)
**Purpose**: Detailed algorithms for all payroll calculations

**Key Sections**:
- Wage calculations with zone and shift differentials
- Complex overtime rules (daily, weekly, special schedules)
- Show-up pay and meal penalty calculations
- Federal and California tax calculations
- FICA and SDI calculations
- Fringe benefit calculations
- Edge cases and special scenarios

**When to Read**: Before implementing calculation engine

### 4. [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md)
**Purpose**: Security architecture and compliance requirements

**Key Sections**:
- Data classification and encryption requirements
- Authentication and authorization (RBAC)
- Audit logging specifications
- Labor law compliance (FLSA, California Labor Code)
- Tax compliance (IRS, EDD)
- Privacy compliance (CCPA)
- Security testing requirements

**When to Read**: Throughout development, especially for sensitive features

### 5. [API_SPECIFICATION.md](./API_SPECIFICATION.md)
**Purpose**: IPC communication between Electron main and renderer processes

**Key Sections**:
- Channel naming conventions
- Request/response formats
- Complete API for all features
- Error handling standards
- Security considerations
- Performance optimizations
- Testing strategies

**When to Read**: Before implementing UI/backend communication

### 6. [CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md)
**Purpose**: California state tax withholding calculations using Method B

**Key Sections**:
- 5-step calculation process
- Low income exemption tables
- Standard deduction tables
- Tax rate brackets for all filing statuses
- Implementation data tables

**When to Read**: When implementing California tax calculations

### 7. [FED 2025 TAX RETURN GUIDE.md](./FED%202025%20TAX%20RETURN%20GUIDE.md)
**Purpose**: Federal income tax withholding using automated percentage method

**Key Sections**:
- Worksheet 1A step-by-step calculation
- Annual percentage method tables
- Standard vs Step 2 checkbox withholding rates
- Special adjustments for NRA employees
- Form W-4 version handling

**When to Read**: When implementing federal tax calculations

## Planned Documentation (To Be Created)

### Development & Implementation
1. **DEVELOPMENT_SETUP.md** - Environment setup, dependencies, configuration
2. **IMPLEMENTATION_ROADMAP.md** - Step-by-step development guide
3. **CALCULATION_ENGINE_GUIDE.md** - Implementing the calculation engine
4. **IPC_IMPLEMENTATION_GUIDE.md** - Practical IPC implementation patterns
5. **TESTING_GUIDE.md** - Test strategy, test data, and execution

### Architecture & Design
1. **ARCHITECTURE_DECISIONS.md** - Key architectural choices and rationale
2. **UI_SPECIFICATIONS.md** - User interface design and components
3. **STATE_MANAGEMENT.md** - React state management patterns

### Operations & Deployment
1. **BUILD_AND_DEPLOYMENT.md** - Building and deploying the application
2. **MAINTENANCE_GUIDE.md** - Annual updates, tax tables, backups
3. **TROUBLESHOOTING_GUIDE.md** - Common issues and solutions
4. **RELEASE_PROCESS.md** - Version control and distribution

### Integration & Migration
1. **DATA_MIGRATION_GUIDE.md** - Importing existing payroll data
2. **INTEGRATION_GUIDE.md** - External systems integration
3. **PERFORMANCE_OPTIMIZATION.md** - Performance tuning guide

### End User Documentation
1. **USER_MANUAL.md** - Complete user guide
2. **QUICK_START_GUIDE.md** - Getting started for new users

## Quick Reference Guide

### For Project Managers
1. Start with [PROJECT_ANALYSIS.md](./PROJECT_ANALYSIS.md) - Sections 1, 4, 5, 7
2. Review timeline and resource requirements
3. Understand risks and mitigation strategies

### For Database Developers
1. [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) - Complete document
2. Pay special attention to encryption fields
3. Review indexing strategy

### For Backend Developers
1. [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) - Complete document
2. [API_SPECIFICATION.md](./API_SPECIFICATION.md) - Sections 4-9
3. [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md) - Sections 2, 3, 6

### For Frontend Developers
1. [API_SPECIFICATION.md](./API_SPECIFICATION.md) - Sections 3, 11, 12
2. Review request/response formats
3. Understand error handling

### For Security Engineers
1. [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md) - Complete document
2. [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) - Section on encrypted fields
3. [API_SPECIFICATION.md](./API_SPECIFICATION.md) - Security considerations

### For QA Engineers
1. [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) - Section 12 (Testing)
2. [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md) - Section 7 (Security Testing)
3. [API_SPECIFICATION.md](./API_SPECIFICATION.md) - Section 13 (Testing)

## Implementation Priority

### Phase 1: Foundation (Weeks 1-3)
- Set up project structure
- Implement [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) tables 1-4, 12
- Basic authentication from [API_SPECIFICATION.md](./API_SPECIFICATION.md) Section 4
- Security foundation from [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md) Sections 2-3

### Phase 2: Core Features (Weeks 4-8)
- Employee management (Tables 2-4, API Section 5)
- Basic time tracking (Table 8, API Section 6)
- Simple calculations from [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) Sections 1-2

### Phase 3: Advanced Calculations (Weeks 9-12)
- Tax calculations (Tables 11, Calc Spec Sections 6)
- Complex overtime (Calc Spec Section 2.2-2.3)
- Special pay rules (Calc Spec Sections 3-5)

### Phase 4: Payroll Processing (Weeks 13-15)
- Payroll tables (Tables 9-10)
- Processing API (API Section 7)
- Fringe benefits (Calc Spec Section 7)

### Phase 5: Compliance & Reporting (Weeks 16-18)
- Reporting API (API Section 8)
- Compliance validations (Security Section 5)
- Audit trail completion

### Phase 6: Polish & Security (Weeks 19-20)
- Security hardening
- Performance optimization
- Comprehensive testing

## Key Decisions Required

Before starting development, decide on:

1. **Database Engine**: SQLite vs PostgreSQL (see PROJECT_ANALYSIS.md Section 2.2)
2. **Tax Calculation**: Build vs buy (see PROJECT_ANALYSIS.md Section 6.2)
3. **Deployment Model**: Single-user vs multi-user capability
4. **Update Strategy**: Auto-update mechanism
5. **Backup Strategy**: Local vs cloud backup

## Compliance Checklist

Critical compliance requirements that cannot be compromised:

- [ ] Accurate overtime calculations per union agreements
- [ ] Correct tax withholding per IRS/California requirements
- [ ] Certified payroll report generation
- [ ] Complete audit trail for all changes
- [ ] PII encryption for SSN and bank accounts
- [ ] Data retention per legal requirements
- [ ] Prevailing wage compliance for government projects

## Testing Strategy

### Unit Testing
- All calculation functions (100% coverage required)
- Database operations
- Security functions

### Integration Testing
- IPC communication
- End-to-end workflows
- Tax calculation validation

### Compliance Testing
- Compare against current system
- Validate with IRS/EDD examples
- Union agreement compliance

### Security Testing
- Penetration testing
- Vulnerability scanning
- Access control validation

## Support Resources

### External Documentation
- IRS Publication 15-T (Federal Income Tax Withholding)
- California Employer's Guide (DE 44)
- Union Collective Bargaining Agreements
- Davis-Bacon Wage Determinations

### Development Tools
- Electron documentation
- React documentation
- TypeScript handbook
- SQLite/PostgreSQL documentation

## Maintenance Considerations

### Annual Updates Required
- Federal tax tables (December)
- California tax tables (December)
- Minimum wage updates
- Union agreement changes

### Ongoing Maintenance
- Security patches
- Bug fixes
- Performance optimization
- Feature requests

## Contact Information

For questions about specific documentation:
- Database: Refer to DATABASE_SCHEMA.md
- Calculations: Refer to CALCULATION_SPECIFICATIONS.md
- Security: Refer to SECURITY_COMPLIANCE.md
- API: Refer to API_SPECIFICATION.md

## Version History

- v1.0 - Initial documentation suite
- Last updated: Current date

---

## Cross-References Between Documents

### Tax Implementation Flow
1. Start with [CALCULATION_SPECIFICATIONS.md](./CALCULATION_SPECIFICATIONS.md) Section 6
2. Reference [CA TAX GUIDE.md](./CA%20TAX%20GUIDE.md) for California calculations
3. Reference [FED 2025 TAX RETURN GUIDE.md](./FED%202025%20TAX%20RETURN%20GUIDE.md) for federal calculations
4. Use [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) Section 11 for tax table structure
5. Implement via [API_SPECIFICATION.md](./API_SPECIFICATION.md) Section 7

### Security Implementation Flow
1. Review [SECURITY_COMPLIANCE.md](./SECURITY_COMPLIANCE.md) for requirements
2. Implement encryption per [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) encrypted fields
3. Apply security to [API_SPECIFICATION.md](./API_SPECIFICATION.md) channels
4. Follow audit requirements in both security and database docs

## Glossary of Key Terms

- **Zone**: Geographic wage areas (1-5) determining base pay rates
- **Shift Differential**: Additional pay percentage for 2nd (6%) and 3rd (13%) shifts
- **Show-up Pay**: Minimum compensation when employee reports but gets limited work
- **Meal Penalty**: Overtime rate penalty for not providing meal period after 4.5 hours
- **Fringe Benefits**: Employer-paid benefits (health, pension, vacation, training)
- **Journeyman**: Fully qualified tradesperson at 100% wage rate
- **Apprentice**: Trainee at 50-95% of journeyman rate based on level (1-8)
- **Prevailing Wage**: Minimum wage for government contract work
- **Certified Payroll**: Weekly report required for government projects

**Important**: This documentation represents the minimum requirements for a compliant payroll system. Additional features may be needed based on specific union requirements.