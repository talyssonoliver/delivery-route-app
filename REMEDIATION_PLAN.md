# Security Remediation Plan
**Generated:** 2025-10-22
**Repository:** delivery-route-app
**Branch:** claude/security-repo-audit-011CUN7KDSLYFa8ox6XXMWur

## Quick Start - High Priority Fixes

### 1. Add Subresource Integrity (30 minutes)

**Current Risk:** HIGH
**Effort:** LOW
**Impact:** Prevents CDN compromise attacks

#### Step 1: Generate SRI Hashes

```bash
# Generate SHA-384 hash for Tesseract.js
curl -s https://cdn.jsdelivr.net/npm/tesseract.js@5.0.5/dist/tesseract.min.js | \
openssl dgst -sha384 -binary | openssl base64 -A
```

#### Step 2: Update Script Tags

Replace lines 212-214 in `index.html`:

```html
<!-- BEFORE (Vulnerable) -->
<script src="https://cdn.jsdelivr.net/npm/tesseract.js@5.0.5/dist/tesseract.min.js"></script>

<!-- AFTER (Secure) -->
<script src="https://cdn.jsdelivr.net/npm/tesseract.js@5.0.5/dist/tesseract.min.js"
        integrity="sha384-[GENERATED_HASH_HERE]"
        crossorigin="anonymous"></script>
```

**Alternative:** Use fallback CDNs with SRI:

```html
<script>
(function loadTesseractWithFallback() {
  const cdns = [
    {
      url: 'https://cdn.jsdelivr.net/npm/tesseract.js@5.0.5/dist/tesseract.min.js',
      integrity: 'sha384-[HASH1]'
    },
    {
      url: 'https://unpkg.com/tesseract.js@v5.0.5/dist/tesseract.min.js',
      integrity: 'sha384-[HASH2]'
    },
    {
      url: 'https://cdnjs.cloudflare.com/ajax/libs/tesseract.js/5.0.5/tesseract.min.js',
      integrity: 'sha384-[HASH3]'
    }
  ];

  let currentIndex = 0;

  function tryLoadNext() {
    if (currentIndex >= cdns.length) {
      console.error('❌ All CDNs failed to load Tesseract.js');
      window.tesseractLoadFailed = true;
      return;
    }

    const cdn = cdns[currentIndex];
    const script = document.createElement('script');
    script.src = cdn.url;
    script.integrity = cdn.integrity;
    script.crossOrigin = 'anonymous';
    script.async = false;

    script.onload = () => {
      console.log(`✅ Loaded Tesseract.js from: ${new URL(cdn.url).hostname}`);
      window.tesseractLoadFailed = false;
    };

    script.onerror = () => {
      console.warn(`⚠️ Failed to load from: ${cdn.url}`);
      currentIndex++;
      tryLoadNext();
    };

    document.head.appendChild(script);
  }

  tryLoadNext();
})();
</script>
```

---

### 2. Implement Content Security Policy (2 hours)

**Current Risk:** HIGH
**Effort:** MEDIUM
**Impact:** Mitigates XSS and injection attacks

#### Add to `<head>` section (after line 6):

```html
<meta http-equiv="Content-Security-Policy"
      content="
        default-src 'self';
        script-src 'self' 'unsafe-inline'
                   https://cdn.jsdelivr.net
                   https://unpkg.com
                   https://cdnjs.cloudflare.com;
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: blob:;
        font-src 'self' data:;
        connect-src 'self';
        frame-ancestors 'none';
        base-uri 'self';
        form-action 'self';
      ">
```

#### Testing CSP

1. Add the CSP header
2. Open browser console (F12)
3. Check for CSP violation errors
4. Adjust policy if needed

**Note:** `'unsafe-inline'` is needed for inline styles/scripts. Consider moving to external files in future.

---

### 3. Add Security Headers (1 hour)

**Current Risk:** HIGH
**Effort:** LOW
**Impact:** Defense-in-depth protection

#### Add to `<head>` section:

