# Packaging a Vite App for SCORM / xAPI

## Why `base: './'` is critical

Vite defaults to `base: '/'` — absolute paths. Inside an LMS iframe, the absolute path refers to the LMS server, not your course files. Every asset (JS, CSS, images) will 404.

Always set `base: './'` in `vite.config.ts`:

```typescript
// vite.config.ts
export default defineConfig({
  base: './',
  // ...rest of config
});
```

---

## Manifest file location

Place the manifest in `public/` — Vite copies everything in `public/` to `dist/` root unchanged.

| Standard | File | Location |
|----------|------|----------|
| SCORM 1.2 | `imsmanifest.xml` | `public/imsmanifest.xml` → `dist/imsmanifest.xml` |
| SCORM 2004 | `imsmanifest.xml` | `public/imsmanifest.xml` → `dist/imsmanifest.xml` |
| xAPI | `tincan.xml` | `public/tincan.xml` → `dist/tincan.xml` |

**Important:** An xAPI package must NOT contain `imsmanifest.xml`. If both files are in `public/`, the LMS will detect SCORM first and try to use the SCORM API — which won't exist. For xAPI packaging, use a filter in the packer to exclude `imsmanifest.xml`.

---

## SCORM packaging script (pack-scorm.mjs)

```javascript
// pack-scorm.mjs
import AdmZip from 'adm-zip';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const distDir   = path.join(__dirname, 'dist');
const outFile   = path.join(__dirname, '..', 'course-scorm.zip');

if (!fs.existsSync(distDir)) { console.error('dist/ not found – run npm run build first'); process.exit(1); }
if (fs.existsSync(outFile))  fs.unlinkSync(outFile);

const zip = new AdmZip();
zip.addLocalFolder(distDir, '');
zip.writeZip(outFile);

const mb = (fs.statSync(outFile).size / 1024 / 1024).toFixed(2);
console.log(`✓ ${outFile}  (${mb} MB)`);
zip.getEntries().forEach(e => console.log('  ' + e.entryName));
```

```json
// package.json scripts
"pack-scorm": "vite build && node pack-scorm.mjs"
```

```
npm install --save-dev adm-zip
```

---

## xAPI packaging script (pack-xapi.mjs)

Excludes `imsmanifest.xml` to prevent LMS misdetection.

```javascript
// pack-xapi.mjs
import AdmZip from 'adm-zip';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const distDir   = path.join(__dirname, 'dist');
const outFile   = path.join(__dirname, '..', 'course-xapi.zip');

if (!fs.existsSync(distDir)) { console.error('dist/ not found – run npm run build first'); process.exit(1); }
if (fs.existsSync(outFile))  fs.unlinkSync(outFile);

const zip = new AdmZip();
// Exclude imsmanifest.xml — its presence causes LMSes to treat the package as SCORM
zip.addLocalFolder(distDir, '', (filename) => filename !== 'imsmanifest.xml');
zip.writeZip(outFile);

const mb = (fs.statSync(outFile).size / 1024 / 1024).toFixed(2);
console.log(`✓ ${outFile}  (${mb} MB)`);
zip.getEntries().forEach(e => console.log('  ' + e.entryName));
```

```json
"pack-xapi": "vite build && node pack-xapi.mjs"
```

---

## ZIP structure requirements

### SCORM (1.2 or 2004)
```
course.zip
├── imsmanifest.xml   ← MUST be at root
├── index.html
└── assets/
    ├── index-xxx.js
    └── index-xxx.css
```

### xAPI (Tin Can)
```
course.zip
├── tincan.xml        ← MUST be at root
├── index.html
└── assets/
    └── ...
```

The manifest must be at the ZIP root — not inside a subfolder. If the ZIP contains a folder and you open it to find `course/imsmanifest.xml`, the LMS will reject it.

---

## Standalone (single-file) packaging

For delivery without an LMS, use `vite-plugin-singlefile` to inline all assets into a single `index.html`:

```typescript
// vite.standalone.config.ts
import { viteSingleFile } from 'vite-plugin-singlefile';

export default defineConfig({
  base: './',
  plugins: [react(), tailwindcss(), viteSingleFile()],
  build: {
    outDir: 'dist-standalone',
    cssCodeSplit: false,
  },
});
```

```json
"pack-standalone": "vite build --config vite.standalone.config.ts && node pack-standalone.mjs"
```

The standalone ZIP contains only `index.html` (+ any manifest). No `assets/` folder needed.

---

## Common packaging mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `base: '/'` (default) | 404 on all assets inside LMS | Set `base: './'` |
| Manifest not in `public/` | Not copied to `dist/` | Move to `public/` |
| `imsmanifest.xml` in xAPI ZIP | LMS detects as SCORM, SCORM API not found | Filter it out in packer |
| Manifest inside a subfolder in ZIP | LMS rejects upload | Pack directly from `dist/`, not from a parent folder |
| Images in project root (not in `public/`) | May not be copied to `dist/` | Put images in `public/` or `src/assets/` and import them in code |
| Large images not compressed | ZIP > 100MB | Compress images before packaging |
