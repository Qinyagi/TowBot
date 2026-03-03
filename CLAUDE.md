# CLAUDE.md — TowBot Codebase Guide

## Project Overview

**TowBot** is a browser-based Single Page Application (SPA) for managing and communicating aircraft towing actions at commercial airports. It is engineered as a **fully self-contained HTML file** with no backend, no npm dependencies, and no external network calls during operation — making it suitable for air-gapped and security-sensitive environments.

- **Current Version:** 4.1.3
- **Language:** German (UI and domain terminology)
- **Deployment:** GitHub Pages (static file hosting)
- **License:** Embedded third-party libraries under MIT

---

## Repository Structure

```
TowBot/
├── README.md                    # One-line project description
├── index.html                   # 16-section documentation Exposé + test data generator
├── Ttowbot_V._4.1.3.html       # Full application (main artifact, ~829 KB)
└── .nojekyll                    # GitHub Pages: disables Jekyll processing
```

There are **no build directories**, **no node_modules**, **no package.json**, and **no configuration files beyond `.nojekyll`**. All configuration is embedded in the HTML files or stored in LocalStorage at runtime.

### File Roles

| File | Role |
|---|---|
| `Ttowbot_V._4.1.3.html` | **The application.** All functionality lives here: HTML structure, CSS styling, JavaScript logic, and embedded PDF libraries. |
| `index.html` | **Documentation + demo.** A 16-section Exposé covering architecture, security, DSGVO compliance, and a live test data generator. Embeds the app in an `<iframe>`. |
| `README.md` | Minimal. One-liner description only. |
| `.nojekyll` | Tells GitHub Pages not to process files through Jekyll. |

---

## Technology Stack

- **HTML5 / CSS3 / JavaScript ES6+** — no transpilation, no bundler
- **jsPDF 3.0.1** (MIT) — PDF generation entirely in the browser; embedded verbatim
- **jsPDF-AutoTable 5.0.2** (MIT) — table layout for PDFs; embedded verbatim
- **Browser APIs:** Clipboard API, LocalStorage, Blob/Download, Service Worker (with graceful fallback)

No CDN links are used. All dependencies are bundled directly into the HTML file. The most recent commit (`f947d05`) specifically eliminated the last CDN dependency by embedding jsPDF locally.

---

## Development Workflow

### Editing the Application

1. Open `Ttowbot_V._4.1.3.html` in a text editor.
2. The file is structured as a single HTML document with:
   - `<head>` containing all CSS (via `<style>` tags)
   - `<body>` containing the full UI
   - `<script>` blocks containing all JavaScript (at end of body)
   - Two large embedded `<script>` blocks for jsPDF and jsPDF-AutoTable
3. There is **no build step**. The edited file is the deployable artifact.

### Testing Changes

There is no automated test suite. Testing is done manually using the built-in test data generator in `index.html`:

1. Open `index.html` in a browser.
2. Navigate to **Section 14: Testwert-Generator**.
3. Click **"Neue Testdaten generieren"** to generate weighted random test data.
4. Click **"📋 In Zwischenablage kopieren"** to copy TSV data to clipboard.
5. Open `Ttowbot_V._4.1.3.html` (or the embedded iframe).
6. Click **"Daten einfügen"** to import from clipboard.
7. Verify data appears, change detection works, PDF/EML generation is functional.

### Deployment

Deployment is automatic via **GitHub Pages**. Push to the `master` branch and GitHub Pages serves the files at the repository's Pages URL.

```bash
git add Ttowbot_V._4.1.3.html
git commit -m "Describe the change clearly"
git push origin master
```

---

## Architecture & Key Conventions

### Data Flow

```
Clipboard (TSV input)
  → importData()
  → parseClipboardData()
  → markEventChanges()
  → LocalStorage (towbot_data)
  → PDF (buildPDFBase64) / EML (buildAndDownloadEML)
```

### LocalStorage Schema

All persistent state is stored in `localStorage` under these keys:

| Key | Content |
|---|---|
| `towbot_data` | Main JSON data object (airlines, iterations, hangar events) |
| `towbot_last_shift` | Last recorded shift code (`F`, `S`, or `N`) |
| `towbot_theme` | Active theme name string |
| `towbot_theme_settings` | Theme configuration JSON |
| `towbot_welcome_dismissed` | Boolean flag for welcome modal |

### Main Data Object Shape

```javascript
{
  airlines: [
    {
      code: "EW",           // 2–3 uppercase alphanumeric
      name: "Eurowings",
      emails: "ops@...",    // semicolon-separated
      logo: "[base64]"      // image/png or image/jpeg, max 2 MB
    }
  ],
  iterations: [
    {
      id: 1,
      type: "DOKUMENT",     // or "UPDATE"
      number: "001",
      airline: "EW",
      timestamp: "2026-03-02 14:30:45",
      shift: "F",           // F=Früh, S=Spät, N=Nacht
      data: [
        {
          _id: "D-ABCD|LH456",          // Registration|Flight Dep
          _status: "new|changed|cancelled",
          _comment: "",
          "Registration": "D-ABCD",
          "Aircraft Type": "A320",
          "Flight In": "LH456",
          "Flight Dep": "LH457",
          "From POS": "B12",
          "To POS": "C05",
          "STA": "14:00",
          "STD": "15:30",
          "Schl. Schleppbeginn": "14:10",
          "Tat. Schleppbeginn": "14:12"
        }
      ]
    }
  ],
  hangarEvents: [],
  theme: "standard",
  version: "4.1.3"
}
```

