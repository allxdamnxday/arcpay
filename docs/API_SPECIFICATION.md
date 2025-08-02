# API Specification - IPC Communication

## Overview

This document specifies the Inter-Process Communication (IPC) API between the Electron main process and renderer process. All communication follows a request-response pattern with proper error handling and validation.

## 1. Architecture Overview

### 1.1 Communication Flow
```
Renderer (React UI) 
    ↓ IPC Invoke/Send
Main Process (Business Logic)
    ↓ Service Layer
Database / External Services
```

### 1.2 Security Model
- Context isolation enabled
- No direct node integration in renderer
- All IPC channels are explicitly defined
- Input validation on both sides
- Sanitization of all data

## 2. IPC Channel Naming Convention

```typescript
enum IPCChannel {
    // Authentication
    AUTH_LOGIN = 'auth:login',
    AUTH_LOGOUT = 'auth:logout',
    AUTH_VERIFY_SESSION = 'auth:verify-session',
    AUTH_CHANGE_PASSWORD = 'auth:change-password',
    
    // Employee Management
    EMPLOYEE_LIST = 'employee:list',
    EMPLOYEE_GET = 'employee:get',
    EMPLOYEE_CREATE = 'employee:create',
    EMPLOYEE_UPDATE = 'employee:update',
    EMPLOYEE_DEACTIVATE = 'employee:deactivate',
    
    // Time Card Management
    TIMECARD_LIST = 'timecard:list',
    TIMECARD_GET = 'timecard:get',
    TIMECARD_CREATE = 'timecard:create',
    TIMECARD_UPDATE = 'timecard:update',
    TIMECARD_APPROVE = 'timecard:approve',
    TIMECARD_CALCULATE = 'timecard:calculate',
    
    // Payroll Processing
    PAYROLL_CALCULATE = 'payroll:calculate',
    PAYROLL_PREVIEW = 'payroll:preview',
    PAYROLL_PROCESS = 'payroll:process',
    PAYROLL_APPROVE = 'payroll:approve',
    PAYROLL_VOID = 'payroll:void',
    
    // Reports
    REPORT_GENERATE = 'report:generate',
    REPORT_LIST = 'report:list',
    REPORT_EXPORT = 'report:export',
    
    // System
    SYSTEM_BACKUP = 'system:backup',
    SYSTEM_RESTORE = 'system:restore',
    SYSTEM_SETTINGS = 'system:settings',
    SYSTEM_UPDATE_CHECK = 'system:update-check'
}
```

## 3. Common Types

### 3.1 Request/Response Wrapper
```typescript
interface IPCRequest<T = any> {
    id: string;          // Unique request ID
    timestamp: number;   // Request timestamp
    userId: string;      // Authenticated user ID
    sessionId: string;   // Session token
    data: T;            // Request payload
}

interface IPCResponse<T = any> {
    id: string;          // Matching request ID
    timestamp: number;   // Response timestamp
    success: boolean;    // Operation success
    data?: T;           // Response payload (if success)
    error?: IPCError;   // Error details (if failure)
}

interface IPCError {
    code: string;        // Error code
    message: string;     // Human-readable message
    details?: any;       // Additional error context
}
```

### 3.2 Pagination
```typescript
interface PaginationRequest {
    page: number;        // 1-based page number
    pageSize: number;    // Items per page
    sortBy?: string;     // Sort field
    sortOrder?: 'asc' | 'desc';
}

interface PaginatedResponse<T> {
    items: T[];
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
}
```

### 3.3 Common Enums
```typescript
enum SortOrder {
    ASC = 'asc',
    DESC = 'desc'
}

enum DateRange {
    TODAY = 'today',
    THIS_WEEK = 'this_week',
    THIS_MONTH = 'this_month',
    THIS_QUARTER = 'this_quarter',
    THIS_YEAR = 'this_year',
    CUSTOM = 'custom'
}
```

## 4. Authentication API

