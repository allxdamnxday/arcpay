# Architecture Decision Records (ADR)

## Overview

This document captures the key architectural decisions made for the ArcPay Ironworkers Payroll System. Each decision is documented with context, options considered, decision made, and consequences.

## ADR-001: Desktop Application Platform

**Date**: December 2024  
**Status**: Accepted

### Context
The application needs to run on Windows, macOS, and Linux desktops with access to local file systems and databases. Users may not always have internet connectivity, especially at job sites.

### Options Considered
1. **Native Applications** (separate for each OS)
   - Pros: Best performance, native look and feel
   - Cons: Triple development effort, maintenance nightmare

2. **Electron + Web Technologies**
   - Pros: Single codebase, familiar technologies, good ecosystem
   - Cons: Larger bundle size, memory usage

3. **Tauri (Rust + Web Frontend)**
   - Pros: Smaller bundle, better performance
   - Cons: Less mature, smaller ecosystem, Rust learning curve

4. **.NET MAUI**
   - Pros: Good Windows integration, single codebase
   - Cons: Limited Linux support, C# requirement

### Decision
**Electron + React + TypeScript**

### Rationale
- Single codebase for all platforms
- Large ecosystem and community support
- Team familiarity with web technologies
- Proven track record for desktop applications
- Good balance of development speed and performance

### Consequences
- ✅ Faster development with familiar tools
- ✅ Easy to find developers
- ✅ Rich UI capabilities
- ❌ Larger application size (~150MB)
- ❌ Higher memory usage (~200MB baseline)
- ❌ Need to optimize for performance

---

## ADR-002: Database Selection

**Date**: December 2024  
**Status**: Accepted

### Context
Need local data storage for payroll data including sensitive information. Must work offline, support complex queries, and handle concurrent access from the application.

### Options Considered
1. **SQLite**
   - Pros: Embedded, no server, single file, proven reliability
   - Cons: Limited concurrent writes, size limitations

2. **PostgreSQL** (local installation)
   - Pros: Full SQL support, better concurrency, advanced features
   - Cons: Requires installation, more complex deployment

3. **LevelDB/RocksDB**
   - Pros: Fast key-value storage, embedded
   - Cons: No SQL, limited query capabilities

4. **SQL Server Express**
   - Pros: Robust, good Windows integration
   - Cons: Windows-centric, installation required

### Decision
**SQLite with better-sqlite3**

### Rationale
- Zero configuration for users
- Single file backup/restore
- Sufficient for single-user desktop application
- Excellent Node.js integration with better-sqlite3
- Can migrate to PostgreSQL later if needed

### Consequences
- ✅ Simple deployment and backup
- ✅ No external dependencies
- ✅ Fast for read operations
- ✅ ACID compliant
- ❌ Limited to single writer at a time
- ❌ Database size limit (~281TB theoretical, 50GB practical)
- ❌ No built-in replication

### Migration Path
If concurrent access becomes necessary:
1. Abstract database layer
2. Support both SQLite and PostgreSQL
3. Let users choose based on needs

---

## ADR-003: State Management

**Date**: December 2024  
**Status**: Accepted

### Context
React application needs predictable state management for complex payroll calculations, employee data, and UI state.

### Options Considered
1. **Redux + Redux Toolkit**
   - Pros: Predictable, time-travel debugging, large ecosystem
   - Cons: Boilerplate, learning curve

2. **Zustand**
   - Pros: Simple API, less boilerplate, TypeScript friendly
   - Cons: Smaller ecosystem, less tooling

3. **MobX**
   - Pros: Less boilerplate, reactive
   - Cons: Magic, harder to debug

4. **Context API + useReducer**
   - Pros: Built-in, no dependencies
   - Cons: Performance issues at scale, verbose

### Decision
**Zustand for global state + React Query for server state**

### Rationale
- Minimal boilerplate (KISS principle)
- Excellent TypeScript support
- Easy to understand and debug
- React Query handles caching and synchronization
- Can be contained in small modules (<500 lines)

### Consequences
- ✅ Fast development
- ✅ Easy onboarding
- ✅ Small bundle size
- ✅ Good performance
- ❌ Less ecosystem than Redux
- ❌ Fewer debugging tools

---

## ADR-004: IPC Communication

**Date**: December 2024  
**Status**: Accepted

