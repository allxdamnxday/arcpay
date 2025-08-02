# ArcPay - Ironworkers Payroll System

## Overview

ArcPay is a comprehensive desktop payroll application designed specifically for California ironworkers unions. It handles complex wage calculations, zone-based pay rates, union-specific overtime rules, and ensures full compliance with federal and state tax regulations.

## Features

- **Zone-Based Wage Calculations**: 5-zone system with automatic rate determination
- **Complex Overtime Rules**: Union-specific overtime, including 4-10 schedules
- **Tax Compliance**: Federal and California tax calculations with current tables
- **Multi-Project Support**: Track time across multiple projects and contractors  
- **Fringe Benefits**: Automated calculation of health, pension, vacation, and training contributions
- **Certified Payroll**: Generate compliant reports for government projects
- **Secure & Auditable**: Encrypted sensitive data with complete audit trails

## Documentation

Comprehensive documentation is available in the `/docs` directory:

- [Project Analysis](./docs/PROJECT_ANALYSIS.md) - Project overview and timeline
- [Implementation Roadmap](./docs/IMPLEMENTATION_ROADMAP.md) - Development phases and milestones
- [Database Schema](./docs/DATABASE_SCHEMA.md) - Complete database design
- [API Specification](./docs/API_SPECIFICATION.md) - IPC communication specs
- [Calculation Specifications](./docs/CALCULATION_SPECIFICATIONS.md) - Detailed calculation algorithms
- [Security & Compliance](./docs/SECURITY_COMPLIANCE.md) - Security requirements and compliance

For development setup, see [DEVELOPMENT_SETUP.md](./DEVELOPMENT_SETUP.md).

## Quick Start

```bash
# Clone the repository
git clone https://github.com/allxdamnxday/arcpay.git
cd arcpay

# Install dependencies
npm install

# Set up the database
npm run db:migrate

# Start development mode
npm run dev
```

## Technology Stack

- **Frontend**: React 18 + TypeScript + Material-UI
- **Backend**: Electron + Node.js
- **Database**: SQLite with better-sqlite3
- **Security**: Field-level encryption, bcrypt for passwords
- **Build**: Webpack + electron-builder
- **Testing**: Jest + Playwright

## System Requirements

- **OS**: Windows 10+, macOS 10.15+, Ubuntu 20.04+
- **RAM**: 4GB minimum, 8GB recommended
- **Storage**: 500MB for application + database growth
- **Display**: 1280x720 minimum resolution

## Development

See [DEVELOPMENT_SETUP.md](./DEVELOPMENT_SETUP.md) for detailed setup instructions.

### Key Commands

```bash
npm run dev          # Start development mode
npm run test         # Run test suite
npm run lint         # Lint code
npm run build        # Build for production
npm run dist         # Create distributable
```

## Architecture

```
src/
├── main/           # Electron main process
├── renderer/       # React application
├── shared/         # Shared types and utilities
└── database/       # Database layer
```

## Contributing

1. Follow the coding guidelines in [CLAUDE.md](./CLAUDE.md)
2. Write tests for new features
3. Ensure all tests pass before submitting PR
4. Keep files under 500 lines (modular architecture)
5. Follow the KISS principle

## Security

- All PII is encrypted at rest
- Secure IPC communication
- Role-based access control
- Complete audit logging
- Regular security updates

## License

Copyright © 2024 ArcPay. All rights reserved.

This is proprietary software. Unauthorized copying, distribution, or use is strictly prohibited.

## Support

For technical documentation, see the `/docs` directory.
For issues, please use the GitHub issue tracker.