### 4.1 Login
```typescript
// Channel: auth:login
interface LoginRequest {
    username: string;
    password: string;
    mfaCode?: string;    // If MFA enabled
}

interface LoginResponse {
    user: {
        id: string;
        username: string;
        firstName: string;
        lastName: string;
        role: UserRole;
        unionLocalId?: string;
        permissions: string[];
    };
    session: {
        id: string;
        token: string;
        expiresAt: number;
    };
    requiresMFA?: boolean;
    requiresPasswordChange?: boolean;
}

// Main process handler
ipcMain.handle(IPCChannel.AUTH_LOGIN, async (event, request: IPCRequest<LoginRequest>) => {
    try {
        // Validate input
        validateLoginRequest(request.data);
        
        // Attempt authentication
        const result = await authService.login(
            request.data.username,
            request.data.password,
            request.data.mfaCode
        );
        
        // Log successful login
        await auditService.log({
            event: AuditEvent.LOGIN_SUCCESS,
            userId: result.user.id,
            ipAddress: getClientIP(event)
        });
        
        return {
            id: request.id,
            timestamp: Date.now(),
            success: true,
            data: result
        };
    } catch (error) {
        // Log failed login
        await auditService.log({
            event: AuditEvent.LOGIN_FAILURE,
            userId: request.data.username,
            ipAddress: getClientIP(event),
            error: error.message
        });
        
        return {
            id: request.id,
            timestamp: Date.now(),
            success: false,
            error: {
                code: 'AUTH_FAILED',
                message: 'Invalid credentials'
            }
        };
    }
});
```

### 4.2 Logout
```typescript
// Channel: auth:logout
interface LogoutRequest {
    // No additional data needed
}

interface LogoutResponse {
    success: boolean;
}
```

### 4.3 Verify Session
```typescript
// Channel: auth:verify-session
interface VerifySessionRequest {
    // Session ID included in wrapper
}

interface VerifySessionResponse {
    valid: boolean;
    expiresAt?: number;
    user?: {
        id: string;
        role: UserRole;
        permissions: string[];
    };
}
```

## 5. Employee Management API

### 5.1 List Employees
```typescript
// Channel: employee:list
interface ListEmployeesRequest extends PaginationRequest {
    filters?: {
        active?: boolean;
        unionLocalId?: string;
        classificationId?: string;
        search?: string;     // Search by name or employee number
    };
}

interface ListEmployeesResponse extends PaginatedResponse<EmployeeSummary> {
    // Inherits items, total, page, etc.
}

interface EmployeeSummary {
    id: string;
    employeeNumber: string;
    firstName: string;
    lastName: string;
    classification: string;
    status: 'active' | 'terminated';
    hireDate: string;
}
```

### 5.2 Get Employee Details
```typescript
// Channel: employee:get
interface GetEmployeeRequest {
    employeeId: string;
    includeSensitive?: boolean; // Requires special permission
}

interface GetEmployeeResponse {
    employee: EmployeeDetail;
}

interface EmployeeDetail {
    id: string;
    employeeNumber: string;
    firstName: string;
    lastName: string;
    middleName?: string;
    dateOfBirth: string;
    ssn?: string;           // Only if includeSensitive=true
    address: {
        line1: string;
        line2?: string;
        city: string;
        state: string;
        zipCode: string;
    };
    contact: {
        phone?: string;
        email?: string;
    };
    employment: {
        hireDate: string;
        terminationDate?: string;
        classification: {
            id: string;
            name: string;
            isApprentice: boolean;
        };
        apprenticeLevel?: number;
        journeymanDate?: string;
    };
    tax: {
        federalFilingStatus: string;
        federalAllowances: number;
        stateFilingStatus: string;
        stateAllowances: number;
        additionalFederalWithholding: number;
        additionalStateWithholding: number;
    };
    payment: {
        method: 'check' | 'direct_deposit';
        bankAccounts?: BankAccount[]; // Only if direct deposit
    };
}
```

### 5.3 Create Employee
```typescript
// Channel: employee:create
interface CreateEmployeeRequest {
    employee: {
        employeeNumber: string;
        unionLocalId: string;
        firstName: string;
        lastName: string;
        middleName?: string;
        dateOfBirth: string;
        ssn: string;
        address: {
            line1: string;
            line2?: string;
            city: string;
            state: string;
            zipCode: string;
        };
        contact: {
            phone?: string;
            email?: string;
        };
        hireDate: string;
        classificationId: string;
        apprenticeLevel?: number;
        tax: {
            federalFilingStatus: string;
            federalAllowances: number;
            stateFilingStatus: string;
            stateAllowances: number;
        };
        paymentMethod: 'check' | 'direct_deposit';
    };
}

interface CreateEmployeeResponse {
    employeeId: string;
    employeeNumber: string;
}
```

### 5.4 Update Employee
```typescript
// Channel: employee:update
interface UpdateEmployeeRequest {
    employeeId: string;
    updates: Partial<CreateEmployeeRequest['employee']>;
}

interface UpdateEmployeeResponse {
    success: boolean;
    employee: EmployeeDetail;
}
```

## 6. Time Card Management API