### Context
Electron requires secure communication between main process (Node.js) and renderer process (React).

### Options Considered
1. **Traditional IPC with channels**
   - Pros: Explicit, secure
   - Cons: Verbose, manual type safety

2. **Electron-typed-ipc**
   - Pros: Type-safe, less boilerplate
   - Cons: Additional dependency

3. **tRPC adapter**
   - Pros: Full type safety, RPC-like
   - Cons: Complexity, overhead

### Decision
**Traditional IPC with TypeScript wrapper**

### Rationale
- Full control over security
- No magic or hidden behavior
- Can enforce validation
- Type safety through code generation

### Implementation
```typescript
// Typed wrapper around IPC
interface IPCChannel<TRequest, TResponse> {
  name: string;
  request: TRequest;
  response: TResponse;
}

// Auto-generated from API specification
```

### Consequences
- ✅ Explicit and secure
- ✅ Full type safety
- ✅ Easy to audit
- ❌ More initial setup
- ❌ Need to maintain type definitions

---

## ADR-005: Security Architecture

**Date**: December 2024  
**Status**: Accepted

### Context
Payroll application handles sensitive data including SSNs, bank accounts, and salary information. Must comply with various regulations.

### Options Considered
1. **Application-level encryption**
   - Pros: Full control, field-level encryption
   - Cons: Complex key management

2. **Full disk encryption** (rely on OS)
   - Pros: Simple, transparent
   - Cons: All-or-nothing, no field control

3. **Database encryption** (SQLCipher)
   - Pros: Transparent to app, proven
   - Cons: Entire database encrypted

### Decision
**Hybrid approach: SQLCipher + field-level encryption**

### Rationale
- Database encryption for data at rest
- Field-level encryption for highly sensitive data (SSN, bank accounts)
- Allows partial decryption for performance
- Defense in depth

### Implementation
- SQLCipher for database encryption
- AES-256-GCM for field encryption
- Key derivation with PBKDF2
- OS keychain for master key storage

### Consequences
- ✅ Strong security
- ✅ Granular control
- ✅ Compliance ready
- ❌ Complex key management
- ❌ Performance overhead
- ❌ Careful backup procedures needed

---

## ADR-006: Modular Architecture

**Date**: December 2024  
**Status**: Accepted

### Context
Need to maintain code quality and enforce KISS principle. Team wants to limit file sizes to 500 lines.

### Options Considered
1. **Monolithic modules**
   - Pros: Fewer files, related code together
   - Cons: Large files, hard to test

2. **Micro-modules**
   - Pros: Small files, single responsibility
   - Cons: Many files, navigation overhead

3. **Feature-based modules**
   - Pros: Logical grouping, reasonable size
   - Cons: Some cross-cutting concerns

### Decision
**Feature-based modules with 500-line limit**

### Structure
```
src/
├── main/
│   ├── employees/
│   │   ├── EmployeeService.ts (<500 lines)
│   │   ├── EmployeeRepository.ts (<500 lines)
│   │   ├── EmployeeValidator.ts (<500 lines)
│   │   └── index.ts
│   └── payroll/
│       ├── PayrollService.ts
│       ├── PayrollCalculator.ts
│       └── PayrollRepository.ts
```

### Rationale
- Enforces single responsibility
- Easier code reviews
- Better testability
- Clear module boundaries

### Consequences
- ✅ Maintainable code
- ✅ Easy to understand
- ✅ Parallel development
- ❌ More files to manage
- ❌ Some code duplication
- ❌ Need clear interfaces

---

## ADR-007: Tax Calculation Strategy

**Date**: December 2024  
**Status**: Accepted

### Context
Tax calculations must be 100% accurate and update annually. Need to balance accuracy, maintainability, and cost.

### Options Considered
1. **Build from scratch**
   - Pros: Full control, no dependencies
   - Cons: Complex, high maintenance

2. **Use tax service API**
   - Pros: Always current, maintained by experts
   - Cons: Internet required, ongoing cost

3. **Commercial tax library**
   - Pros: Proven accuracy, support
   - Cons: Expensive, licensing

4. **Hybrid implementation**
   - Pros: Control with verification
   - Cons: More complex

### Decision
**Build from scratch with official examples verification**

### Rationale
- Full offline capability required
- Can verify against IRS/California examples
- One-time development cost
- Annual update is manageable
- Transparency for audits

