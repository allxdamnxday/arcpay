# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
Desktop payroll application for California ironworkers unions featuring complex wage calculations, zone-based pay rates, overtime rules, and comprehensive tax withholding. Built with Electron, React, TypeScript, and SQLite.

## Documentation Navigation

- **[DEVELOPMENT_SETUP.md](./DEVELOPMENT_SETUP.md)**: Environment setup, prerequisites, and initial configuration
- **[BUILD_AND_DEPLOYMENT.md](./BUILD_AND_DEPLOYMENT.md)**: Production build process and deployment instructions
- **[docs/](./docs/)**: Detailed technical documentation
  - API specifications, calculation guides, database schemas
  - Testing and security compliance documentation

## Development Commands

### Build & Run
```bash
npm start          # Start Electron app
npm run dev        # Run webpack dev server
npm run build      # Build production app with electron-builder
npm test           # Run Jest tests
```

### Installation
```bash
npm install        # Install all dependencies
```

## Architecture Overview

### Technology Stack
- **Frontend**: React + TypeScript + Material UI
- **Backend**: Electron main process with SQLite database
- **Database**: better-sqlite3 for local data storage
- **Build Tools**: Webpack, Babel, electron-builder

### Project Structure
```
src/
├── main/               # Electron main process
│   ├── services/       # Database and business logic services
│   └── controllers/    # IPC controllers
├── renderer/           # React frontend
│   ├── components/     # Reusable UI components
│   ├── pages/          # Route-based page components
│   ├── utils/          # Frontend utilities
│   └── hooks/          # Custom React hooks
├── shared/             # Shared business logic
│   └── calculations/   # Wage, tax, overtime calculators
└── database/           # Database schemas and migrations
```

### Core Modules

#### Wage Calculation System
- **5-zone wage system**: Different hourly rates based on geographic zones
- **Shift differentials**: 6% for 2nd shift, 13% for 3rd shift
- **Apprentice scales**: 50-95% of journeyman rate
- **Fringe benefits**: Calculated per hour worked

#### Overtime Rules
- **Monday-Friday**: Time and a half for first 2 hours over 8
- **Saturday**: Time and a half for first 8 hours
- **Sunday/Holidays**: All hours at double time
- **4-10 workweek**: Special provisions for 4-day/10-hour schedules

#### Tax Calculations
- **Federal withholding**: IRS Publication 15-T implementation
- **California state tax**: Method B calculation
- **FICA**: Social Security and Medicare
- **CA SDI/UI**: State disability and unemployment insurance

#### Special Rules
- **Show-up pay**: $60 base, 4-hour minimum if put to work
- **Meal penalties**: Overtime rate after 4.5 hours without lunch
- **Travel/subsistence**: Distance-based calculations from union halls
- **Federal installation premium**: $9/hour additional

### Database Schema
Key tables include:
- `employees`: Employee information and classifications
- `projects`: Job sites with zone information
- `wage_rates`: Zone-based wage and benefit rates
- `timecards`: Daily time entries
- `tax_tables`: Federal and California withholding tables
- `payroll_calculations`: Computed payroll records

### Key Features
- **Zone detection**: Automatic wage zone determination based on project location
- **Real-time calculations**: Overtime alerts during time entry
- **Audit trail**: Complete calculation history for compliance
- **Tax table imports**: PDF parser for IRS/California tax tables
- **Backup automation**: Daily backups with 30-day retention
- **Export capabilities**: CSV, PDF pay stubs, ACH files

## Coding Guidelines

### Modular Architecture
- **File size limit**: Keep all source files under 500 lines
- **Single responsibility**: Each module should have one clear purpose
- **Component breakdown**: Split large components into smaller, focused units
- **Service separation**: Separate business logic from UI components

### File Organization Examples
- Split large services: `PayrollService.ts` → `WageCalculator.ts`, `TaxCalculator.ts`, `OvertimeCalculator.ts`
- Break down complex components: `PayrollForm.tsx` → `EmployeeSelector.tsx`, `TimeEntry.tsx`, `WageDisplay.tsx`
- Extract shared logic: Move reusable calculations to `src/shared/calculations/`

### KISS Principle (Keep It Simple, Stupid)
- **Avoid premature optimization**: Write clear code first, optimize only when necessary
- **Minimize dependencies**: Use built-in solutions before adding external libraries
- **Clear naming**: Use descriptive names that explain intent without comments
- **Simple algorithms**: Choose straightforward solutions over clever ones
- **Limit abstractions**: Only abstract when you have 3+ similar use cases
- **Direct communication**: Prefer direct function calls over complex event systems

### Code Style
- Use TypeScript strict mode
- Prefer functional components with hooks in React
- Use async/await over promises
- Implement proper error handling with try/catch
- Add JSDoc comments for public APIs

## Testing Approach
- Test all overtime scenarios against union agreements
- Verify tax calculations match IRS/CA examples
- Test edge cases for zone boundaries and overtime limits
- Generate test data for each employee classification

## Security Considerations
- Database encryption for sensitive data
- User authentication required
- Secure storage of tax information
- No hardcoded credentials or API keys