# Build and Deployment Guide

## Production Build Overview

This guide covers building and deploying the ArcPay Payroll application for Windows, macOS, and Linux platforms.

## Prerequisites

### General Requirements
- Complete development setup (see DEVELOPMENT_SETUP.md)
- Node.js 18+ with npm 9+
- Clean git working directory
- All tests passing

### Platform-Specific Requirements

#### Windows
- Windows 10/11 development machine (or Wine on Linux/macOS)
- Code signing certificate (.pfx file)
- Windows SDK (for native modules)

#### macOS
- macOS 10.15+ development machine
- Apple Developer ID certificates
- Xcode Command Line Tools
- Notarization credentials

#### Linux
- Ubuntu 20.04+ or equivalent
- Build essentials package
- AppImage tools

## Build Configuration

### package.json Scripts
```json
{
  "scripts": {
    "build": "npm run build:prod && npm run dist",
    "build:prod": "cross-env NODE_ENV=production webpack --config webpack.prod.js",
    "dist": "electron-builder",
    "dist:win": "electron-builder --win",
    "dist:mac": "electron-builder --mac",
    "dist:linux": "electron-builder --linux"
  }
}
```

### electron-builder Configuration
```javascript
// electron-builder.config.js
module.exports = {
  appId: "com.arcpay.payroll",
  productName: "ArcPay Payroll",
  directories: {
    output: "dist",
    buildResources: "build"
  },
  files: [
    "build/**/*",
    "node_modules/**/*",
    "package.json"
  ],
  win: {
    target: ["nsis", "portable"],
    icon: "build/icons/icon.ico",
    certificateFile: process.env.WINDOWS_CERT_FILE,
    certificatePassword: process.env.WINDOWS_CERT_PASSWORD
  },
  mac: {
    target: ["dmg", "zip"],
    icon: "build/icons/icon.icns",
    category: "public.app-category.finance",
    hardenedRuntime: true,
    gatekeeperAssess: false,
    entitlements: "build/entitlements.mac.plist",
    entitlementsInherit: "build/entitlements.mac.plist"
  },
  linux: {
    target: ["AppImage", "deb"],
    icon: "build/icons",
    category: "Office"
  }
}
```

## Build Process

### 1. Pre-Build Checklist

```bash
# Verify clean working directory
git status

# Run all tests
npm test
npm run test:e2e

# Check for security vulnerabilities
npm audit

# Update version number
npm version patch/minor/major

# Update changelog
echo "## v$(node -p "require('./package.json').version") - $(date +%Y-%m-%d)" >> CHANGELOG.md
```

### 2. Environment Setup

Create `.env.production` file:
```bash
NODE_ENV=production
ELECTRON_IS_DEV=0
AUTO_UPDATE_URL=https://your-update-server.com
SENTRY_DSN=your-sentry-dsn
```

### 3. Build Commands

#### Development Build (for testing)
```bash
npm run build:prod
npm run dist -- --publish never
```

#### Production Build

##### Windows
```bash
# Set environment variables
export WINDOWS_CERT_FILE=./build/certs/certificate.pfx
export WINDOWS_CERT_PASSWORD=your-password

# Build
npm run dist:win
```

##### macOS
```bash
# Ensure certificates are in keychain
security find-identity -v -p codesigning

# Build and sign
npm run dist:mac

# Notarize (automated via electron-notarize)
# Requires APPLE_ID and APPLE_ID_PASSWORD env vars
```

##### Linux
```bash
# Install required packages
sudo apt-get install -y rpm fakeroot dpkg

# Build
npm run dist:linux
```

### 4. Build Output

Builds are generated in the `dist/` directory:
- Windows: `.exe` installer and `.exe` portable
- macOS: `.dmg` installer and `.zip` archive
- Linux: `.AppImage` and `.deb` packages

## Code Signing

### Windows Code Signing

1. Obtain code signing certificate from trusted CA
2. Convert to .pfx format if needed
3. Set environment variables:
```bash
export CSC_LINK=path/to/certificate.pfx
export CSC_KEY_PASSWORD=certificate-password
```

### macOS Code Signing

1. Install Developer ID certificates
2. Configure electron-builder to use certificates
3. Enable hardened runtime
4. Set up notarization:
```javascript
// scripts/notarize.js
const { notarize } = require('electron-notarize');

exports.default = async function notarizing(context) {
  const { electronPlatformName, appOutDir } = context;
  if (electronPlatformName !== 'darwin') return;

  const appName = context.packager.appInfo.productFilename;
  
  return await notarize({
    appBundleId: 'com.arcpay.payroll',
    appPath: `${appOutDir}/${appName}.app`,
    appleId: process.env.APPLE_ID,
    appleIdPassword: process.env.APPLE_ID_PASSWORD,
  });
};
```