```html
<!-- Prevent clickjacking -->
<meta http-equiv="X-Frame-Options" content="DENY">

<!-- Prevent MIME sniffing -->
<meta http-equiv="X-Content-Type-Options" content="nosniff">

<!-- Referrer policy for privacy -->
<meta name="referrer" content="no-referrer">

<!-- Permissions policy -->
<meta http-equiv="Permissions-Policy"
      content="geolocation=(), microphone=(), camera=()">
```

---

## Medium Priority Fixes (Within 30 Days)

### 4. File Type Validation (1 hour)

**Current Risk:** MEDIUM
**Location:** index.html:1304-1329

#### Add Magic Number Validation

Replace the file validation section:

```javascript
// Current validation (MIME type only)
if (!f.type.startsWith('image/')) {
  debugLog(`Rejected ${f.name}: not an image`, null, 'warning');
  return false;
}

// Enhanced validation (Magic numbers)
async function validateImageFile(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();

    reader.onload = (e) => {
      const arr = new Uint8Array(e.target.result).subarray(0, 4);
      let header = '';
      for (let i = 0; i < arr.length; i++) {
        header += arr[i].toString(16).padStart(2, '0');
      }

      // Check magic numbers
      const validHeaders = {
        '89504e47': 'PNG',
        'ffd8ffe0': 'JPEG',
        'ffd8ffe1': 'JPEG',
        'ffd8ffe2': 'JPEG',
        'ffd8ffe3': 'JPEG',
        '47494638': 'GIF',
        '424d': 'BMP',
        '49492a00': 'TIFF',
        '4d4d002a': 'TIFF'
      };

      const isValid = Object.keys(validHeaders).some(h =>
        header.startsWith(h.slice(0, Math.min(header.length, h.length)))
      );

      resolve(isValid);
    };

    reader.onerror = () => reject(false);
    reader.readAsArrayBuffer(file.slice(0, 4));
  });
}

// Usage in file selection
document.getElementById('imageInput').addEventListener('change', async (e) => {
  const files = Array.from(e.target.files);
  debugLog(`File input changed: ${files.length} files selected`);

  const validatedFiles = [];

  for (const f of files) {
    // Check MIME type
    if (!f.type.startsWith('image/')) {
      debugLog(`Rejected ${f.name}: not an image (MIME)`, null, 'warning');
      continue;
    }

    // Check file size
    if (f.size > 10485760) {
      debugLog(`Rejected ${f.name}: too large (${formatFileSize(f.size)})`, null, 'warning');
      continue;
    }

    // Validate magic number
    const isValidImage = await validateImageFile(f);
    if (!isValidImage) {
      debugLog(`Rejected ${f.name}: invalid image format (magic number)`, null, 'warning');
      continue;
    }

    validatedFiles.push(f);
  }

  if (validatedFiles.length < files.length) {
    showWarning(`Some files were skipped (invalid format or too large)`);
  }

  images = validatedFiles;
  debugLog(`Accepted ${images.length} validated images`);

  updateImageList();
  updateProcessButton();
});
```

---

### 5. Create Package.json (4 hours)

**Current Risk:** MEDIUM
**Impact:** Better dependency management

#### Step 1: Initialize Package

```bash
cd /home/user/delivery-route-app
npm init -y
```

#### Step 2: Add Dependencies

```json
{
  "name": "delivery-route-app",
  "version": "1.0.0",
  "description": "Delivery Route Generator with OCR",
  "main": "index.html",
  "scripts": {
    "dev": "python3 -m http.server 8000",
    "build": "npm run lint && npm run test",
    "lint": "eslint index.html --ext .js",
    "test": "echo \"No tests yet\" && exit 0",
    "security": "npm audit"
  },
  "dependencies": {
    "tesseract.js": "^5.0.5"
  },
  "devDependencies": {
    "eslint": "^8.0.0",
    "html-validate": "^8.0.0"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/talyssonoliver/delivery-route-app"
  },
  "keywords": ["delivery", "route", "ocr", "logistics"],
  "author": "Talysson Da Silva Oliveira",
  "license": "MIT"
}
```

