Desktop Payroll Application - Complete Implementation Guide
Project Overview
Build a desktop payroll application for California ironworkers unions with complex wage calculations, zone-based pay, overtime rules, and full tax withholding capabilities.
Initial Setup
1. Create Project Directory Structure
bashmkdir ironworkers-payroll
cd ironworkers-payroll
mkdir -p src/{main,renderer,shared,database}
mkdir -p src/renderer/{components,pages,utils,hooks}
mkdir -p src/main/{services,controllers}
mkdir -p assets/{icons,fonts}
mkdir -p data/{backups,exports}
2. Initialize Node.js Project
bashnpm init -y
3. Install Core Dependencies
bash# Electron and build tools
npm install --save-dev electron electron-builder @electron-forge/cli

# React and TypeScript
npm install react react-dom react-router-dom
npm install --save-dev @types/react @types/react-dom @types/node typescript

# Database
npm install better-sqlite3
npm install --save-dev @types/better-sqlite3

# UI Framework
npm install @mui/material @emotion/react @emotion/styled

# Utilities
npm install date-fns decimal.js pdf-parse
npm install --save-dev @types/pdf-parse

# Development tools
npm install --save-dev webpack webpack-cli webpack-dev-server
npm install --save-dev babel-loader @babel/core @babel/preset-react @babel/preset-typescript
npm install --save-dev css-loader style-loader
4. Configure TypeScript
Create tsconfig.json:
json{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020", "DOM"],
    "jsx": "react",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "allowSyntheticDefaultImports": true,
    "outDir": "./dist"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
Database Schema Implementation
1. Create Database Service
Create src/main/services/database.ts:
typescriptimport Database from 'better-sqlite3';
import path from 'path';
import { app } from 'electron';

class DatabaseService {
  private db: Database.Database;

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'payroll.db');
    this.db = new Database(dbPath);
    this.db.pragma('foreign_keys = ON');
    this.initializeTables();
  }

  private initializeTables() {
    // Run all CREATE TABLE statements from the schema
    this.createEmployeeTables();
    this.createProjectTables();
    this.createWageTables();
    this.createTimeCardTables();
    this.createTaxTables();
  }

  // Implement each table creation method
  // Copy SQL from the planning document
}
2. Implement Core Tables
Create the following tables in order:

Employees table - Basic employee information
Projects table - Job sites with zone information
Wage rates table - Zone-based wage and benefit rates
Timecards table - Daily time entries
Tax tables - Federal and California withholding tables
Payroll calculations table - Computed payroll records

Core Calculation Modules
1. Overtime Calculator
Create src/shared/calculations/overtime.ts:
typescriptexport class OvertimeCalculator {
  // Implement union-specific overtime rules:
  // - Time and a half for first 2 hours over 8 Mon-Fri
  // - Time and a half for first 8 hours on Saturday
  // - Double time for all Sunday/holiday work
  // - Special 4-day/10-hour workweek provisions
}
2. Zone-Based Wage Calculator
Create src/shared/calculations/wages.ts:
typescriptexport class WageCalculator {
  // Implement 5-zone wage system
  // Handle shift differentials (6% 2nd shift, 13% 3rd shift)
  // Calculate fringe benefits per hour
  // Handle apprentice wage scales (50-95% of journeyman)
}
3. Tax Calculator
Create src/shared/calculations/taxes.ts:
typescriptexport class TaxCalculator {
  // Implement federal tax withholding (IRS Publication 15-T)
  // Implement California state tax (Method B)
  // Calculate FICA (Social Security & Medicare)
  // Handle CA SDI and unemployment
}
4. Travel & Subsistence Calculator
Create src/shared/calculations/travel.ts:
typescriptexport class TravelCalculator {
  // Distance-based calculations from union halls
  // Travel reimbursement matrix
  // Federal installation premium ($9/hour)
}
UI Implementation
1. Main Window Setup
Create src/main/index.ts:
typescriptimport { app, BrowserWindow } from 'electron';

