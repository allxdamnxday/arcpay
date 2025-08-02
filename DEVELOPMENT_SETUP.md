# Development Setup Guide

## Prerequisites
- Node.js 18+ and npm 9+
- Git
- Visual Studio Code (recommended)
- Windows 10/11, macOS 10.15+, or Linux

## Initial Setup

### 1. Clone Repository
```bash
git clone [repository-url]
cd arcpay
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Environment Configuration
Create `.env.local` file in the root directory:
```
NODE_ENV=development
ELECTRON_IS_DEV=1
```

### 4. Database Setup
```bash
npm run db:migrate    # Run database migrations
npm run db:seed       # Load sample data (optional)
```

## Development Workflow

### Running the Application
```bash
npm run dev           # Start webpack dev server (hot reload)
npm start             # Start Electron app in development mode
```

### Code Quality Tools
```bash
npm run lint          # Run ESLint
npm run format        # Format code with Prettier
npm run typecheck     # TypeScript type checking
npm test              # Run Jest test suite
```

## VS Code Setup

### Recommended Extensions
- ESLint
- Prettier - Code formatter
- TypeScript and JavaScript Language Features
- SQLite Viewer
- Jest Runner
- Material Icon Theme (optional)

### Settings (.vscode/settings.json)
```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "editor.rulers": [80, 120],
  "files.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.git": true
  }
}
```

## Project Structure

### Key Directories
- `src/main/` - Electron main process code
- `src/renderer/` - React frontend application
- `src/shared/` - Shared business logic and types
- `src/database/` - Database schemas and migrations
- `tests/` - Test files mirroring src structure

### Development Patterns
- Use TypeScript strict mode
- Follow React hooks best practices
- Implement proper error boundaries
- Use IPC for main/renderer communication

## Troubleshooting

### Common Issues

1. **Electron not starting**
   - Check Node.js version: `node --version`
   - Rebuild native modules: `npm rebuild`

2. **Database connection errors**
   - Ensure SQLite3 is properly built: `npm rebuild better-sqlite3`
   - Check database file permissions

3. **TypeScript errors**
   - Run `npm run typecheck` to see all errors
   - Ensure VS Code is using workspace TypeScript version

4. **Hot reload not working**
   - Check webpack dev server is running
   - Verify no port conflicts on 3000

### Debug Configuration

Create `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Main Process",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceFolder}",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "args": ["."],
      "outputCapture": "std"
    },
    {
      "name": "Debug Renderer Process",
      "type": "chrome",
      "request": "attach",
      "port": 9223,
      "webRoot": "${workspaceFolder}/src/renderer",
      "timeout": 30000
    }
  ]
}
```

## Database Development

### Running Migrations
```bash
npm run db:migrate:create -- migration-name  # Create new migration
npm run db:migrate:up                        # Run pending migrations
npm run db:migrate:down                      # Rollback last migration
```

### Sample Data
The seed script creates:
- 5 sample employees (different classifications)
- 3 sample projects (different zones)
- Current wage rates for all zones
- Sample timecards for testing

## Testing During Development

### Unit Tests
```bash
npm test                    # Run all tests
npm test -- --watch         # Watch mode
npm test -- --coverage      # Generate coverage report
```

### E2E Testing
```bash
npm run test:e2e           # Run Electron E2E tests
```

### Manual Testing Checklist
- [ ] Time entry with various overtime scenarios
- [ ] Zone-based wage calculations
- [ ] Tax withholding calculations
- [ ] Report generation
- [ ] Database backup/restore

## Performance Profiling

### React DevTools
1. Install React Developer Tools browser extension
2. Use Profiler tab to identify performance bottlenecks

### Electron Performance
```bash
npm run start -- --inspect  # Enable Chrome DevTools
```

## Getting Help

- Check `docs/` folder for detailed documentation
- Review existing code patterns in similar modules
- Consult CLAUDE.md for AI assistance guidelines