### 6.1 List Time Cards
```typescript
// Channel: timecard:list
interface ListTimeCardsRequest extends PaginationRequest {
    filters: {
        employeeId?: string;
        projectId?: string;
        dateRange: {
            start: string;  // ISO date
            end: string;    // ISO date
        };
        status?: TimeCardStatus[];
    };
}

interface ListTimeCardsResponse extends PaginatedResponse<TimeCardSummary> {
    totals: {
        regularHours: number;
        overtimeHours: number;
        doubleTimeHours: number;
    };
}

interface TimeCardSummary {
    id: string;
    employeeName: string;
    projectName: string;
    workDate: string;
    regularHours: number;
    overtimeHours: number;
    doubleTimeHours: number;
    status: TimeCardStatus;
}
```

### 6.2 Create Time Card
```typescript
// Channel: timecard:create
interface CreateTimeCardRequest {
    timeCard: {
        employeeId: string;
        projectId: string;
        workDate: string;
        startTime?: string;     // Optional, for time tracking
        endTime?: string;       // Optional, for time tracking
        totalHours: number;     // If not using start/end
        shiftNumber: 1 | 2 | 3;
        mealPeriodTaken: boolean;
        notes?: string;
    };
}

interface CreateTimeCardResponse {
    timeCardId: string;
    calculations: {
        regularHours: number;
        overtimeHours: number;
        doubleTimeHours: number;
        showUpPay: number;
        mealPenalty: number;
    };
}
```

### 6.3 Approve Time Cards
```typescript
// Channel: timecard:approve
interface ApproveTimeCardsRequest {
    timeCardIds: string[];
    approverNotes?: string;
}

interface ApproveTimeCardsResponse {
    approved: string[];
    failed: Array<{
        timeCardId: string;
        reason: string;
    }>;
}
```

### 6.4 Calculate Time Card Preview
```typescript
// Channel: timecard:calculate
interface CalculateTimeCardRequest {
    employeeId: string;
    projectId: string;
    workDate: string;
    hours: number;
    shiftNumber: 1 | 2 | 3;
    mealPeriodTaken: boolean;
}

interface CalculateTimeCardResponse {
    breakdown: {
        regularHours: number;
        overtimeHours: number;
        doubleTimeHours: number;
    };
    wages: {
        hourlyRate: number;
        regularPay: number;
        overtimePay: number;
        doubleTimePay: number;
        showUpPay: number;
        mealPenalty: number;
        totalGross: number;
    };
    messages: string[];  // Warnings or info
}
```

## 7. Payroll Processing API

### 7.1 Calculate Payroll
```typescript
// Channel: payroll:calculate
interface CalculatePayrollRequest {
    payrollPeriodId: string;
    employeeIds?: string[];  // If not specified, all employees
    options: {
        includeTerminated: boolean;
        includeDraft: boolean;
    };
}

interface CalculatePayrollResponse {
    calculations: PayrollCalculation[];
    summary: {
        totalEmployees: number;
        totalGrossWages: number;
        totalNetPay: number;
        totalTaxes: number;
        totalDeductions: number;
        totalFringeBenefits: number;
    };
    errors: Array<{
        employeeId: string;
        employeeName: string;
        error: string;
    }>;
}

interface PayrollCalculation {
    employeeId: string;
    employeeName: string;
    employeeNumber: string;
    hours: {
        regular: number;
        overtime: number;
        doubleTime: number;
    };
    wages: {
        regular: number;
        overtime: number;
        doubleTime: number;
        other: number;      // Show-up, meal, travel, etc.
        gross: number;
    };
    taxes: {
        federal: number;
        state: number;
        socialSecurity: number;
        medicare: number;
        sdi: number;
    };
    deductions: {
        unionDues: number;
        other: number;
    };
    netPay: number;
    fringeBenefits: {
        health: number;
        pension: number;
        vacation: number;
        training: number;
    };
}
```

### 7.2 Preview Payroll
```typescript
// Channel: payroll:preview
interface PreviewPayrollRequest {
    payrollPeriodId: string;
    format: 'summary' | 'detailed';
}

interface PreviewPayrollResponse {
    preview: {
        period: {
            start: string;
            end: string;
            payDate: string;
        };
        employees: PayrollCalculation[];
        totals: PayrollTotals;
    };
    warnings: string[];
}
```

### 7.3 Process Payroll
```typescript
// Channel: payroll:process
interface ProcessPayrollRequest {
    payrollPeriodId: string;
    calculations: string[];  // Calculation IDs to process
    options: {
        generatePayStubs: boolean;
        scheduleDirectDeposit: boolean;
        printChecks: boolean;
    };
}

interface ProcessPayrollResponse {
    processed: number;
    failed: number;
    directDeposits: {
        scheduled: number;
        achFile: string;    // Path to ACH file
    };
    checks: {
        count: number;
        startNumber: string;
        endNumber: string;
    };
    payStubs: {
        generated: number;
        path: string;       // Directory path
    };
}
```