## Auto-Update Configuration

### 1. Set Up Update Server

Options:
- GitHub Releases (free, simple)
- Amazon S3 (scalable, reliable)
- Custom server (full control)

### 2. Configure electron-updater

```javascript
// src/main/updater.js
const { autoUpdater } = require('electron-updater');

function configureUpdater() {
  autoUpdater.setFeedURL({
    provider: 'github',
    owner: 'your-org',
    repo: 'arcpay',
    private: false
  });

  autoUpdater.checkForUpdatesAndNotify();
  
  // Check for updates every 4 hours
  setInterval(() => {
    autoUpdater.checkForUpdatesAndNotify();
  }, 4 * 60 * 60 * 1000);
}
```

### 3. Version Management

Follow semantic versioning:
- MAJOR.MINOR.PATCH (e.g., 1.2.3)
- Update package.json version before build
- Tag releases in git

## Deployment Process

### 1. Final Testing

```bash
# Install production build locally
# Test all critical paths:
- Employee management
- Time entry and editing
- Payroll calculations
- Report generation
- Database operations
- Auto-update mechanism
```

### 2. Create Release

#### GitHub Releases
```bash
# Create git tag
git tag -a v1.2.3 -m "Release version 1.2.3"
git push origin v1.2.3

# Upload artifacts to GitHub Release
# Use GitHub UI or gh CLI
```

#### Custom Server
```bash
# Upload to your server
scp dist/*.exe user@server:/releases/windows/
scp dist/*.dmg user@server:/releases/mac/
scp dist/*.AppImage user@server:/releases/linux/

# Update latest.yml files for auto-updater
```

### 3. Database Migrations

For updates that include database changes:
```javascript
// src/main/database/migrator.js
async function runMigrations() {
  const currentVersion = await db.getVersion();
  const migrations = await getMigrationsSince(currentVersion);
  
  for (const migration of migrations) {
    await db.transaction(() => {
      migration.up();
      db.setVersion(migration.version);
    });
  }
}
```

### 4. Rollback Plan

Always maintain ability to rollback:
```bash
# Keep previous versions available
/releases/
  /1.2.3/   # Current
  /1.2.2/   # Previous
  /1.2.1/   # Older

# Database rollback scripts
/migrations/
  /rollback-1.2.3.sql
```

## Post-Deployment

### 1. Monitoring

Set up monitoring for:
- Application crashes (Sentry)
- Update adoption rates
- Performance metrics
- User feedback

### 2. Error Tracking

```javascript
// src/main/sentry.js
const Sentry = require('@sentry/electron');

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  beforeSend(event) {
    // Remove sensitive data
    delete event.user;
    return event;
  }
});
```

### 3. Analytics (Optional)

Privacy-compliant usage tracking:
```javascript
// Only track with user consent
if (userConsent) {
  analytics.track('app_launched', {
    version: app.getVersion(),
    platform: process.platform
  });
}
```

## Troubleshooting Deployment Issues

### Common Build Errors

1. **Native module rebuild failures**
```bash
# Clean and rebuild
rm -rf node_modules
npm install
npm rebuild
```

2. **Code signing failures**
- Verify certificate validity
- Check certificate password
- Ensure proper certificate chain

3. **Notarization failures (macOS)**
- Check Apple ID credentials
- Verify app bundle ID matches
- Review notarization log

### User Installation Issues

1. **Windows SmartScreen**
- Sign with EV certificate
- Build reputation over time

2. **macOS Gatekeeper**
- Ensure proper notarization
- Provide clear installation instructions

3. **Linux permissions**
- Make AppImage executable
- Document required system libraries

## Security Considerations

### Release Security
- Sign all distributed binaries
- Use HTTPS for update server
- Implement update signature verification
- Regular security audits

### Data Protection
- Encrypt sensitive data at rest
- Secure communication channels
- Regular backup verification
- GDPR/CCPA compliance

## Documentation Updates

After each release:
1. Update user documentation
2. Document known issues
3. Update API documentation
4. Notify users of changes

## Continuous Deployment (Optional)

GitHub Actions example:
```yaml
name: Build and Release
on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    strategy:
      matrix:
        os: [windows-latest, macos-latest, ubuntu-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm ci
      - run: npm test
      - run: npm run dist
      - uses: softprops/action-gh-release@v1
        with:
          files: dist/*
```