### Naming Conventions

- **JavaScript functions:** camelCase (e.g., `importData`, `parseClipboardData`, `buildPDFBase64`)
- **CSS classes:** kebab-case (e.g., `.import-btn`, `.email-success`, `.theme-option`)
- **Data field names:** Quoted strings with spaces, matching TSV column headers (e.g., `"Tat. Schleppbeginn"`)
- **Airline codes:** 2–3 uppercase alphanumeric characters (e.g., `EW`, `FR`, `X3`)
- **Shift codes:** Single uppercase letter (`F` = Früh/Morning, `S` = Spät/Late, `N` = Nacht/Night)
- **Row IDs:** Pipe-delimited composite key `Registration|FlightDep`

---

## Key Functions Reference

### Data Import

```javascript
importData()                            // Entry point for clipboard import
readClipboardWithRetry(maxRetries)      // Clipboard API with fallback
parseClipboardData(rawText)             // TSV → row array
generateRowId(row)                      // "REG|FLIGHT_DEP" composite key
markEventChanges(previousIter, currentData)  // Diff detection: new/changed/cancelled
```

### Export / Output

```javascript
buildPDFBase64(airlineCode, data, ...)  // Generates PDF as base64 string
downloadBase64PDF(base64, filename)     // Triggers browser download
buildAndDownloadEML(to, subject, html)  // Generates RFC-822 .eml file
buildEmailHTMLBody(airlineCode, ...)    // Builds HTML email body
buildHangarPDFBase64(airlineCode, ...) // Generates hangar-specific PDF
exportAppWithMasterData()              // Full JSON export for backup
```

### UI Navigation

```javascript
showSection(sectionName)               // Shows/hides UI sections
switchTheme(theme)                     // Applies theme from settings
initializeTheme()                      // Loads theme from LocalStorage on startup
toggleThemeMenu() / closeThemeMenu()   // Theme selector dropdown
```

### Data Management

```javascript
deleteAirline(code)
deleteIteration(id)
clearAllIterations()
clearArchive()
clearCompletedEvents()
clearCancelledEvents()
clearHangarEvents()
updateIterationsView()
compareIteration(id)                   // Visual diff of two iterations
getCurrentShiftKey()                   // Returns "F", "S", or "N"
checkShiftReset()                      // Detects shift boundary crossings
```

---

## Security Conventions

The application is intentionally security-hardened. Maintain these practices in all changes:

- **HTML entity escaping** for all user-supplied content rendered into the DOM
- **Regex validation** for airline codes: `/^[A-Z0-9]{2,3}$/`
- **Email validation** before sending: `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
- **File type restriction** for logo uploads: images only (`image/*`), max 2 MB
- **Filename sanitization** applied before any file download
- **No external HTTP requests** during normal operation — all resources are embedded
- **Clipboard API** used with user permission model; graceful fallback to `execCommand`

---

## Domain Terminology (German → English)

| German | English |
|---|---|
| Schleppbeginn | Tow start time |
| Tat. Schleppbeginn | Actual tow start |
| Schl. Schleppbeginn | Scheduled tow start |
| Früh (F) | Morning shift |
| Spät (S) | Late shift |
| Nacht (N) | Night shift |
| Dokument | Initial iteration / first issue |
| Update | Revised iteration |
| Vorfeld | Apron |
| Hangar | Hangar |
| REG | Aircraft registration (tail number) |
| POS | Apron/stand position |

---

## What Not to Do

- **Do not introduce external CDN links.** The last CDN dependency was removed intentionally. All libraries must be embedded.
- **Do not add a build step** unless it's absolutely necessary and agreed upon. The simplicity of the single-file model is a feature.
- **Do not add a backend or database.** The app is deliberately stateless server-side and DSGVO-compliant by design.
- **Do not split the application** into multiple JS/CSS files unless restructuring to a proper bundled project is explicitly requested.
- **Do not use `innerHTML` with unsanitized user input.** Always escape before rendering.
- **Do not commit large binary assets** (logos, images) directly. Logos are stored as base64 in LocalStorage.

---

## Git Workflow

- **Main branch:** `master` (auto-deploys to GitHub Pages)
- **Feature/AI branches:** `claude/<session-id>` pattern
- **Commit messages:** Descriptive, in English or German consistent with existing log style
- **No CI/CD pipeline** — GitHub Pages deploys automatically on push to `master`

Recent commit history style for reference:
```
jsPDF lokal eingebettet – CDN-Abhängigkeit eliminiert
Add welcome modal (Anschreiben) on page load
Add TowBot V4.1.3 Exposé + App für GitHub Pages
Initial commit
```