### Implementation
- Parse official PDF tax tables
- Implement calculations per documentation
- Extensive testing against examples
- Annual update process

### Consequences
- ✅ No ongoing costs
- ✅ Full offline operation
- ✅ Complete control
- ✅ Audit transparency
- ❌ Annual maintenance required
- ❌ Risk of errors
- ❌ Need tax expertise

---

## ADR-008: Error Handling Strategy

**Date**: December 2024  
**Status**: Accepted

### Context
Payroll errors can have serious consequences. Need robust error handling that helps users recover.

### Options Considered
1. **Exceptions everywhere**
   - Pros: Standard approach
   - Cons: Can crash application

2. **Result types** (Either/Result pattern)
   - Pros: Explicit error handling, type-safe
   - Cons: Verbose, unfamiliar to some

3. **Error boundaries + exceptions**
   - Pros: Graceful degradation
   - Cons: React-specific

### Decision
**Result types for business logic, Error boundaries for UI**

### Implementation
```typescript
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

// Business logic returns Result
function calculatePayroll(): Result<PayrollCalculation> {
  // ...
}

// UI uses error boundaries
<ErrorBoundary fallback={<PayrollErrorFallback />}>
  <PayrollCalculator />
</ErrorBoundary>
```

### Rationale
- Explicit error handling in business logic
- Can't ignore errors
- Graceful UI degradation
- Good error messages for users

### Consequences
- ✅ Robust error handling
- ✅ Type-safe errors
- ✅ Better user experience
- ❌ More verbose code
- ❌ Learning curve for Result type

---

## ADR-009: Build and Deployment

**Date**: December 2024  
**Status**: Accepted

### Context
Need to distribute application to Windows, macOS, and Linux users with automatic updates.

### Options Considered
1. **Manual distribution**
   - Pros: Simple, full control
   - Cons: No auto-updates, manual process

2. **App stores** (Windows Store, Mac App Store)
   - Pros: Trust, easy distribution
   - Cons: Review process, restrictions, fees

3. **GitHub Releases + electron-updater**
   - Pros: Free, automatic updates, control
   - Cons: Need code signing certificates

4. **Custom update server**
   - Pros: Full control
   - Cons: Infrastructure cost, complexity

### Decision
**GitHub Releases + electron-updater**

### Rationale
- Free hosting with GitHub
- Automatic updates
- Version control integration
- Code signing for trust
- Falls back to manual if needed

### Implementation
- electron-builder for packaging
- GitHub Actions for CI/CD
- Code signing certificates for each platform
- Staged rollout capability

### Consequences
- ✅ Free distribution
- ✅ Automatic updates
- ✅ Version control
- ❌ Need code signing certificates ($$$)
- ❌ GitHub dependency
- ❌ Size limits on releases

---

## ADR-010: Performance Strategy

**Date**: December 2024  
**Status**: Accepted

### Context
Must handle payroll for 1000+ employees efficiently. Users expect responsive UI during calculations.

### Options Considered
1. **Everything in main process**
   - Pros: Simple architecture
   - Cons: Blocks UI

2. **Web Workers**
   - Pros: Non-blocking UI
   - Cons: Complex data transfer

3. **Worker threads** (Node.js)
   - Pros: True parallelism
   - Cons: Complex architecture

4. **Strategic optimization**
   - Pros: Simple until needed
   - Cons: May need refactoring

### Decision
**Strategic optimization with worker threads for batch operations**

### Implementation
```typescript
// Normal operations in main process
async function calculateSinglePayroll() {
  // Fast enough for single employee
}

// Batch operations use worker threads
async function calculateBatchPayroll(employeeIds: number[]) {
  const workers = new WorkerPool(cpuCount);
  const chunks = chunkArray(employeeIds, 50);
  
  const results = await Promise.all(
    chunks.map(chunk => workers.process(chunk))
  );
  
  return flatten(results);
}
```

### Rationale
- Keep simple operations simple
- Optimize only where needed
- Maintain responsive UI
- Use all CPU cores for batch operations

### Consequences
- ✅ Good performance
- ✅ Responsive UI
- ✅ Scalable approach
- ❌ More complex for batch operations
- ❌ Memory usage for workers

---

## ADR-011: Testing Strategy

**Date**: December 2024  
**Status**: Accepted