let mainWindow: BrowserWindow | null;

app.whenReady().then(() => {
  mainWindow = new BrowserWindow({
    width: 1400,
    height: 900,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js')
    }
  });
  
  mainWindow.loadFile('index.html');
});
2. React App Structure
Create src/renderer/App.tsx:
typescriptimport React from 'react';
import { Routes, Route } from 'react-router-dom';

export function App() {
  return (
    <Routes>
      <Route path="/" element={<Dashboard />} />
      <Route path="/employees" element={<EmployeeManagement />} />
      <Route path="/timeentry" element={<TimeEntry />} />
      <Route path="/payroll" element={<PayrollProcessing />} />
      <Route path="/reports" element={<Reports />} />
    </Routes>
  );
}
3. Key Components to Build

Dashboard - Overview of current pay period
Employee Management - Add/edit employees, track apprentice progression
Time Entry - Daily timecard entry with automatic calculations
Payroll Processing - Batch processing with preview
Reports - Certified payroll, union remittances, tax reports

Special Features Implementation
1. Show-up Pay Rules

$60 base (higher for overtime days)
4-hour minimum if put to work
6-hour minimum if work 4+ hours

2. Meal Period Penalties

Track breaks
Apply overtime rate after 4.5 hours without lunch

3. Import Tax Tables
Create PDF parser to import IRS and California tax tables:
typescriptimport pdfParse from 'pdf-parse';

export async function importTaxTables(pdfPath: string) {
  // Parse PDF and extract tax bracket information
  // Store in database for calculations
}
Data Management
1. Backup System
Implement automatic backups:
typescriptexport class BackupService {
  // Daily automatic backups
  // Export to Google Drive/Dropbox
  // Maintain 30-day backup history
}
2. Data Export
Create export functionality:
typescriptexport class ExportService {
  // Export payroll data to CSV
  // Generate PDF pay stubs
  // Create ACH files for direct deposit
}
Testing Strategy
1. Create Test Data

Generate employees in each classification
Create projects in each zone
Test all overtime scenarios
Verify tax calculations against examples

2. Validation Tests

Compare calculations to union agreement
Verify tax withholdings match IRS/CA examples
Test edge cases (zone boundaries, overtime limits)

Development Phases
Phase 1 (Weeks 1-2): Core Framework

Set up Electron + React + SQLite
Implement basic database schema
Create employee and project management

Phase 2 (Weeks 3-4): Time Tracking

Build timecard entry interface
Implement basic overtime calculations
Add shift differential logic

Phase 3 (Weeks 5-6): Payroll Calculations

Complete wage calculations with zones
Add all benefit contributions
Implement subsistence/travel calculations

Phase 4 (Weeks 7-8): Tax Integration

Import tax tables from PDFs
Implement federal tax calculations
Add California state tax calculations

Phase 5 (Weeks 9-10): Reporting & Polish

Build required reports
Add export capabilities
Implement backup automation
Comprehensive testing

Key Configuration Files
1. Package.json Scripts
json{
  "scripts": {
    "start": "electron .",
    "dev": "webpack serve --config webpack.dev.js",
    "build": "webpack --config webpack.prod.js && electron-builder",
    "test": "jest"
  }
}
2. Electron Builder Config
json{
  "build": {
    "appId": "com.ironworkers.payroll",
    "productName": "Ironworkers Payroll",
    "directories": {
      "output": "dist"
    },
    "win": {
      "target": "nsis"
    },
    "mac": {
      "target": "dmg"
    }
  }
}
Critical Implementation Notes

Zone Detection: Projects automatically determine wage zone based on location
Overtime Tracking: Real-time overtime alerts during time entry
Compliance: Maintain audit trail for all calculations
Updates: Annual tax table update process
Security: Encrypt sensitive data, implement user authentication

Next Steps

Start with Phase 1 - create the basic project structure
Implement the database schema exactly as specified
Build a simple UI to test employee and project creation
Add time entry functionality with basic calculations
Progressively add complexity through each phase