#### Step 3: Install Dependencies Locally

```bash
npm install
```

#### Step 4: Update HTML to Use Local Dependencies

```html
<!-- Replace CDN script with local -->
<script src="node_modules/tesseract.js/dist/tesseract.min.js"></script>
```

---

### 6. Code Refactoring (16 hours)

**Current Risk:** MEDIUM (Technical Debt)
**Impact:** Improved maintainability

#### Proposed Structure

```
delivery-route-app/
├── index.html (structure only, ~100 lines)
├── css/
│   └── styles.css (styling, ~200 lines)
├── js/
│   ├── app.js (main application logic)
│   ├── ocr-processor.js (OCR functionality)
│   ├── address-parser.js (UK address parsing)
│   ├── route-generator.js (route generation)
│   └── utils.js (utilities and helpers)
├── package.json
└── README.md
```

#### Phase 1: Extract CSS (2 hours)

Create `css/styles.css` and move all `<style>` content.

#### Phase 2: Extract JavaScript (8 hours)

**app.js** - Main application:
```javascript
import { initializeOcrWorker, processImages } from './ocr-processor.js';
import { generateRoutes } from './route-generator.js';
import { escapeHtml, debugLog } from './utils.js';

// Main application logic
```

**ocr-processor.js** - OCR functionality:
```javascript
export async function initializeOcrWorker() { /* ... */ }
export async function processImages(images) { /* ... */ }
export async function preprocessImage(file) { /* ... */ }
```

**address-parser.js** - Address parsing:
```javascript
export function parseOcrToAddresses(text) { /* ... */ }
export function validateAndCleanAddresses(addresses) { /* ... */ }
export function reconstructFromLine(line) { /* ... */ }
```

**route-generator.js** - Route generation:
```javascript
export function generateRoutes(addresses) { /* ... */ }
export function createAppleMapsUrl(addresses) { /* ... */ }
```

**utils.js** - Utilities:
```javascript
export function escapeHtml(s) { /* ... */ }
export function debugLog(message, data, level) { /* ... */ }
export function formatFileSize(bytes) { /* ... */ }
```

#### Phase 3: Update HTML (2 hours)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1.0">

  <!-- Security Headers -->
  <meta http-equiv="Content-Security-Policy" content="...">
  <meta http-equiv="X-Frame-Options" content="DENY">
  <meta http-equiv="X-Content-Type-Options" content="nosniff">
  <meta name="referrer" content="no-referrer">

  <title>Delivery Route Generator</title>
  <link rel="stylesheet" href="css/styles.css">
</head>
<body>
  <div class="container">
    <!-- Application structure -->
  </div>

  <script src="node_modules/tesseract.js/dist/tesseract.min.js"
          integrity="sha384-..."
          crossorigin="anonymous"></script>
  <script type="module" src="js/app.js"></script>
</body>
</html>
```

#### Phase 4: Testing (4 hours)

- Test all functionality
- Verify OCR processing
- Check route generation
- Validate error handling

---

## Low Priority Enhancements (Future)

### 7. Add Progressive Enhancement

```html
<noscript>
  <div class="alert alert-error" style="margin: 20px; padding: 20px;">
    <h2>JavaScript Required</h2>
    <p>This application requires JavaScript to function. Please enable JavaScript in your browser settings.</p>
    <p><strong>Why JavaScript is required:</strong></p>
    <ul>
      <li>Client-side OCR processing (Tesseract.js)</li>
      <li>Image preprocessing</li>
      <li>Address parsing and validation</li>
      <li>Route generation</li>
    </ul>
    <p>All processing happens in your browser - no data is sent to external servers.</p>
  </div>
</noscript>
```

---

### 8. Implement Error Logging

```javascript
// Add to app.js
class ErrorLogger {
  constructor() {
    this.errors = [];
    this.maxErrors = 100;
  }