## 8. Reporting API

### 8.1 Generate Report
```typescript
// Channel: report:generate
interface GenerateReportRequest {
    reportType: ReportType;
    parameters: {
        dateRange: {
            start: string;
            end: string;
        };
        employeeIds?: string[];
        projectIds?: string[];
        format: 'pdf' | 'csv' | 'excel';
    };
}

enum ReportType {
    CERTIFIED_PAYROLL = 'certified_payroll',
    UNION_REMITTANCE = 'union_remittance',
    QUARTERLY_941 = 'quarterly_941',
    QUARTERLY_DE9 = 'quarterly_de9',
    ANNUAL_W2 = 'annual_w2',
    EMPLOYEE_EARNINGS = 'employee_earnings',
    PROJECT_LABOR = 'project_labor',
    FRINGE_BENEFITS = 'fringe_benefits'
}

interface GenerateReportResponse {
    reportId: string;
    fileName: string;
    filePath: string;
    format: string;
    size: number;
    generatedAt: string;
}
```

### 8.2 List Reports
```typescript
// Channel: report:list
interface ListReportsRequest extends PaginationRequest {
    filters?: {
        reportType?: ReportType[];
        generatedBy?: string;
        dateRange?: {
            start: string;
            end: string;
        };
    };
}

interface ListReportsResponse extends PaginatedResponse<ReportSummary> {
    // Inherits standard pagination
}

interface ReportSummary {
    id: string;
    type: ReportType;
    name: string;
    generatedAt: string;
    generatedBy: string;
    size: number;
    parameters: any;
}
```

## 9. System API

### 9.1 Backup Database
```typescript
// Channel: system:backup
interface BackupRequest {
    type: 'manual' | 'scheduled';
    destination: 'local' | 'cloud';
    encrypt: boolean;
}

interface BackupResponse {
    backupId: string;
    fileName: string;
    size: number;
    encrypted: boolean;
    location: string;
    createdAt: string;
}
```

### 9.2 System Settings
```typescript
// Channel: system:settings
interface GetSettingsRequest {
    category?: string;
}

interface GetSettingsResponse {
    settings: {
        general: {
            companyName: string;
            defaultUnionLocal: string;
            payrollFrequency: string;
            overtimeRules: string;
        };
        security: {
            sessionTimeout: number;
            passwordExpiration: number;
            mfaRequired: boolean;
            auditRetention: number;
        };
        backup: {
            autoBackupEnabled: boolean;
            backupFrequency: string;
            backupRetention: number;
            backupLocation: string;
        };
        integration: {
            taxTableUpdateUrl: string;
            achExportFormat: string;
        };
    };
}

interface UpdateSettingsRequest {
    category: string;
    settings: any;
}

interface UpdateSettingsResponse {
    success: boolean;
    settings: any;
}
```

## 10. Error Handling

### 10.1 Standard Error Codes
```typescript
enum ErrorCode {
    // Authentication
    AUTH_INVALID_CREDENTIALS = 'AUTH_001',
    AUTH_SESSION_EXPIRED = 'AUTH_002',
    AUTH_INSUFFICIENT_PERMISSIONS = 'AUTH_003',
    AUTH_ACCOUNT_LOCKED = 'AUTH_004',
    
    // Validation
    VALIDATION_REQUIRED_FIELD = 'VAL_001',
    VALIDATION_INVALID_FORMAT = 'VAL_002',
    VALIDATION_OUT_OF_RANGE = 'VAL_003',
    VALIDATION_DUPLICATE = 'VAL_004',
    
    // Business Logic
    BUSINESS_INVALID_STATE = 'BUS_001',
    BUSINESS_CALCULATION_ERROR = 'BUS_002',
    BUSINESS_RULE_VIOLATION = 'BUS_003',
    
    // System
    SYSTEM_DATABASE_ERROR = 'SYS_001',
    SYSTEM_FILE_ERROR = 'SYS_002',
    SYSTEM_NETWORK_ERROR = 'SYS_003',
    SYSTEM_UNKNOWN_ERROR = 'SYS_999'
}
```

### 10.2 Error Response Format
```typescript
interface ErrorResponse {
    code: ErrorCode;
    message: string;
    details?: {
        field?: string;
        value?: any;
        constraint?: string;
        suggestion?: string;
    };
    stack?: string;  // Only in development mode
}
```