### Context
Payroll calculations must be 100% accurate. Need comprehensive testing without slowing development.

### Options Considered
1. **100% coverage mandate**
   - Pros: Thorough testing
   - Cons: Slows development, false confidence

2. **Critical path testing only**
   - Pros: Fast development
   - Cons: Risk of bugs

3. **Risk-based testing**
   - Pros: Balanced approach
   - Cons: Requires judgment

### Decision
**Risk-based with 95% coverage for calculations**

### Implementation
- 95% coverage for calculation engine
- 80% coverage for other code
- Integration tests for critical paths
- Property-based tests for calculations
- Visual regression tests for pay stubs

### Rationale
- Focus testing where it matters most
- Calculations are critical
- UI can be tested manually
- Balance speed and safety

### Consequences
- ✅ High confidence in calculations
- ✅ Reasonable development speed
- ✅ Good bug detection
- ❌ Some code less tested
- ❌ Requires test maintenance

---

## ADR-012: Logging and Monitoring

**Date**: December 2024  
**Status**: Accepted

### Context
Need to troubleshoot issues in production without compromising sensitive data.

### Options Considered
1. **Local logging only**
   - Pros: No privacy concerns
   - Cons: Hard to troubleshoot

2. **Cloud logging** (with PII scrubbing)
   - Pros: Central troubleshooting
   - Cons: Privacy concerns, cost

3. **Hybrid approach**
   - Pros: Flexible, safe
   - Cons: More complex

### Decision
**Local structured logging with optional anonymous telemetry**

### Implementation
```typescript
// Structured logging
logger.info('Payroll calculated', {
  employeeId: hash(employee.id), // Anonymous
  duration: calculationTime,
  recordCount: timecards.length,
  // Never log: SSN, wages, taxes
});

// Optional telemetry
telemetry.track('payroll.calculated', {
  duration: calculationTime,
  employeeCount: count,
  version: app.version
});
```

### Rationale
- Respect privacy
- Enable troubleshooting
- Optional improvement data
- No sensitive data leaves device

### Consequences
- ✅ Privacy preserved
- ✅ Good troubleshooting
- ✅ Performance insights
- ❌ No central monitoring
- ❌ Harder remote support

---

## Future Considerations

### Potential Architecture Changes

1. **Multi-tenant SaaS**
   - Would require database redesign
   - Different security model
   - API-first architecture

2. **Mobile Companion App**
   - View-only pay stubs
   - Time entry
   - Would need API layer

3. **Cloud Backup**
   - Optional encrypted backup
   - Sync between devices
   - Privacy considerations

4. **Plugin Architecture**
   - Custom reports
   - Third-party integrations
   - Would need sandboxing

### Technology Upgrades

1. **Database Migration**
   - PostgreSQL for concurrency
   - When: Multiple simultaneous users
   - Impact: Minor with abstraction layer

2. **WebAssembly Calculations**
   - For performance
   - When: >5000 employees
   - Impact: Calculation engine rewrite

3. **GraphQL Layer**
   - For flexible queries
   - When: API needed
   - Impact: New API layer

## Decision Log

| ADR | Decision | Date | Status |
|-----|----------|------|--------|
| 001 | Electron Platform | Dec 2024 | Accepted |
| 002 | SQLite Database | Dec 2024 | Accepted |
| 003 | Zustand State Management | Dec 2024 | Accepted |
| 004 | Traditional IPC | Dec 2024 | Accepted |
| 005 | Hybrid Encryption | Dec 2024 | Accepted |
| 006 | Modular Architecture | Dec 2024 | Accepted |
| 007 | Build Tax Calculations | Dec 2024 | Accepted |
| 008 | Result Types | Dec 2024 | Accepted |
| 009 | GitHub Distribution | Dec 2024 | Accepted |
| 010 | Strategic Performance | Dec 2024 | Accepted |
| 011 | Risk-based Testing | Dec 2024 | Accepted |
| 012 | Local Logging | Dec 2024 | Accepted |

## Review Schedule

- Quarterly architecture review
- Annual technology assessment
- As-needed for major features

## Conclusion

These architectural decisions provide a solid foundation for the ArcPay system while maintaining flexibility for future changes. The emphasis on simplicity (KISS), modularity, and security ensures the system can evolve with changing requirements while maintaining reliability and performance.