  log(error, context = {}) {
    const entry = {
      timestamp: new Date().toISOString(),
      message: error.message,
      stack: error.stack,
      context,
      userAgent: navigator.userAgent
    };

    this.errors.push(entry);

    // Keep only last N errors
    if (this.errors.length > this.maxErrors) {
      this.errors.shift();
    }

    // Log to console
    console.error('Error logged:', entry);

    // Could send to monitoring service here
    // this.sendToMonitoring(entry);
  }

  export() {
    return JSON.stringify(this.errors, null, 2);
  }

  clear() {
    this.errors = [];
  }
}

const errorLogger = new ErrorLogger();

// Usage
window.addEventListener('error', (e) => {
  errorLogger.log(e.error, {
    filename: e.filename,
    lineno: e.lineno,
    colno: e.colno
  });
});

window.addEventListener('unhandledrejection', (e) => {
  errorLogger.log(new Error(e.reason), {
    type: 'unhandled-promise-rejection'
  });
});
```

---

### 9. Add Automated Security Testing

#### Create `.github/workflows/security.yml`

```yaml
name: Security Scan

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  security:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Run Gitleaks (Secret Scanning)
      uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    - name: Run npm audit
      run: |
        npm install
        npm audit --audit-level=moderate

    - name: Run ESLint Security Plugin
      run: |
        npm install -g eslint eslint-plugin-security
        eslint . --ext .js --plugin security

    - name: Check for outdated dependencies
      run: npm outdated || true

    - name: Generate SBOM
      run: |
        npm install -g @cyclonedx/cyclonedx-npm
        cyclonedx-npm --output-file sbom.json

    - name: Upload SBOM
      uses: actions/upload-artifact@v3
      with:
        name: sbom
        path: sbom.json
```

---

### 10. Create SECURITY.md

```markdown
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 3.3.x   | :white_check_mark: |
| < 3.0   | :x:                |

## Reporting a Vulnerability

If you discover a security vulnerability, please email security@[domain].com

**Please do not:**
- Open a public issue
- Disclose the vulnerability publicly before it's fixed

**Please include:**
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

We aim to respond within 48 hours and provide a fix within 7 days for critical issues.

## Security Measures

This application implements:
- ✅ Client-side only processing (no data transmission)
- ✅ XSS protection via HTML escaping
- ✅ Content Security Policy
- ✅ Subresource Integrity for CDN resources
- ✅ Input validation and sanitization
- ✅ No cookies or tracking

## Regular Audits

Security audits are conducted quarterly. Last audit: 2025-10-22

## Dependencies

We use:
- Tesseract.js v5.0.5 (OCR library)

Dependencies are reviewed monthly for known vulnerabilities.
```

---

## Verification Checklist

After implementing fixes, verify:

- [ ] SRI hashes added to all external scripts
- [ ] Content Security Policy implemented and tested
- [ ] All security headers added (X-Frame-Options, etc.)
- [ ] File validation includes magic number check
- [ ] package.json created with dependencies
- [ ] Code refactored into modules (optional)
- [ ] Error logging implemented (optional)
- [ ] SECURITY.md created
- [ ] All changes tested in multiple browsers
- [ ] No console errors or CSP violations
- [ ] Application functions correctly with new security measures

---

## Testing Commands

```bash
# Test SRI integrity
curl -s https://cdn.jsdelivr.net/npm/tesseract.js@5.0.5/dist/tesseract.min.js | \
openssl dgst -sha384 -binary | openssl base64 -A

# Check for secrets
git secrets --scan

# Run security audit
npm audit

# Test with strict CSP
# (Use browser developer tools to check CSP violations)
```

---

## Support

For questions about this remediation plan:
- Review the security audit report (SECURITY_AUDIT_REPORT.md)
- Check the findings JSON (security-findings.json)
- Consult the findings CSV (security-findings.csv)

**Timeline:**
- High Priority: Complete within 7 days
- Medium Priority: Complete within 30 days
- Low Priority: Future enhancements

**Estimated Total Effort:**
- High Priority: 3.5 hours
- Medium Priority: 21 hours
- Low Priority: 48+ hours

---

*Last Updated: 2025-10-22*