## 11. Preload Script Bridge

### 11.1 Exposed API
```typescript
// preload.js
contextBridge.exposeInMainWorld('payrollAPI', {
    // Authentication
    login: (username: string, password: string, mfaCode?: string) => 
        ipcRenderer.invoke(IPCChannel.AUTH_LOGIN, createRequest({ username, password, mfaCode })),
    
    logout: () => 
        ipcRenderer.invoke(IPCChannel.AUTH_LOGOUT, createRequest({})),
    
    // Employee Management
    listEmployees: (request: ListEmployeesRequest) => 
        ipcRenderer.invoke(IPCChannel.EMPLOYEE_LIST, createRequest(request)),
    
    getEmployee: (employeeId: string, includeSensitive = false) => 
        ipcRenderer.invoke(IPCChannel.EMPLOYEE_GET, createRequest({ employeeId, includeSensitive })),
    
    createEmployee: (employee: CreateEmployeeRequest['employee']) => 
        ipcRenderer.invoke(IPCChannel.EMPLOYEE_CREATE, createRequest({ employee })),
    
    updateEmployee: (employeeId: string, updates: any) => 
        ipcRenderer.invoke(IPCChannel.EMPLOYEE_UPDATE, createRequest({ employeeId, updates })),
    
    // ... continue for all API methods
});
```

### 11.2 Renderer Usage
```typescript
// In React components
const PayrollService = {
    async login(username: string, password: string): Promise<LoginResponse> {
        const response = await window.payrollAPI.login(username, password);
        if (!response.success) {
            throw new Error(response.error.message);
        }
        return response.data;
    },
    
    async getEmployees(filters?: any): Promise<EmployeeSummary[]> {
        const response = await window.payrollAPI.listEmployees({
            page: 1,
            pageSize: 100,
            filters
        });
        if (!response.success) {
            throw new Error(response.error.message);
        }
        return response.data.items;
    }
    
    // ... continue for all methods
};
```

## 12. Performance Considerations

### 12.1 Request Batching
```typescript
// For bulk operations
interface BatchRequest<T> {
    operations: Array<{
        id: string;
        operation: T;
    }>;
}

interface BatchResponse<T> {
    results: Array<{
        id: string;
        success: boolean;
        data?: T;
        error?: IPCError;
    }>;
}
```

### 12.2 Streaming Large Results
```typescript
// For reports or large data exports
interface StreamRequest {
    streamId: string;
    chunkSize: number;
}

interface StreamChunk<T> {
    streamId: string;
    chunkIndex: number;
    totalChunks: number;
    data: T[];
    isLast: boolean;
}
```

### 12.3 Caching Strategy
```typescript
interface CachePolicy {
    channel: IPCChannel;
    ttl: number;        // Time to live in seconds
    key: (request: any) => string;
}

const cachePolicies: CachePolicy[] = [
    {
        channel: IPCChannel.EMPLOYEE_GET,
        ttl: 300,  // 5 minutes
        key: (req) => `employee:${req.employeeId}`
    },
    {
        channel: IPCChannel.SYSTEM_SETTINGS,
        ttl: 3600, // 1 hour
        key: (req) => `settings:${req.category || 'all'}`
    }
];
```

## 13. Testing

### 13.1 Mock IPC for Testing
```typescript
class MockIPCRenderer {
    private handlers = new Map<string, Function>();
    
    invoke(channel: string, ...args: any[]): Promise<any> {
        const handler = this.handlers.get(channel);
        if (!handler) {
            return Promise.reject(new Error(`No handler for ${channel}`));
        }
        return handler(...args);
    }
    
    mockResponse(channel: string, response: any): void {
        this.handlers.set(channel, () => Promise.resolve(response));
    }
}
```

### 13.2 Integration Test Example
```typescript
describe('Employee API', () => {
    let mockIPC: MockIPCRenderer;
    
    beforeEach(() => {
        mockIPC = new MockIPCRenderer();
        global.window.payrollAPI = createAPIFromIPC(mockIPC);
    });
    
    test('should list employees', async () => {
        const mockResponse = {
            success: true,
            data: {
                items: [{ id: '1', firstName: 'John', lastName: 'Doe' }],
                total: 1
            }
        };
        
        mockIPC.mockResponse(IPCChannel.EMPLOYEE_LIST, mockResponse);
        
        const result = await PayrollService.getEmployees();
        expect(result).toHaveLength(1);
        expect(result[0].firstName).toBe('John');
    });
});
```