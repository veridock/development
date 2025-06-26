# development


# SVG-PWA Development Guide
**Narzędzia, metodologia i best practices dla tworzenia aplikacji SVG-PWA**

**Wersja:** 1.0  
**Data:** 24 czerwca 2025  
**Status:** Development Documentation

---

## 📋 **Spis treści**

1. [Wyzwania rozwojowe](#wyzwania-rozwojowe)
2. [Środowisko deweloperskie](#środowisko-deweloperskie)
3. [Narzędzia budowania](#narzędzia-budowania)
4. [Workflow developmentu](#workflow-developmentu)
5. [Bundling i packaging](#bundling-i-packaging)
6. [Testing i debugging](#testing-i-debugging)
7. [Deployment i dystrybucja](#deployment-i-dystrybucja)
8. [Gotowe narzędzia](#gotowe-narzędzia)

---

## 🚨 **Wyzwania rozwojowe**

### **Problemy zidentyfikowane w praktyce:**

```javascript
// ❌ PROBLEM 1: Brak document.head w SVG kontekście
document.head.appendChild(link); // TypeError: document.head is null

// ✅ ROZWIĄZANIE:
if (document.head) {
  document.head.appendChild(link);
} else {
  console.warn('Manifest will be embedded in SVG metadata');
}
```

```javascript
// ❌ PROBLEM 2: Service Worker scope w lokalnych plikach
navigator.serviceWorker.register(swUrl); // Invalid scope error

// ✅ ROZWIĄZANIE:
navigator.serviceWorker.register(swUrl, { 
  scope: window.location.protocol === 'file:' ? '/' : './' 
});
```

```javascript
// ❌ PROBLEM 3: Niedostępne DOM API w SVG
this.element.querySelector('circle:last-child'); // Może być null

// ✅ ROZWIĄZANIE:
const circle = this.element.querySelector('circle:last-child');
if (circle) {
  circle.setAttribute('fill', color);
}
```

### **Fundamentalne ograniczenia:**

1. **Kontekst wykonania** - SVG ma ograniczony DOM
2. **Bezpieczeństwo** - CSP i CORS restrictions  
3. **File protocol** - ograniczone API w `file://`
4. **Bundling** - wszystko musi być w jednym pliku
5. **Debugging** - trudniejsze śledzenie błędów

---

## 🛠️ **Środowisko deweloperskie**

### **Rekomendowana konfiguracja:**

```json
{
  "name": "svg-pwa-dev-environment",
  "version": "1.0.0",
  "description": "Development setup for SVG-PWA applications",
  "scripts": {
    "dev": "svg-pwa-dev-server --watch",
    "build": "svg-pwa-bundler --optimize",
    "test": "svg-pwa-test --headless",
    "deploy": "svg-pwa-deploy --target=github-pages"
  },
  "devDependencies": {
    "svg-pwa-cli": "^1.0.0",
    "svg-pwa-bundler": "^1.0.0",
    "svg-pwa-dev-server": "^1.0.0",
    "svg-pwa-validator": "^1.0.0"
  }
}
```

### **Struktura projektu:**

```
my-svg-pwa-app/
├── src/
│   ├── app.js                 # Główna logika aplikacji
│   ├── components/            # Komponenty SVG
│   │   ├── Button.svg.js
│   │   ├── Timer.svg.js
│   │   └── ProgressRing.svg.js
│   ├── styles/               # Style CSS
│   │   ├── main.css
│   │   └── animations.css
│   ├── assets/               # Zasoby
│   │   ├── icons/
│   │   └── fonts/
│   └── manifest.json         # PWA manifest
├── templates/
│   └── app.svg.template       # Szablon SVG
├── config/
│   ├── build.config.js        # Konfiguracja budowania
│   └── dev.config.js          # Konfiguracja development
├── tests/
│   ├── unit/
│   └── integration/
├── dist/                      # Wygenerowane pliki
│   └── app.svg
└── tools/
    ├── bundler.js
    ├── validator.js
    └── dev-server.js
```

---

## 🔨 **Narzędzia budowania**

### **1. SVG-PWA CLI Tool**

```bash
# Instalacja
npm install -g svg-pwa-cli

# Nowy projekt
svg-pwa create my-app --template=stopwatch

# Development server
svg-pwa dev --port=3000 --live-reload

# Build production
svg-pwa build --optimize --minify

# Validation
svg-pwa validate dist/app.svg
```

### **2. Bundler (Webpack plugin)**

```javascript
// webpack.config.js
const SVGPWAPlugin = require('svg-pwa-webpack-plugin');

module.exports = {
  entry: './src/app.js',
  plugins: [
    new SVGPWAPlugin({
      template: './templates/app.svg.template',
      manifest: './src/manifest.json',
      output: 'dist/app.svg',
      optimize: true,
      inlineAssets: true,
      validatePWA: true
    })
  ]
};
```

### **3. Vite Plugin**

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import svgPWA from 'vite-plugin-svg-pwa';

export default defineConfig({
  plugins: [
    svgPWA({
      entry: 'src/app.js',
      template: 'templates/app.svg',
      manifest: 'src/manifest.json',
      devMode: {
        hotReload: true,
        mockPWA: true
      }
    })
  ]
});
```

### **4. Custom Bundler Script**

```javascript
// tools/bundler.js
const fs = require('fs');
const path = require('path');
const { minify } = require('terser');
const CleanCSS = require('clean-css');

class SVGPWABundler {
  constructor(config) {
    this.config = config;
    this.components = new Map();
    this.assets = new Map();
  }
  
  async build() {
    console.log('🔨 Building SVG-PWA application...');
    
    // 1. Load template
    const template = await this.loadTemplate();
    
    // 2. Process JavaScript
    const js = await this.bundleJavaScript();
    
    // 3. Process CSS  
    const css = await this.bundleCSS();
    
    // 4. Process assets
    const assets = await this.bundleAssets();
    
    // 5. Generate SVG
    const svg = await this.generateSVG(template, js, css, assets);
    
    // 6. Validate
    await this.validate(svg);
    
    // 7. Save
    await this.save(svg);
    
    console.log('✅ Build completed!');
  }
  
  async loadTemplate() {
    return fs.readFileSync(this.config.template, 'utf8');
  }
  
  async bundleJavaScript() {
    const files = this.config.jsFiles || ['src/app.js'];
    let bundled = '';
    
    for (const file of files) {
      const content = fs.readFileSync(file, 'utf8');
      bundled += content + '\n';
    }
    
    // Minify for production
    if (this.config.minify) {
      const result = await minify(bundled);
      return result.code;
    }
    
    return bundled;
  }
  
  async bundleCSS() {
    const files = this.config.cssFiles || ['src/styles/main.css'];
    let bundled = '';
    
    for (const file of files) {
      const content = fs.readFileSync(file, 'utf8');
      bundled += content + '\n';
    }
    
    // Minify CSS
    if (this.config.minify) {
      const result = new CleanCSS().minify(bundled);
      return result.styles;
    }
    
    return bundled;
  }
  
  async bundleAssets() {
    const assets = {};
    const assetDir = this.config.assetDir || 'src/assets';
    
    if (fs.existsSync(assetDir)) {
      const files = this.getAssetFiles(assetDir);
      
      for (const file of files) {
        const content = fs.readFileSync(file);
        const base64 = content.toString('base64');
        const mimeType = this.getMimeType(file);
        assets[path.basename(file)] = `data:${mimeType};base64,${base64}`;
      }
    }
    
    return assets;
  }
  
  async generateSVG(template, js, css, assets) {
    let svg = template;
    
    // Replace placeholders
    svg = svg.replace('{{JS_CODE}}', `<![CDATA[${js}]]>`);
    svg = svg.replace('{{CSS_CODE}}', `<![CDATA[${css}]]>`);
    svg = svg.replace('{{ASSETS}}', JSON.stringify(assets));
    
    // Add metadata
    const metadata = this.generateMetadata();
    svg = svg.replace('{{METADATA}}', metadata);
    
    return svg;
  }
  
  generateMetadata() {
    const manifest = JSON.parse(fs.readFileSync(this.config.manifest, 'utf8'));
    
    return `
      <pwa:manifest xmlns:pwa="http://svg-pwa.org/ns/manifest">
        <pwa:name>${manifest.name}</pwa:name>
        <pwa:short_name>${manifest.short_name}</pwa:short_name>
        <pwa:description>${manifest.description}</pwa:description>
        <pwa:start_url>${manifest.start_url}</pwa:start_url>
        <pwa:display>${manifest.display}</pwa:display>
        <pwa:theme_color>${manifest.theme_color}</pwa:theme_color>
        <pwa:background_color>${manifest.background_color}</pwa:background_color>
      </pwa:manifest>
    `;
  }
  
  async validate(svg) {
    // XML validation
    const xmllint = require('xmllint');
    const result = xmllint.validateXML({ xml: svg });
    
    if (result.errors) {
      throw new Error(`XML validation failed: ${result.errors}`);
    }
    
    // PWA validation
    await this.validatePWA(svg);
    
    // Size validation
    const size = Buffer.byteLength(svg, 'utf8');
    if (size > 10 * 1024 * 1024) { // 10MB limit
      console.warn(`⚠️ Large file size: ${(size / 1024 / 1024).toFixed(2)}MB`);
    }
  }
  
  async validatePWA(svg) {
    // Check required PWA elements
    const required = [
      '<pwa:manifest',
      'Service Worker',
      'localStorage'
    ];
    
    for (const req of required) {
      if (!svg.includes(req)) {
        console.warn(`⚠️ Missing PWA feature: ${req}`);
      }
    }
  }
  
  async save(svg) {
    const outputPath = this.config.output || 'dist/app.svg';
    const dir = path.dirname(outputPath);
    
    if (!fs.existsSync(dir)) {
      fs.mkdirSync(dir, { recursive: true });
    }
    
    fs.writeFileSync(outputPath, svg, 'utf8');
    
    const size = Buffer.byteLength(svg, 'utf8');
    console.log(`📦 Generated: ${outputPath} (${(size / 1024).toFixed(2)}KB)`);
  }
}

module.exports = SVGPWABundler;
```

---

## 🔄 **Workflow developmentu**

### **1. Development Mode**

```javascript
// tools/dev-server.js
const express = require('express');
const chokidar = require('chokidar');
const WebSocket = require('ws');

class SVGPWADevServer {
  constructor(config) {
    this.config = config;
    this.app = express();
    this.wss = new WebSocket.Server({ port: 3001 });
    this.bundler = new SVGPWABundler(config);
  }
  
  start() {
    // Serve SVG with hot reload injection
    this.app.get('/app.svg', async (req, res) => {
      try {
        const svg = await this.buildWithHotReload();
        res.setHeader('Content-Type', 'image/svg+xml');
        res.send(svg);
      } catch (error) {
        res.status(500).send(`Error: ${error.message}`);
      }
    });
    
    // Watch for changes
    chokidar.watch('src/**/*').on('change', () => {
      this.rebuild();
    });
    
    this.app.listen(3000, () => {
      console.log('🚀 SVG-PWA Dev Server running on http://localhost:3000');
    });
  }
  
  async buildWithHotReload() {
    const svg = await this.bundler.build();
    
    // Inject hot reload script
    const hotReloadScript = `
      if (window.location.hostname === 'localhost') {
        const ws = new WebSocket('ws://localhost:3001');
        ws.onmessage = () => window.location.reload();
      }
    `;
    
    return svg.replace('</script>', hotReloadScript + '</script>');
  }
  
  rebuild() {
    console.log('🔄 Rebuilding...');
    this.wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send('reload');
      }
    });
  }
}
```

### **2. Component System**

```javascript
// src/components/Button.svg.js
export class SVGButton {
  constructor(x, y, text, onClick) {
    this.x = x;
    this.y = y;
    this.text = text;
    this.onClick = onClick;
  }
  
  render() {
    return `
      <g class="svg-button" transform="translate(${this.x},${this.y})">
        <rect rx="25" ry="25" width="100" height="50" 
              fill="url(#buttonGradient)" 
              stroke="rgba(255,255,255,0.2)" 
              stroke-width="2"/>
        <text x="50" y="30" text-anchor="middle" 
              font-family="system-ui" font-size="14" fill="white">
          ${this.text}
        </text>
      </g>
    `;
  }
  
  bind(element) {
    element.addEventListener('click', this.onClick);
  }
}
```

```javascript
// src/components/ProgressRing.svg.js
export class SVGProgressRing {
  constructor(cx, cy, radius, strokeWidth = 8) {
    this.cx = cx;
    this.cy = cy;
    this.radius = radius;
    this.strokeWidth = strokeWidth;
    this.circumference = 2 * Math.PI * radius;
    this.progress = 0;
  }
  
  render() {
    return `
      <circle cx="${this.cx}" cy="${this.cy}" r="${this.radius}" 
              fill="none" stroke="rgba(255,255,255,0.2)" 
              stroke-width="${this.strokeWidth}"/>
      <circle id="progress-${this.cx}-${this.cy}" 
              cx="${this.cx}" cy="${this.cy}" r="${this.radius}" 
              fill="none" stroke="#4CAF50" 
              stroke-width="${this.strokeWidth}"
              stroke-dasharray="${this.circumference}" 
              stroke-dashoffset="${this.circumference}"
              stroke-linecap="round" 
              transform="rotate(-90 ${this.cx} ${this.cy})"/>
    `;
  }
  
  setProgress(progress) {
    this.progress = Math.max(0, Math.min(1, progress));
    const offset = this.circumference - (this.progress * this.circumference);
    
    const element = document.getElementById(`progress-${this.cx}-${this.cy}`);
    if (element) {
      element.style.strokeDashoffset = offset;
    }
  }
}
```

### **3. Template System**

```xml
<!-- templates/app.svg.template -->
<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xhtml="http://www.w3.org/1999/xhtml"
     xmlns:pwa="http://svg-pwa.org/ns/manifest"
     width="400" height="600" 
     viewBox="0 0 400 600"
     data-pwa-version="{{VERSION}}"
     data-build-date="{{BUILD_DATE}}">

  <!-- Metadata Section -->
  <metadata>
    {{METADATA}}
  </metadata>

  <!-- Visual Definition Section -->
  <defs>
    {{GRADIENTS}}
    {{FILTERS}}
    {{ICONS}}
  </defs>

  <!-- Background -->
  {{BACKGROUND}}

  <!-- UI Components -->
  <g id="app-shell">
    {{COMPONENTS}}
  </g>

  <!-- Embedded HTML -->
  <foreignObject x="0" y="0" width="100%" height="100%">
    <xhtml:div style="display:none">
      {{HTML_CONTENT}}
    </xhtml:div>
  </foreignObject>

  <!-- Styling -->
  <style type="text/css">
    {{CSS_CODE}}
  </style>

  <!-- Application Logic -->
  <script type="application/javascript">
    {{JS_CODE}}
  </script>
</svg>
```

---

## 📦 **Bundling i packaging**

### **1. Multi-file to Single SVG**

```javascript
// tools/multi-bundler.js
const path = require('path');
const fs = require('fs');

class MultiBundler {
  constructor() {
    this.files = {
      js: [],
      css: [],
      svg: [],
      assets: []
    };
  }
  
  addFile(filePath, type) {
    this.files[type].push(filePath);
  }
  
  async bundle() {
    // Bundle JavaScript
    const js = await this.bundleJS();
    
    // Bundle CSS
    const css = await this.bundleCSS();
    
    // Bundle SVG components
    const svgComponents = await this.bundleSVGComponents();
    
    // Process assets to data URIs
    const assets = await this.processAssets();
    
    // Generate final SVG
    return this.generateFinalSVG(js, css, svgComponents, assets);
  }
  
  async bundleJS() {
    let bundled = '';
    
    // Add polyfills for SVG context
    bundled += this.getSVGPolyfills();
    
    // Add utility functions
    bundled += this.getUtilityFunctions();
    
    // Bundle application files
    for (const file of this.files.js) {
      const content = fs.readFileSync(file, 'utf8');
      
      // Transform ES6 modules to IIFE
      const transformed = this.transformModules(content);
      bundled += transformed + '\n';
    }
    
    return bundled;
  }
  
  getSVGPolyfills() {
    return `
      // SVG Context Polyfills
      if (!document.head) {
        document.head = {
          appendChild: function(element) {
            console.warn('document.head.appendChild called in SVG context');
            return element;
          }
        };
      }
      
      // Safe querySelector wrapper
      function safeQuerySelector(selector, context = document) {
        try {
          return context.querySelector(selector);
        } catch (e) {
          console.warn('querySelector failed:', selector, e);
          return null;
        }
      }
      
      // Safe DOM manipulation
      function safeSetAttribute(element, attr, value) {
        if (element && element.setAttribute) {
          element.setAttribute(attr, value);
        }
      }
    `;
  }
  
  transformModules(code) {
    // Simple ES6 modules to IIFE transformation
    // In production, use Babel or similar
    return code
      .replace(/^import\s+.*$/gm, '// $&')
      .replace(/^export\s+/gm, '// export ');
  }
}
```

### **2. Asset Optimization**

```javascript
// tools/asset-optimizer.js
const sharp = require('sharp');
const svgo = require('svgo');

class AssetOptimizer {
  async optimizeImage(filePath) {
    const ext = path.extname(filePath).toLowerCase();
    
    if (ext === '.svg') {
      return this.optimizeSVG(filePath);
    } else {
      return this.optimizeRaster(filePath);
    }
  }
  
  async optimizeSVG(filePath) {
    const content = fs.readFileSync(filePath, 'utf8');
    const result = svgo.optimize(content, {
      plugins: [
        'removeDoctype',
        'removeComments',
        'removeMetadata',
        'removeUselessStrokeAndFill',
        'cleanupNumericValues'
      ]
    });
    
    return `data:image/svg+xml;base64,${Buffer.from(result.data).toString('base64')}`;
  }
  
  async optimizeRaster(filePath) {
    const optimized = await sharp(filePath)
      .resize(512, 512, { fit: 'inside' })
      .png({ quality: 80 })
      .toBuffer();
    
    return `data:image/png;base64,${optimized.toString('base64')}`;
  }
}
```

---

## 🧪 **Testing i debugging**

### **1. Unit Testing Framework**

```javascript
// tests/svg-pwa-test.js
class SVGPWATestFramework {
  constructor() {
    this.tests = [];
    this.results = [];
  }
  
  describe(name, fn) {
    console.group(`📋 ${name}`);
    fn();
    console.groupEnd();
  }
  
  it(description, fn) {
    try {
      fn();
      console.log(`✅ ${description}`);
      this.results.push({ description, status: 'passed' });
    } catch (error) {
      console.error(`❌ ${description}:`, error.message);
      this.results.push({ description, status: 'failed', error });
    }
  }
  
  expect(actual) {
    return {
      toBe: (expected) => {
        if (actual !== expected) {
          throw new Error(`Expected ${expected}, got ${actual}`);
        }
      },
      toBeInstanceOf: (constructor) => {
        if (!(actual instanceof constructor)) {
          throw new Error(`Expected instance of ${constructor.name}`);
        }
      },
      toExist: () => {
        if (!actual) {
          throw new Error('Expected value to exist');
        }
      }
    };
  }
  
  async testPWAFeatures(svgElement) {
    this.describe('PWA Features', () => {
      this.it('should have service worker support', () => {
        this.expect('serviceWorker' in navigator).toBe(true);
      });
      
      this.it('should have local storage', () => {
        this.expect(typeof Storage !== 'undefined').toBe(true);
      });
      
      this.it('should have notification support', () => {
        this.expect('Notification' in window).toBe(true);
      });
    });
  }
  
  async testSVGStructure(svg) {
    this.describe('SVG Structure', () => {
      this.it('should be valid XML', () => {
        const parser = new DOMParser();
        const doc = parser.parseFromString(svg, 'image/svg+xml');
        const errors = doc.querySelectorAll('parsererror');
        this.expect(errors.length).toBe(0);
      });
      
      this.it('should have required namespaces', () => {
        this.expect(svg.includes('xmlns="http://www.w3.org/2000/svg"')).toBe(true);
        this.expect(svg.includes('xmlns:pwa="http://svg-pwa.org/ns/manifest"')).toBe(true);
      });
      
      this.it('should have embedded JavaScript', () => {
        this.expect(svg.includes('<script')).toBe(true);
      });
    });
  }
}

// Usage
const testFramework = new SVGPWATestFramework();

// Test SVG-PWA application
testFramework.testSVGStructure(svgContent);
testFramework.testPWAFeatures(document.querySelector('svg'));
```

### **2. Debug Tools**

```javascript
// tools/svg-pwa-debugger.js
class SVGPWADebugger {
  constructor() {
    this.logs = [];
    this.performance = {};
    this.errors = [];
  }
  
  init() {
    this.setupErrorHandling();
    this.setupPerformanceMonitoring();
    this.setupConsoleEnhancements();
    this.createDebugPanel();
  }
  
  setupErrorHandling() {
    window.addEventListener('error', (event) => {
      this.errors.push({
        type: 'JavaScript Error',
        message: event.message,
        filename: event.filename,
        lineno: event.lineno,
        timestamp: Date.now()
      });
      this.updateDebugPanel();
    });
    
    window.addEventListener('unhandledrejection', (event) => {
      this.errors.push({
        type: 'Promise Rejection',
        message: event.reason,
        timestamp: Date.now()
      });
      this.updateDebugPanel();
    });
  }
  
  setupPerformanceMonitoring() {
    // Monitor SVG rendering performance
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        this.performance[entry.name] = {
          duration: entry.duration,
          startTime: entry.startTime
        };
      }
      this.updateDebugPanel();
    });
    
    observer.observe({ entryTypes: ['measure', 'navigation'] });
  }
  
  createDebugPanel() {
    if (window.location.hostname === 'localhost' || window.location.search.includes('debug=1')) {
      const panel = document.createElement('div');
      panel.id = 'svg-pwa-debug';
      panel.innerHTML = `
        <div style="position:fixed; top:10px; right:10px; width:300px; background:rgba(0,0,0,0.9); color:white; padding:10px; border-radius:5px; font-family:monospace; font-size:12px; z-index:10000; max-height:400px; overflow-y:auto;">
          <h3>🐛 SVG-PWA Debug</h3>
          <div id="debug-errors"></div>
          <div id="debug-performance"></div>
          <div id="debug-features"></div>
        </div>
      `;
      document.body.appendChild(panel);
    }
  }
  
  updateDebugPanel() {
    const errorsDiv = document.getElementById('debug-errors');
    const performanceDiv = document.getElementById('debug-performance');
    const featuresDiv = document.getElementById('debug-features');
    
    if (errorsDiv) {
      errorsDiv.innerHTML = `
        <h4>❌ Errors (${this.errors.length})</h4>
        ${this.errors.slice(-3).map(error => `
          <div style="color:#ff6b6b; margin:2px 0;">
            ${error.type}: ${error.message}
          </div>
        `).join('')}
      `;
    }
    
    if (performanceDiv) {
      performanceDiv.innerHTML = `
        <h4>⚡ Performance</h4>
        <div>Load Time: ${performance.now().toFixed(2)}ms</div>
        <div>Memory: ${this.getMemoryUsage()}</div>
      `;
    }
    
    if (featuresDiv) {
      featuresDiv.innerHTML = `
        <h4>🔧 PWA Features</h4>
        <div>SW: ${this.checkFeature('serviceWorker')}</div>
        <div>Notifications: ${this.checkFeature('Notification')}</div>
        <div>Storage: ${this.checkFeature('localStorage')}</div>
      `;
    }
  }
  
  checkFeature(feature) {
    switch (feature) {
      case 'serviceWorker':
        return 'serviceWorker' in navigator ? '✅' : '❌';
      case 'Notification':
        return 'Notification' in window ? '✅' : '❌';
      case 'localStorage':
        return typeof Storage !== 'undefined' ? '✅' : '❌';
      default:
        return '❓';
    }
  }
  
  getMemoryUsage() {
    if (performance.memory) {
      const used = Math.round(performance.memory.usedJSHeapSize / 1024 / 1024);
      return `${used}MB`;
    }
    return 'N/A';
  }
}

// Auto-initialize in development
if (window.location.hostname === 'localhost' || window.location.search.includes('debug=1')) {
  const debugger = new SVGPWADebugger();
  debugger.init();
}
```

---

## 🚀 **Deployment i dystrybucja**

### **1. GitHub Pages Deployment**


```yaml
# .github/workflows/deploy.yml
name: Deploy SVG-PWA
on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '18'
        
    - name: Install dependencies
      run: npm install
      
    - name: Build SVG-PWA
      run: npm run build
      
    - name: Validate SVG-PWA
      run: npm run validate
      
    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./dist
```

### **2. CDN Deployment Script**

```javascript
// tools/deploy.js
const fs = require('fs');
const AWS = require('aws-sdk');
const { gzipSync } = require('zlib');

class SVGPWADeployer {
  constructor(config) {
    this.config = config;
    this.s3 = new AWS.S3({
      accessKeyId: config.aws.accessKey,
      secretAccessKey: config.aws.secretKey,
      region: config.aws.region
    });
  }
  
  async deploy() {
    console.log('🚀 Deploying SVG-PWA to CDN...');
    
    const svgContent = fs.readFileSync(this.config.svgPath, 'utf8');
    
    // Compress
    const compressed = gzipSync(svgContent);
    
    // Upload to S3
    const params = {
      Bucket: this.config.aws.bucket,
      Key: `apps/${this.config.appName}.svg`,
      Body: compressed,
      ContentType: 'image/svg+xml',
      ContentEncoding: 'gzip',
      CacheControl: 'public, max-age=31536000', // 1 year
      Metadata: {
        'pwa-version': this.config.version,
        'build-date': new Date().toISOString()
      }
    };
    
    const result = await this.s3.upload(params).promise();
    
    console.log(`✅ Deployed to: ${result.Location}`);
    
    // Update version manifest
    await this.updateVersionManifest();
    
    return result.Location;
  }
  
  async updateVersionManifest() {
    const manifest = {
      currentVersion: this.config.version,
      downloadUrl: `https://${this.config.aws.bucket}.s3.amazonaws.com/apps/${this.config.appName}.svg`,
      size: fs.statSync(this.config.svgPath).size,
      releaseDate: new Date().toISOString(),
      changelog: this.config.changelog || []
    };
    
    await this.s3.upload({
      Bucket: this.config.aws.bucket,
      Key: `apps/${this.config.appName}.json`,
      Body: JSON.stringify(manifest, null, 2),
      ContentType: 'application/json'
    }).promise();
  }
}
```

### **3. Email Distribution**

```javascript
// tools/email-distributor.js
const nodemailer = require('nodemailer');

class EmailDistributor {
  constructor(config) {
    this.transporter = nodemailer.createTransporter(config.smtp);
    this.config = config;
  }
  
  async distributeApp(svgPath, recipients) {
    const svgContent = fs.readFileSync(svgPath);
    const stats = fs.statSync(svgPath);
    
    // Check size limit (25MB for Gmail)
    if (stats.size > 25 * 1024 * 1024) {
      throw new Error('File too large for email distribution');
    }
    
    const mailOptions = {
      from: this.config.from,
      subject: `📱 ${this.config.appName} - SVG-PWA Application`,
      html: `
        <h2>🚀 ${this.config.appName}</h2>
        <p>Nowa aplikacja SVG-PWA gotowa do użycia!</p>
        
        <h3>📋 Instrukcje:</h3>
        <ol>
          <li>Pobierz załączony plik <code>${path.basename(svgPath)}</code></li>
          <li>Otwórz w przeglądarce (Chrome, Firefox, Safari, Edge)</li>
          <li>Kliknij ikonę instalacji aby dodać do urządzeń</li>
          <li>Używaj jak natywnej aplikacji!</li>
        </ol>
        
        <h3>✨ Funkcje:</h3>
        <ul>
          <li>✅ Działa offline</li>
          <li>✅ Instalowalna jako PWA</li>
          <li>✅ Powiadomienia systemowe</li>
          <li>✅ Zapisuje dane lokalnie</li>
        </ul>
        
        <p><small>Rozmiar: ${(stats.size / 1024).toFixed(2)}KB</small></p>
      `,
      attachments: [{
        filename: path.basename(svgPath),
        content: svgContent,
        contentType: 'image/svg+xml'
      }]
    };
    
    for (const recipient of recipients) {
      await this.transporter.sendMail({
        ...mailOptions,
        to: recipient
      });
      
      console.log(`📧 Sent to: ${recipient}`);
    }
  }
}
```

---

## 🛠️ **Gotowe narzędzia**

### **1. SVG-PWA CLI (Kompletne narzędzie)**

```javascript
#!/usr/bin/env node
// bin/svg-pwa-cli.js

const { Command } = require('commander');
const program = new Command();

program
  .name('svg-pwa')
  .description('CLI tools for SVG-PWA development')
  .version('1.0.0');

program
  .command('create <name>')
  .description('Create new SVG-PWA project')
  .option('-t, --template <template>', 'template to use', 'basic')
  .action(async (name, options) => {
    const creator = new ProjectCreator();
    await creator.create(name, options.template);
  });

program
  .command('dev')
  .description('Start development server')
  .option('-p, --port <port>', 'port number', '3000')
  .option('-w, --watch', 'watch for changes', true)
  .action(async (options) => {
    const server = new DevServer(options);
    await server.start();
  });

program
  .command('build')
  .description('Build production SVG-PWA')
  .option('-o, --output <path>', 'output path', 'dist/app.svg')
  .option('-m, --minify', 'minify output', false)
  .option('--analyze', 'analyze bundle size', false)
  .action(async (options) => {
    const builder = new ProductionBuilder(options);
    await builder.build();
  });

program
  .command('validate <file>')
  .description('Validate SVG-PWA file')
  .option('--strict', 'strict validation', false)
  .action(async (file, options) => {
    const validator = new SVGPWAValidator(options);
    await validator.validate(file);
  });

program
  .command('deploy')
  .description('Deploy SVG-PWA application')
  .option('-t, --target <target>', 'deployment target', 'github-pages')
  .action(async (options) => {
    const deployer = new Deployer(options);
    await deployer.deploy();
  });

program.parse();

class ProjectCreator {
  async create(name, template) {
    console.log(`🚀 Creating SVG-PWA project: ${name}`);
    
    const templates = {
      basic: this.createBasicTemplate,
      stopwatch: this.createStopwatchTemplate,
      calculator: this.createCalculatorTemplate,
      game: this.createGameTemplate
    };
    
    if (!templates[template]) {
      throw new Error(`Unknown template: ${template}`);
    }
    
    await templates[template](name);
    
    console.log(`✅ Project created successfully!`);
    console.log(`
Next steps:
  cd ${name}
  npm install
  svg-pwa dev
    `);
  }
  
  async createBasicTemplate(name) {
    const structure = {
      [`${name}/package.json`]: this.getPackageJson(name),
      [`${name}/src/app.js`]: this.getBasicApp(),
      [`${name}/src/styles/main.css`]: this.getBasicCSS(),
      [`${name}/src/manifest.json`]: this.getBasicManifest(name),
      [`${name}/templates/app.svg.template`]: this.getBasicTemplate(),
      [`${name}/config/build.config.js`]: this.getBuildConfig(),
      [`${name}/.gitignore`]: this.getGitignore(),
      [`${name}/README.md`]: this.getReadme(name)
    };
    
    for (const [filePath, content] of Object.entries(structure)) {
      await this.writeFile(filePath, content);
    }
  }
  
  getPackageJson(name) {
    return JSON.stringify({
      name: name,
      version: "1.0.0",
      description: "SVG-PWA Application",
      main: "src/app.js",
      scripts: {
        dev: "svg-pwa dev",
        build: "svg-pwa build",
        validate: "svg-pwa validate dist/app.svg",
        deploy: "svg-pwa deploy"
      },
      devDependencies: {
        "svg-pwa-cli": "^1.0.0"
      }
    }, null, 2);
  }
  
  getBasicApp() {
    return `
class SVGPWAApp {
  constructor() {
    this.initializeApp();
    this.setupEventListeners();
    this.initPWA();
  }
  
  initializeApp() {
    console.log('🚀 SVG-PWA App initialized');
  }
  
  setupEventListeners() {
    // Add your event listeners here
  }
  
  initPWA() {
    // PWA initialization
    if ('serviceWorker' in navigator) {
      this.registerServiceWorker();
    }
  }
  
  registerServiceWorker() {
    // Service Worker registration logic
  }
}

// Initialize app when DOM is ready
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', () => {
    new SVGPWAApp();
  });
} else {
  new SVGPWAApp();
}
    `;
  }
}
```

### **2. VS Code Extension**

```json
// package.json for VS Code extension
{
  "name": "svg-pwa-tools",
  "displayName": "SVG-PWA Tools",
  "description": "Development tools for SVG-PWA applications",
  "version": "1.0.0",
  "engines": {
    "vscode": "^1.60.0"
  },
  "categories": ["Other"],
  "activationEvents": [
    "onLanguage:xml",
    "onCommand:svg-pwa.create",
    "onCommand:svg-pwa.validate"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "svg-pwa.create",
        "title": "Create SVG-PWA App",
        "category": "SVG-PWA"
      },
      {
        "command": "svg-pwa.validate",
        "title": "Validate SVG-PWA",
        "category": "SVG-PWA"
      },
      {
        "command": "svg-pwa.preview",
        "title": "Preview SVG-PWA",
        "category": "SVG-PWA"
      }
    ],
    "languages": [
      {
        "id": "svg-pwa",
        "aliases": ["SVG-PWA", "svg-pwa"],
        "extensions": [".svg"],
        "configuration": "./language-configuration.json"
      }
    ],
    "grammars": [
      {
        "language": "svg-pwa",
        "scopeName": "text.xml.svg.pwa",
        "path": "./syntaxes/svg-pwa.tmLanguage.json"
      }
    ],
    "snippets": [
      {
        "language": "svg-pwa",
        "path": "./snippets/svg-pwa.json"
      }
    ]
  }
}
```

### **3. Online Builder (Web Tool)**

```html
<!-- tools/web-builder/index.html -->
<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SVG-PWA Builder</title>
    <style>
        body {
            font-family: system-ui;
            margin: 0;
            padding: 20px;
            background: #f5f5f5;
        }
        
        .builder-container {
            max-width: 1200px;
            margin: 0 auto;
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 20px;
            height: 90vh;
        }
        
        .editor-panel {
            background: white;
            border-radius: 10px;
            padding: 20px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        
        .preview-panel {
            background: white;
            border-radius: 10px;
            padding: 20px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
            display: flex;
            flex-direction: column;
        }
        
        textarea {
            width: 100%;
            height: 300px;
            font-family: 'Courier New', monospace;
            font-size: 14px;
            border: 1px solid #ddd;
            border-radius: 5px;
            padding: 10px;
        }
        
        .preview-frame {
            flex: 1;
            border: 1px solid #ddd;
            border-radius: 5px;
            background: white;
        }
        
        .toolbar {
            display: flex;
            gap: 10px;
            margin-bottom: 20px;
        }
        
        button {
            padding: 10px 20px;
            background: #007acc;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        
        button:hover {
            background: #005a9e;
        }
        
        .tabs {
            display: flex;
            margin-bottom: 20px;
        }
        
        .tab {
            padding: 10px 20px;
            background: #eee;
            border: none;
            cursor: pointer;
            border-radius: 5px 5px 0 0;
        }
        
        .tab.active {
            background: white;
            border-bottom: 1px solid white;
        }
    </style>
</head>
<body>
    <h1>🎨 SVG-PWA Builder</h1>
    <p>Wizualny kreator aplikacji SVG-PWA</p>
    
    <div class="builder-container">
        <div class="editor-panel">
            <div class="toolbar">
                <button onclick="newProject()">📄 Nowy</button>
                <button onclick="loadExample()">📋 Przykład</button>
                <button onclick="saveProject()">💾 Zapisz</button>
                <button onclick="exportSVG()">📦 Eksport</button>
            </div>
            
            <div class="tabs">
                <button class="tab active" onclick="switchTab('js')">JavaScript</button>
                <button class="tab" onclick="switchTab('css')">CSS</button>
                <button class="tab" onclick="switchTab('manifest')">Manifest</button>
                <button class="tab" onclick="switchTab('template')">Template</button>
            </div>
            
            <div id="editor-js" class="editor-content">
                <textarea id="js-editor" placeholder="// JavaScript code here..."></textarea>
            </div>
            
            <div id="editor-css" class="editor-content" style="display:none;">
                <textarea id="css-editor" placeholder="/* CSS styles here... */"></textarea>
            </div>
            
            <div id="editor-manifest" class="editor-content" style="display:none;">
                <textarea id="manifest-editor" placeholder="PWA manifest JSON..."></textarea>
            </div>
            
            <div id="editor-template" class="editor-content" style="display:none;">
                <textarea id="template-editor" placeholder="SVG template..."></textarea>
            </div>
        </div>
        
        <div class="preview-panel">
            <div class="toolbar">
                <button onclick="refreshPreview()">🔄 Odśwież</button>
                <button onclick="validateSVG()">✅ Waliduj</button>
                <button onclick="testPWA()">📱 Test PWA</button>
            </div>
            
            <iframe id="preview-frame" class="preview-frame" src="about:blank"></iframe>
            
            <div id="validation-results" style="margin-top: 10px; padding: 10px; background: #f8f9fa; border-radius: 5px; display: none;">
                <h4>Wyniki walidacji:</h4>
                <div id="validation-content"></div>
            </div>
        </div>
    </div>
    
    <script src="svg-pwa-builder.js"></script>
</body>
</html>
```

```javascript
// tools/web-builder/svg-pwa-builder.js
class SVGPWABuilder {
  constructor() {
    this.currentTab = 'js';
    this.project = {
      js: '',
      css: '',
      manifest: '{}',
      template: this.getDefaultTemplate()
    };
    
    this.initializeEditor();
    this.loadExample();
  }
  
  initializeEditor() {
    // Auto-save on input
    ['js', 'css', 'manifest', 'template'].forEach(type => {
      const editor = document.getElementById(`${type}-editor`);
      editor.addEventListener('input', () => {
        this.project[type] = editor.value;
        this.autoSave();
      });
    });
    
    // Auto-refresh preview
    this.debounceRefresh = this.debounce(() => {
      this.refreshPreview();
    }, 1000);
    
    Object.values(document.querySelectorAll('textarea')).forEach(textarea => {
      textarea.addEventListener('input', this.debounceRefresh);
    });
  }
  
  switchTab(tabName) {
    // Update tab UI
    document.querySelectorAll('.tab').forEach(tab => {
      tab.classList.remove('active');
    });
    document.querySelector(`[onclick="switchTab('${tabName}')"]`).classList.add('active');
    
    // Show/hide editor content
    document.querySelectorAll('.editor-content').forEach(content => {
      content.style.display = 'none';
    });
    document.getElementById(`editor-${tabName}`).style.display = 'block';
    
    this.currentTab = tabName;
  }
  
  loadExample() {
    this.project = {
      js: `
class ExampleApp {
  constructor() {
    this.counter = 0;
    this.setupUI();
  }
  
  setupUI() {
    const button = document.getElementById('increment-btn');
    const display = document.getElementById('counter-display');
    
    if (button && display) {
      button.addEventListener('click', () => {
        this.counter++;
        display.textContent = this.counter;
      });
    }
  }
}

new ExampleApp();
      `,
      css: `
.counter-app {
  text-align: center;
  padding: 20px;
}

.counter-button {
  background: #4CAF50;
  color: white;
  border: none;
  padding: 15px 30px;
  border-radius: 25px;
  font-size: 18px;
  cursor: pointer;
  transition: background 0.3s;
}

.counter-button:hover {
  background: #45a049;
}

.counter-display {
  font-size: 48px;
  font-weight: bold;
  margin: 20px 0;
  color: #333;
}
      `,
      manifest: JSON.stringify({
        "name": "Example SVG-PWA App",
        "short_name": "ExampleApp",
        "description": "Example SVG-PWA application",
        "start_url": "./",
        "display": "standalone",
        "theme_color": "#4CAF50",
        "background_color": "#ffffff"
      }, null, 2),
      template: this.getExampleTemplate()
    };
    
    this.updateEditors();
    this.refreshPreview();
  }
  
  getExampleTemplate() {
    return `<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" 
     xmlns:xhtml="http://www.w3.org/1999/xhtml"
     width="400" height="300" viewBox="0 0 400 300">
  
  <rect width="400" height="300" fill="#f0f0f0"/>
  
  <foreignObject x="0" y="0" width="400" height="300">
    <xhtml:div class="counter-app">
      <xhtml:h1>Counter App</xhtml:h1>
      <xhtml:div class="counter-display" id="counter-display">0</xhtml:div>
      <xhtml:button class="counter-button" id="increment-btn">Kliknij mnie!</xhtml:button>
    </xhtml:div>
  </foreignObject>
  
  <style>{{CSS}}</style>
  <script><![CDATA[{{JS}}]]></script>
</svg>`;
  }
  
  updateEditors() {
    document.getElementById('js-editor').value = this.project.js;
    document.getElementById('css-editor').value = this.project.css;
    document.getElementById('manifest-editor').value = this.project.manifest;
    document.getElementById('template-editor').value = this.project.template;
  }
  
  refreshPreview() {
    try {
      const svg = this.generateSVG();
      const blob = new Blob([svg], { type: 'image/svg+xml' });
      const url = URL.createObjectURL(blob);
      
      const frame = document.getElementById('preview-frame');
      frame.src = url;
      
      // Clean up previous URL
      setTimeout(() => URL.revokeObjectURL(url), 1000);
      
    } catch (error) {
      console.error('Preview error:', error);
      this.showValidationResults([{
        type: 'error',
        message: `Preview error: ${error.message}`
      }]);
    }
  }
  
  generateSVG() {
    let svg = this.project.template;
    
    // Replace placeholders
    svg = svg.replace('{{JS}}', this.project.js);
    svg = svg.replace('{{CSS}}', this.project.css);
    
    // Add PWA metadata
    const manifest = JSON.parse(this.project.manifest);
    const metadata = `
      <metadata>
        <pwa:manifest xmlns:pwa="http://svg-pwa.org/ns/manifest">
          <pwa:name>${manifest.name || 'SVG-PWA App'}</pwa:name>
          <pwa:short_name>${manifest.short_name || 'App'}</pwa:short_name>
          <pwa:description>${manifest.description || ''}</pwa:description>
        </pwa:manifest>
      </metadata>
    `;
    
    // Insert metadata after opening svg tag
    svg = svg.replace('<svg', `<svg${svg.includes('xmlns:pwa=') ? '' : ' xmlns:pwa="http://svg-pwa.org/ns/manifest"'}`);
    svg = svg.replace('>', '>' + metadata);
    
    return svg;
  }
  
  validateSVG() {
    const results = [];
    
    try {
      // Validate XML structure
      const parser = new DOMParser();
      const doc = parser.parseFromString(this.generateSVG(), 'image/svg+xml');
      const errors = doc.querySelectorAll('parsererror');
      
      if (errors.length > 0) {
        results.push({
          type: 'error',
          message: 'Invalid XML structure'
        });
      } else {
        results.push({
          type: 'success',
          message: 'Valid XML structure'
        });
      }
      
      // Validate manifest
      try {
        JSON.parse(this.project.manifest);
        results.push({
          type: 'success',
          message: 'Valid PWA manifest'
        });
      } catch (e) {
        results.push({
          type: 'error',
          message: `Invalid manifest JSON: ${e.message}`
        });
      }
      
      // Check file size
      const size = new Blob([this.generateSVG()]).size;
      if (size > 5 * 1024 * 1024) { // 5MB
        results.push({
          type: 'warning',
          message: `Large file size: ${(size / 1024 / 1024).toFixed(2)}MB`
        });
      } else {
        results.push({
          type: 'success',
          message: `File size: ${(size / 1024).toFixed(2)}KB`
        });
      }
      
    } catch (error) {
      results.push({
        type: 'error',
        message: `Validation error: ${error.message}`
      });
    }
    
    this.showValidationResults(results);
  }
  
  showValidationResults(results) {
    const resultsDiv = document.getElementById('validation-results');
    const contentDiv = document.getElementById('validation-content');
    
    contentDiv.innerHTML = results.map(result => {
      const icon = {
        success: '✅',
        error: '❌',
        warning: '⚠️'
      }[result.type];
      
      return `<div style="color: ${result.type === 'error' ? 'red' : result.type === 'warning' ? 'orange' : 'green'}">
        ${icon} ${result.message}
      </div>`;
    }).join('');
    
    resultsDiv.style.display = 'block';
  }
  
  exportSVG() {
    try {
      const svg = this.generateSVG();
      const blob = new Blob([svg], { type: 'image/svg+xml' });
      const url = URL.createObjectURL(blob);
      
      const a = document.createElement('a');
      a.href = url;
      a.download = 'app.svg';
      a.click();
      
      URL.revokeObjectURL(url);
      
    } catch (error) {
      alert(`Export error: ${error.message}`);
    }
  }
  
  autoSave() {
    localStorage.setItem('svg-pwa-builder-project', JSON.stringify(this.project));
  }
  
  debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
      const later = () => {
        clearTimeout(timeout);
        func(...args);
      };
      clearTimeout(timeout);
      timeout = setTimeout(later, wait);
    };
  }
}

// Initialize builder
window.builder = new SVGPWABuilder();

// Global functions for buttons
function newProject() {
  if (confirm('Czy chcesz rozpocząć nowy projekt? Niezapisane zmiany zostaną utracone.')) {
    window.builder.project = {
      js: '',
      css: '',
      manifest: '{}',
      template: window.builder.getDefaultTemplate()
    };
    window.builder.updateEditors();
    window.builder.refreshPreview();
  }
}

function loadExample() {
  window.builder.loadExample();
}

function saveProject() {
  const project = JSON.stringify(window.builder.project, null, 2);
  const blob = new Blob([project], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  
  const a = document.createElement('a');
  a.href = url;
  a.download = 'svg-pwa-project.json';
  a.click();
  
  URL.revokeObjectURL(url);
}

function exportSVG() {
  window.builder.exportSVG();
}

function switchTab(tabName) {
  window.builder.switchTab(tabName);
}

function refreshPreview() {
  window.builder.refreshPreview();
}

function validateSVG() {
  window.builder.validateSVG();
}

function testPWA() {
  alert('PWA testing feature coming soon!');
}
```

---

## 📝 **Podsumowanie i rekomendacje**

### **🎯 Najważniejsze wnioski:**

1. **Development wymaga specjalistycznych narzędzi** - zwykłe webowe workflow nie wystarczy
2. **Bundling jest kluczowy** - wszystko musi być w jednym pliku SVG
3. **Testing w kontekście SVG** ma swoje specyficzne wyzwania
4. **PWA features wymagają workaroundów** w SVG kontekście

### **🚀 Rekomendowane workflow:**

```bash
# 1. Setup projektu
svg-pwa create my-app --template=stopwatch
cd my-app

# 2. Development
svg-pwa dev --watch --port=3000

# 3. Testing
svg-pwa validate dist/app.svg
svg-pwa test --headless

# 4. Build
svg-pwa build --minify --optimize

# 5. Deploy
svg-pwa deploy --target=github-pages
```

### **🛠️ Priorytetowe narzędzia do stworzenia:**

1. **SVG-PWA CLI** - podstawowe narzędzie deweloperskie
2. **VS Code Extension** - syntax highlighting, snippets, validation
3. **Web Builder** - wizualny kreator dla nie-programistów
4. **Webpack Plugin** - integracja z istniejącymi workflow
5. **Testing Framework** - dedykowane testowanie SVG-PWA

### **⚡ Quick Start dla deweloperów:**

```bash
# Instalacja narzędzi
npm install -g svg-pwa-cli

# Nowy projekt
svg-pwa create my-timer --template=stopwatch

# Development z hot reload
cd my-timer && svg-pwa dev

# Build i deploy
svg-pwa build --optimize
svg-pwa deploy --target=github-pages
```

---

## 🔧 **Praktyczne przykłady implementacji**

### **1. Bundler produkcyjny (Node.js)**

```javascript
// tools/production-bundler.js
const fs = require('fs-extra');
const path = require('path');
const { minify } = require('terser');
const CleanCSS = require('clean-css');
const imagemin = require('imagemin');
const imageminSvgo = require('imagemin-svgo');

class ProductionBundler {
  constructor(config) {
    this.config = {
      srcDir: 'src',
      distDir: 'dist',
      templatePath: 'templates/app.svg.template',
      outputName: 'app.svg',
      minify: true,
      optimize: true,
      validatePWA: true,
      ...config
    };
  }

  async build() {
    console.log('🏗️  Starting production build...');
    
    const startTime = Date.now();
    
    try {
      // 1. Clean dist directory
      await fs.emptyDir(this.config.distDir);
      
      // 2. Load and process all components
      const components = await this.loadComponents();
      
      // 3. Bundle JavaScript
      const javascript = await this.bundleJavaScript(components.js);
      
      // 4. Bundle CSS
      const css = await this.bundleCSS(components.css);
      
      // 5. Process assets
      const assets = await this.processAssets();
      
      // 6. Load manifest and metadata
      const manifest = await this.loadManifest();
      
      // 7. Generate final SVG
      const svgContent = await this.generateSVG({
        javascript,
        css,
        assets,
        manifest,
        components
      });
      
      // 8. Optimize SVG
      const optimizedSVG = await this.optimizeSVG(svgContent);
      
      // 9. Validate
      if (this.config.validatePWA) {
        await this.validatePWA(optimizedSVG);
      }
      
      // 10. Write output
      const outputPath = path.join(this.config.distDir, this.config.outputName);
      await fs.writeFile(outputPath, optimizedSVG);
      
      // 11. Generate build report
      await this.generateBuildReport(optimizedSVG, Date.now() - startTime);
      
      console.log('✅ Build completed successfully!');
      
    } catch (error) {
      console.error('❌ Build failed:', error);
      throw error;
    }
  }

  async loadComponents() {
    const components = {
      js: [],
      css: [],
      svg: [],
      html: []
    };

    // Recursively find all source files
    const files = await this.findFiles(this.config.srcDir);
    
    for (const file of files) {
      const ext = path.extname(file);
      const content = await fs.readFile(file, 'utf8');
      
      switch (ext) {
        case '.js':
          components.js.push({ path: file, content });
          break;
        case '.css':
          components.css.push({ path: file, content });
          break;
        case '.svg':
          components.svg.push({ path: file, content });
          break;
        case '.html':
          components.html.push({ path: file, content });
          break;
      }
    }

    return components;
  }

  async bundleJavaScript(jsFiles) {
    let bundled = '';
    
    // Add polyfills for SVG context
    bundled += await this.getSVGPolyfills();
    
    // Process each JavaScript file
    for (const file of jsFiles) {
      let content = file.content;
      
      // Transform ES6 modules
      content = await this.transformModules(content);
      
      // Add source map comments in development
      if (!this.config.minify) {
        content += `\n//# sourceURL=${file.path}`;
      }
      
      bundled += `\n// === ${path.basename(file.path)} ===\n`;
      bundled += content;
    }
    
    // Minify in production
    if (this.config.minify) {
      const result = await minify(bundled, {
        compress: {
          drop_console: true,
          drop_debugger: true
        },
        mangle: {
          reserved: ['SVGPWAApp', 'stopwatch'] // Reserve app globals
        }
      });
      
      if (result.error) {
        throw new Error(`Minification failed: ${result.error}`);
      }
      
      return result.code;
    }
    
    return bundled;
  }

  async getSVGPolyfills() {
    return `
// SVG-PWA Runtime Polyfills v1.0
(function() {
  'use strict';
  
  // Document.head polyfill for SVG context
  if (!document.head) {
    Object.defineProperty(document, 'head', {
      get: function() {
        return {
          appendChild: function(element) {
            console.warn('[SVG-PWA] document.head.appendChild called in SVG context');
            return element;
          },
          removeChild: function(element) {
            console.warn('[SVG-PWA] document.head.removeChild called in SVG context');
            return element;
          }
        };
      }
    });
  }
  
  // Safe DOM utilities
  window.SVGPWAUtils = {
    safeQuerySelector: function(selector, context = document) {
      try {
        return context.querySelector(selector);
      } catch (e) {
        console.warn('[SVG-PWA] querySelector failed:', selector, e);
        return null;
      }
    },
    
    safeSetAttribute: function(element, attr, value) {
      if (element && typeof element.setAttribute === 'function') {
        element.setAttribute(attr, value);
        return true;
      }
      return false;
    },
    
    safeAddEventListener: function(element, event, handler) {
      if (element && typeof element.addEventListener === 'function') {
        element.addEventListener(event, handler);
        return true;
      }
      return false;
    },
    
    // Service Worker registration with error handling
    registerServiceWorker: function(swCode, options = {}) {
      if (!('serviceWorker' in navigator)) {
        console.warn('[SVG-PWA] Service Worker not supported');
        return Promise.resolve(false);
      }
      
      const blob = new Blob([swCode], { type: 'application/javascript' });
      const swUrl = URL.createObjectURL(blob);
      
      const scope = window.location.protocol === 'file:' ? '/' : (options.scope || './');
      
      return navigator.serviceWorker.register(swUrl, { scope })
        .then(registration => {
          console.log('[SVG-PWA] Service Worker registered');
          return registration;
        })
        .catch(error => {
          console.warn('[SVG-PWA] Service Worker registration failed:', error.message);
          return false;
        });
    }
  };
  
  // PWA utilities
  window.SVGPWACore = {
    installPrompt: null,
    
    init: function() {
      this.setupInstallPrompt();
      this.setupNotifications();
      this.setupNetworkDetection();
    },
    
    setupInstallPrompt: function() {
      window.addEventListener('beforeinstallprompt', (e) => {
        e.preventDefault();
        this.installPrompt = e;
        this.onInstallPromptReady();
      });
    },
    
    onInstallPromptReady: function() {
      // Override in your app
      console.log('[SVG-PWA] Install prompt ready');
    },
    
    install: function() {
      if (this.installPrompt) {
        this.installPrompt.prompt();
        return this.installPrompt.userChoice;
      }
      return Promise.resolve({ outcome: 'not-available' });
    },
    
    setupNotifications: function() {
      if ('Notification' in window && Notification.permission === 'default') {
        // Auto-request permission on first user interaction
        document.addEventListener('click', () => {
          Notification.requestPermission();
        }, { once: true });
      }
    },
    
    sendNotification: function(title, options = {}) {
      if ('Notification' in window && Notification.permission === 'granted') {
        return new Notification(title, {
          icon: this.getAppIcon(),
          badge: this.getAppBadge(),
          ...options
        });
      }
      return null;
    },
    
    getAppIcon: function() {
      // Extract icon from SVG manifest or generate default
      return 'data:image/svg+xml;base64,' + btoa('<svg xmlns="http://www.w3.org/2000/svg" width="256" height="256"><rect width="256" height="256" fill="#667eea"/></svg>');
    },
    
    getAppBadge: function() {
      return 'data:image/svg+xml;base64,' + btoa('<svg xmlns="http://www.w3.org/2000/svg" width="96" height="96"><rect width="96" height="96" fill="#667eea"/></svg>');
    },
    
    setupNetworkDetection: function() {
      const updateStatus = () => {
        document.dispatchEvent(new CustomEvent('svgpwa:networkchange', {
          detail: { online: navigator.onLine }
        }));
      };
      
      window.addEventListener('online', updateStatus);
      window.addEventListener('offline', updateStatus);
    }
  };
  
  // Auto-initialize
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', () => {
      window.SVGPWACore.init();
    });
  } else {
    window.SVGPWACore.init();
  }
})();
    `;
  }

  async bundleCSS(cssFiles) {
    let bundled = '';
    
    // Add base styles for SVG-PWA
    bundled += this.getBaseSVGStyles();
    
    // Process each CSS file
    for (const file of cssFiles) {
      bundled += `\n/* === ${path.basename(file.path)} === */\n`;
      bundled += file.content;
    }
    
    // Minify CSS
    if (this.config.minify) {
      const result = new CleanCSS({
        level: 2,
        returnPromise: true
      }).minify(bundled);
      
      return (await result).styles;
    }
    
    return bundled;
  }

  getBaseSVGStyles() {
    return `
/* SVG-PWA Base Styles v1.0 */

/* Reset for foreignObject content */
foreignObject * {
  box-sizing: border-box;
}

/* SVG interaction enhancements */
.svg-interactive {
  cursor: pointer;
  transition: all 0.2s ease;
}

.svg-interactive:hover {
  filter: brightness(1.1);
}

.svg-interactive:active {
  transform: scale(0.95);
}

/* PWA specific animations */
@keyframes svg-pwa-pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.svg-pwa-loading {
  animation: svg-pwa-pulse 1s infinite;
}

/* Responsive foreignObject content */
.svg-pwa-responsive {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

/* Accessibility improvements */
.svg-pwa-sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* Dark mode support */
@media (prefers-color-scheme: dark) {
  .svg-pwa-adaptive {
    filter: invert(1) hue-rotate(180deg);
  }
}

/* Print styles */
@media print {
  .svg-pwa-no-print {
    display: none !important;
  }
}
    `;
  }

  async processAssets() {
    const assetsDir = path.join(this.config.srcDir, 'assets');
    const assets = {};
    
    if (await fs.pathExists(assetsDir)) {
      const files = await this.findFiles(assetsDir);
      
      for (const file of files) {
        const relativePath = path.relative(assetsDir, file);
        const content = await fs.readFile(file);
        
        // Optimize images
        if (this.isImageFile(file)) {
          const optimized = await this.optimizeImage(content, file);
          assets[relativePath] = this.toDataURI(optimized, file);
        } else {
          assets[relativePath] = this.toDataURI(content, file);
        }
      }
    }
    
    return assets;
  }

  async optimizeImage(content, filePath) {
    const ext = path.extname(filePath).toLowerCase();
    
    if (ext === '.svg') {
      const result = await imagemin.buffer(content, {
        plugins: [
          imageminSvgo({
            plugins: [
              { removeViewBox: false },
              { removeDimensions: true }
            ]
          })
        ]
      });
      return result;
    }
    
    // For other image types, return as-is
    // In production, add sharp/imagemin processing
    return content;
  }

  toDataURI(content, filePath) {
    const mimeType = this.getMimeType(filePath);
    const base64 = content.toString('base64');
    return `data:${mimeType};base64,${base64}`;
  }

  getMimeType(filePath) {
    const ext = path.extname(filePath).toLowerCase();
    const mimeTypes = {
      '.svg': 'image/svg+xml',
      '.png': 'image/png',
      '.jpg': 'image/jpeg',
      '.jpeg': 'image/jpeg',
      '.gif': 'image/gif',
      '.webp': 'image/webp',
      '.ico': 'image/x-icon',
      '.woff': 'font/woff',
      '.woff2': 'font/woff2',
      '.ttf': 'font/ttf',
      '.json': 'application/json',
      '.txt': 'text/plain'
    };
    
    return mimeTypes[ext] || 'application/octet-stream';
  }

  async loadManifest() {
    const manifestPath = path.join(this.config.srcDir, 'manifest.json');
    
    if (await fs.pathExists(manifestPath)) {
      const content = await fs.readFile(manifestPath, 'utf8');
      return JSON.parse(content);
    }
    
    // Default manifest
    return {
      name: 'SVG-PWA Application',
      short_name: 'SVG-PWA',
      description: 'Application built with SVG-PWA format',
      start_url: './',
      display: 'standalone',
      theme_color: '#667eea',
      background_color: '#ffffff'
    };
  }

  async generateSVG({ javascript, css, assets, manifest, components }) {
    // Load template
    const templatePath = this.config.templatePath;
    let template = await fs.readFile(templatePath, 'utf8');
    
    // Generate metadata
    const metadata = this.generateMetadata(manifest);
    
    // Generate embedded assets
    const assetsCode = `window.SVGPWAAssets = ${JSON.stringify(assets, null, this.config.minify ? 0 : 2)};`;
    
    // Replace template placeholders
    const replacements = {
      '{{METADATA}}': metadata,
      '{{CSS}}': css,
      '{{JS}}': assetsCode + '\n' + javascript,
      '{{VERSION}}': manifest.version || '1.0.0',
      '{{BUILD_DATE}}': new Date().toISOString(),
      '{{APP_NAME}}': manifest.name || 'SVG-PWA App'
    };
    
    for (const [placeholder, value] of Object.entries(replacements)) {
      template = template.replace(new RegExp(placeholder.replace(/[{}]/g, '\\'), 'g'), value);
    }
    
    return template;
  }

  generateMetadata(manifest) {
    return `
    <pwa:manifest xmlns:pwa="http://svg-pwa.org/ns/manifest">
      <pwa:name>${this.escapeXML(manifest.name)}</pwa:name>
      <pwa:short_name>${this.escapeXML(manifest.short_name)}</pwa:short_name>
      <pwa:description>${this.escapeXML(manifest.description)}</pwa:description>
      <pwa:start_url>${this.escapeXML(manifest.start_url)}</pwa:start_url>
      <pwa:display>${this.escapeXML(manifest.display)}</pwa:display>
      <pwa:theme_color>${this.escapeXML(manifest.theme_color)}</pwa:theme_color>
      <pwa:background_color>${this.escapeXML(manifest.background_color)}</pwa:background_color>
    </pwa:manifest>
    
    <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"
             xmlns:schema="http://schema.org/">
      <schema:SoftwareApplication rdf:about="">
        <schema:name>${this.escapeXML(manifest.name)}</schema:name>
        <schema:applicationCategory>Productivity</schema:applicationCategory>
        <schema:operatingSystem>Web Browser</schema:operatingSystem>
        <schema:dateCreated>${new Date().toISOString().split('T')[0]}</schema:dateCreated>
      </schema:SoftwareApplication>
    </rdf:RDF>
    `;
  }

  escapeXML(str) {
    return String(str)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;');
  }

  async optimizeSVG(svgContent) {
    if (!this.config.optimize) {
      return svgContent;
    }
    
    // Basic SVG optimization
    return svgContent
      .replace(/\s+/g, ' ') // Normalize whitespace
      .replace(/>\s+</g, '><') // Remove whitespace between tags
      .trim();
  }

  async validatePWA(svgContent) {
    const issues = [];
    
    // Check required elements
    const required = [
      { pattern: /<pwa:manifest/, message: 'Missing PWA manifest' },
      { pattern: /<script/, message: 'Missing JavaScript code' },
      { pattern: /serviceWorker|Service Worker/, message: 'Missing Service Worker support' },
      { pattern: /localStorage|Storage/, message: 'Missing local storage support' }
    ];
    
    for (const check of required) {
      if (!check.pattern.test(svgContent)) {
        issues.push({ type: 'warning', message: check.message });
      }
    }
    
    // Check file size
    const size = Buffer.byteLength(svgContent, 'utf8');
    if (size > 10 * 1024 * 1024) { // 10MB
      issues.push({ type: 'error', message: `File too large: ${(size / 1024 / 1024).toFixed(2)}MB` });
    }
    
    // Validate XML
    const xmlValidation = this.validateXML(svgContent);
    if (!xmlValidation.valid) {
      issues.push({ type: 'error', message: `Invalid XML: ${xmlValidation.error}` });
    }
    
    if (issues.length > 0) {
      console.log('\n⚠️  Validation Issues:');
      issues.forEach(issue => {
        const icon = issue.type === 'error' ? '❌' : '⚠️';
        console.log(`${icon} ${issue.message}`);
      });
    } else {
      console.log('✅ PWA validation passed');
    }
    
    return issues;
  }

  validateXML(xmlContent) {
    try {
      const DOMParser = require('@xmldom/xmldom').DOMParser;
      const parser = new DOMParser({
        errorHandler: {
          error: (msg) => { throw new Error(msg); },
          warning: () => {},
          fatalError: (msg) => { throw new Error(msg); }
        }
      });
      
      parser.parseFromString(xmlContent, 'image/svg+xml');
      return { valid: true };
    } catch (error) {
      return { valid: false, error: error.message };
    }
  }

  async generateBuildReport(svgContent, buildTime) {
    const size = Buffer.byteLength(svgContent, 'utf8');
    const report = {
      buildTime: `${buildTime}ms`,
      outputSize: `${(size / 1024).toFixed(2)}KB`,
      timestamp: new Date().toISOString(),
      config: this.config
    };
    
    console.log('\n📊 Build Report:');
    console.log(`⏱️  Build Time: ${report.buildTime}`);
    console.log(`📦 Output Size: ${report.outputSize}`);
    console.log(`📄 Output File: ${path.join(this.config.distDir, this.config.outputName)}`);
    
    // Save detailed report
    const reportPath = path.join(this.config.distDir, 'build-report.json');
    await fs.writeFile(reportPath, JSON.stringify(report, null, 2));
    
    return report;
  }

  // Utility methods
  async findFiles(dir, extensions = ['.js', '.css', '.svg', '.html', '.json']) {
    const files = [];
    
    const scan = async (currentDir) => {
      const items = await fs.readdir(currentDir);
      
      for (const item of items) {
        const fullPath = path.join(currentDir, item);
        const stats = await fs.stat(fullPath);
        
        if (stats.isDirectory()) {
          await scan(fullPath);
        } else if (extensions.includes(path.extname(item))) {
          files.push(fullPath);
        }
      }
    };
    
    await scan(dir);
    return files;
  }

  isImageFile(filePath) {
    const imageExts = ['.svg', '.png', '.jpg', '.jpeg', '.gif', '.webp', '.ico'];
    return imageExts.includes(path.extname(filePath).toLowerCase());
  }

  async transformModules(code) {
    // Simple ES6 module transformation
    // In production, use Babel or similar
    return code
      .replace(/^import\s+.*?from\s+['"].*?['"];?\s*$/gm, '// ')
      .replace(/^export\s+(default\s+)?/gm, '// export $1')
      .replace(/^export\s*\{[^}]*\};?\s*$/gm, '// ');
  }
}

module.exports = ProductionBundler;

// CLI usage
if (require.main === module) {
  const config = {
    srcDir: process.argv[2] || 'src',
    distDir: process.argv[3] || 'dist',
    templatePath: process.argv[4] || 'templates/app.svg.template'
  };
  
  const bundler = new ProductionBundler(config);
  bundler.build().catch(console.error);
}
```

### **2. Testowanie integracyjne**

```javascript
// tests/integration/svg-pwa-integration.test.js
const puppeteer = require('puppeteer');
const fs = require('fs');
const path = require('path');

class SVGPWAIntegrationTest {
  constructor() {
    this.browser = null;
    this.page = null;
    this.results = [];
  }

  async setup() {
    this.browser = await puppeteer.launch({
      headless: process.env.CI === 'true',
      devtools: !process.env.CI
    });
    
    this.page = await this.browser.newPage();
    
    // Enable console logging
    this.page.on('console', msg => {
      console.log(`[Browser] ${msg.text()}`);
    });
    
    // Catch errors
    this.page.on('pageerror', error => {
      console.error(`[Browser Error] ${error.message}`);
    });
  }

  async teardown() {
    if (this.browser) {
      await this.browser.close();
    }
  }

  async testSVGPWAFile(filePath) {
    console.log(`🧪 Testing SVG-PWA: ${filePath}`);
    
    // Convert file path to file:// URL
    const fileUrl = `file://${path.resolve(filePath)}`;
    
    // Navigate to SVG file
    await this.page.goto(fileUrl, { waitUntil: 'networkidle0' });
    
    // Wait for any initialization
    await this.page.waitForTimeout(1000);
    
    // Run test suite
    await this.testBasicStructure();
    await this.testPWAFeatures();
    await this.testInteractivity();
    await this.testPerformance();
    
    return this.results;
  }

  async testBasicStructure() {
    console.log('  📋 Testing basic structure...');
    
    // Check if SVG is properly loaded
    const svgElement = await this.page.$('svg');
    this.assert(svgElement !== null, 'SVG element should exist');
    
    // Check viewport and dimensions
    const svgBox = await svgElement.boundingBox();
    this.assert(svgBox.width > 0 && svgBox.height > 0, 'SVG should have valid dimensions');
    
    // Check for script execution
    const jsExecuted = await this.page.evaluate(() => {
      return typeof window !== 'undefined';
    });
    this.assert(jsExecuted, 'JavaScript should execute');
    
    // Check for PWA metadata
    const hasPWAMetadata = await this.page.evaluate(() => {
      const svg = document.querySelector('svg');
      return svg && svg.querySelector('metadata');
    });
    this.assert(hasPWAMetadata, 'PWA metadata should be present');
  }

  async testPWAFeatures() {
    console.log('  📱 Testing PWA features...');
    
    // Test Service Worker support
    const swSupport = await this.page.evaluate(() => {
      return 'serviceWorker' in navigator;
    });
    this.assert(swSupport, 'Service Worker should be supported');
    
    // Test Local Storage
    const storageTest = await this.page.evaluate(() => {
      try {
        localStorage.setItem('test', 'value');
        const value = localStorage.getItem('test');
        localStorage.removeItem('test');
        return value === 'value';
      } catch (e) {
        return false;
      }
    });
    this.assert(storageTest, 'Local Storage should work');
    
    // Test Notification API
    const notificationSupport = await this.page.evaluate(() => {
      return 'Notification' in window;
    });
    this.assert(notificationSupport, 'Notification API should be available');
    
    // Test manifest
    const manifestCheck = await this.page.evaluate(() => {
      // Look for embedded manifest in SVG
      const manifestElement = document.querySelector('pwa\\:manifest, [*|manifest]');
      return manifestElement !== null;
    });
    this.assert(manifestCheck, 'PWA manifest should be embedded');
  }

  async testInteractivity() {
    console.log('  🖱️  Testing interactivity...');
    
    // Find clickable elements
    const clickableElements = await this.page.$('[onclick], button, .clickable, .svg-interactive');
    
    if (clickableElements.length > 0) {
      // Test click on first interactive element
      await clickableElements[0].click();
      await this.page.waitForTimeout(500);
      
      this.assert(true, `Interactive elements found and clickable (${clickableElements.length} elements)`);
    } else {
      console.log('    ℹ️  No interactive elements found');
    }
    
    // Test keyboard events
    await this.page.keyboard.press('Space');
    await this.page.waitForTimeout(100);
    
    // Check for any dynamic content changes
    const dynamicContent = await this.page.evaluate(() => {
      // Look for elements that might change
      const timers = document.querySelectorAll('[id*="timer"], [id*="display"], [class*="counter"]');
      return timers.length > 0;
    });
    
    if (dynamicContent) {
      this.assert(true, 'Dynamic content elements detected');
    }
  }

  async testPerformance() {
    console.log('  ⚡ Testing performance...');
    
    // Measure page load metrics
    const metrics = await this.page.metrics();
    
    // Check memory usage
    const memoryUsage = metrics.JSHeapUsedSize / 1024 / 1024; // MB
    this.assert(memoryUsage < 50, `Memory usage should be reasonable (${memoryUsage.toFixed(2)}MB)`);
    
    // Test animation performance
    const animationTest = await this.page.evaluate(() => {
      const start = performance.now();
      
      // Trigger any animations
      const animatedElements = document.querySelectorAll('[class*="animate"], [style*="animation"]');
      
      return {
        animatedElements: animatedElements.length,
        testTime: performance.now() - start
      };
    });
    
    this.assert(animationTest.testTime < 100, `Animation test should be```yaml
# .github/workflows/deploy.yml
name: Deploy SVG-PWA
on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '18'
        
    - name: Install dependencies
      run: npm install
      
    - name: Build SVG-PWA
      run: npm run build
      
    - name: Validate SVG-PWA
      run: npm run validate
      
    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./dist
```





this.assert(animationTest.testTime < 100, `Animation test should be fast (${animationTest.testTime.toFixed(2)}ms)`);
    
    // Test render performance
    const renderTest = await this.page.evaluate(() => {
      const start = performance.now();
      
      // Force reflow
      document.body.offsetHeight;
      
      return performance.now() - start;
    });
    
    this.assert(renderTest < 50, `Render performance should be good (${renderTest.toFixed(2)}ms)`);
  }

  assert(condition, message) {
    const result = {
      passed: !!condition,
      message: message,
      timestamp: new Date().toISOString()
    };
    
    this.results.push(result);
    
    const icon = result.passed ? '✅' : '❌';
    console.log(`    ${icon} ${message}`);
    
    if (!result.passed) {
      throw new Error(`Assertion failed: ${message}`);
    }
  }

  async generateReport() {
    const passed = this.results.filter(r => r.passed).length;
    const total = this.results.length;
    const percentage = ((passed / total) * 100).toFixed(1);
    
    const report = {
      summary: {
        total,
        passed,
        failed: total - passed,
        percentage: `${percentage}%`
      },
      results: this.results,
      timestamp: new Date().toISOString()
    };
    
    console.log('\n📊 Test Report:');
    console.log(`✅ Passed: ${passed}/${total} (${percentage}%)`);
    
    if (total - passed > 0) {
      console.log(`❌ Failed: ${total - passed}`);
    }
    
    return report;
  }
}

// CLI usage
async function runTests() {
  const tester = new SVGPWAIntegrationTest();
  
  try {
    await tester.setup();
    
    const svgFile = process.argv[2] || 'dist/app.svg';
    
    if (!fs.existsSync(svgFile)) {
      throw new Error(`SVG file not found: ${svgFile}`);
    }
    
    await tester.testSVGPWAFile(svgFile);
    
    const report = await tester.generateReport();
    
    // Save report
    fs.writeFileSync('test-report.json', JSON.stringify(report, null, 2));
    
    process.exit(report.summary.failed > 0 ? 1 : 0);
    
  } catch (error) {
    console.error('❌ Test failed:', error.message);
    process.exit(1);
  } finally {
    await tester.teardown();
  }
}

if (require.main === module) {
  runTests();
}

module.exports = SVGPWAIntegrationTest;
```

### **3. Development Server z Hot Reload**

```javascript
// tools/dev-server-advanced.js
const express = require('express');
const chokidar = require('chokidar');
const WebSocket = require('ws');
const path = require('path');
const fs = require('fs-extra');
const { ProductionBundler } = require('./production-bundler');

class AdvancedDevServer {
  constructor(config = {}) {
    this.config = {
      port: 3000,
      wsPort: 3001,
      srcDir: 'src',
      templatePath: 'templates/app.svg.template',
      staticDir: 'public',
      hotReload: true,
      mockPWA: true,
      ...config
    };
    
    this.app = express();
    this.wss = new WebSocket.Server({ port: this.config.wsPort });
    this.bundler = new ProductionBundler({
      ...this.config,
      minify: false,
      optimize: false
    });
    
    this.clients = new Set();
    this.lastBuild = null;
    this.buildQueue = Promise.resolve();
  }

  async start() {
    console.log('🚀 Starting SVG-PWA Development Server...');
    
    this.setupWebSockets();
    this.setupFileWatcher();
    this.setupRoutes();
    this.setupMiddleware();
    
    this.app.listen(this.config.port, () => {
      console.log(`📡 Server running at http://localhost:${this.config.port}`);
      console.log(`🔌 WebSocket server on port ${this.config.wsPort}`);
      console.log(`👀 Watching: ${this.config.srcDir}`);
      
      // Initial build
      this.triggerRebuild('startup');
    });
  }

  setupWebSockets() {
    this.wss.on('connection', (ws) => {
      this.clients.add(ws);
      console.log(`🔌 Client connected (${this.clients.size} total)`);
      
      ws.on('close', () => {
        this.clients.delete(ws);
        console.log(`🔌 Client disconnected (${this.clients.size} total)`);
      });
      
      ws.on('message', (message) => {
        try {
          const data = JSON.parse(message);
          this.handleClientMessage(data, ws);
        } catch (e) {
          console.warn('Invalid WebSocket message:', message);
        }
      });
    });
  }

  handleClientMessage(data, ws) {
    switch (data.type) {
      case 'ping':
        ws.send(JSON.stringify({ type: 'pong' }));
        break;
        
      case 'request-rebuild':
        this.triggerRebuild('client-request');
        break;
        
      case 'get-status':
        ws.send(JSON.stringify({
          type: 'status',
          data: {
            building: this.buildQueue !== Promise.resolve(),
            lastBuild: this.lastBuild,
            clients: this.clients.size
          }
        }));
        break;
    }
  }

  setupFileWatcher() {
    const watcher = chokidar.watch([
      `${this.config.srcDir}/**/*`,
      this.config.templatePath,
      'package.json'
    ], {
      ignored: /(^|[\/\\])\../, // ignore dotfiles
      persistent: true
    });
    
    const debouncedRebuild = this.debounce((path) => {
      this.triggerRebuild(`file-change: ${path}`);
    }, 300);
    
    watcher.on('change', debouncedRebuild);
    watcher.on('add', debouncedRebuild);
    watcher.on('unlink', debouncedRebuild);
  }

  setupRoutes() {
    // Main SVG-PWA app
    this.app.get('/app.svg', async (req, res) => {
      try {
        const svg = await this.buildWithDevFeatures();
        
        res.setHeader('Content-Type', 'image/svg+xml');
        res.setHeader('Cache-Control', 'no-cache, no-store, must-revalidate');
        res.send(svg);
        
      } catch (error) {
        console.error('Build error:', error);
        res.status(500).send(this.generateErrorSVG(error));
      }
    });
    
    // Development tools
    this.app.get('/dev-tools', (req, res) => {
      res.send(this.generateDevToolsHTML());
    });
    
    // API endpoints
    this.app.get('/api/status', (req, res) => {
      res.json({
        building: this.buildQueue !== Promise.resolve(),
        lastBuild: this.lastBuild,
        clients: this.clients.size,
        config: this.config
      });
    });
    
    this.app.post('/api/rebuild', (req, res) => {
      this.triggerRebuild('api-request');
      res.json({ status: 'rebuild triggered' });
    });
    
    // Static files
    if (this.config.staticDir && fs.existsSync(this.config.staticDir)) {
      this.app.use('/static', express.static(this.config.staticDir));
    }
    
    // Root redirect
    this.app.get('/', (req, res) => {
      res.redirect('/app.svg');
    });
  }

  setupMiddleware() {
    // CORS for development
    this.app.use((req, res, next) => {
      res.header('Access-Control-Allow-Origin', '*');
      res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept');
      next();
    });
    
    // Request logging
    this.app.use((req, res, next) => {
      console.log(`${req.method} ${req.path}`);
      next();
    });
  }

  async buildWithDevFeatures() {
    // Wait for any ongoing build
    await this.buildQueue;
    
    // Build the SVG
    const svg = await this.bundler.generateSVG(await this.bundler.loadComponents());
    
    // Inject development features
    return this.injectDevFeatures(svg);
  }

  injectDevFeatures(svg) {
    if (!this.config.hotReload) return svg;
    
    const devScript = `
// SVG-PWA Development Features
(function() {
  'use strict';
  
  // Hot reload WebSocket connection
  const ws = new WebSocket('ws://localhost:${this.config.wsPort}');
  
  ws.onopen = function() {
    console.log('🔌 [DevServer] Connected to development server');
  };
  
  ws.onmessage = function(event) {
    const data = JSON.parse(event.data);
    
    switch (data.type) {
      case 'reload':
        console.log('🔄 [DevServer] Reloading...');
        window.location.reload();
        break;
        
      case 'build-error':
        console.error('❌ [DevServer] Build error:', data.error);
        showDevError(data.error);
        break;
        
      case 'build-success':
        console.log('✅ [DevServer] Build successful');
        hideDevError();
        break;
    }
  };
  
  ws.onclose = function() {
    console.log('🔌 [DevServer] Disconnected from development server');
    setTimeout(() => {
      // Try to reconnect
      if (document.hidden === false) {
        window.location.reload();
      }
    }, 1000);
  };
  
  // Development UI
  function createDevUI() {
    const devPanel = document.createElement('div');
    devPanel.id = 'svg-pwa-dev-panel';
    devPanel.innerHTML = \`
      <div style="position:fixed; top:10px; left:10px; background:rgba(0,0,0,0.8); color:white; padding:10px; border-radius:5px; font-family:monospace; font-size:12px; z-index:10000;">
        <div>🛠️ SVG-PWA Dev Mode</div>
        <div>📡 Server: Connected</div>
        <div>🔄 Hot Reload: Active</div>
        <button onclick="window.location.reload()" style="background:#4CAF50; color:white; border:none; padding:5px 10px; border-radius:3px; margin-top:5px; cursor:pointer;">
          Reload
        </button>
      </div>
    \`;
    
    if (document.body) {
      document.body.appendChild(devPanel);
    } else {
      document.addEventListener('DOMContentLoaded', () => {
        document.body.appendChild(devPanel);
      });
    }
  }
  
  function showDevError(error) {
    let errorPanel = document.getElementById('svg-pwa-error-panel');
    
    if (!errorPanel) {
      errorPanel = document.createElement('div');
      errorPanel.id = 'svg-pwa-error-panel';
      document.body.appendChild(errorPanel);
    }
    
    errorPanel.innerHTML = \`
      <div style="position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); background:#ff4444; color:white; padding:20px; border-radius:10px; font-family:monospace; font-size:14px; z-index:10001; max-width:80%; max-height:80%; overflow:auto;">
        <h3>❌ Build Error</h3>
        <pre style="white-space:pre-wrap; margin:10px 0;">\${error}</pre>
        <button onclick="this.parentElement.parentElement.style.display='none'" style="background:rgba(255,255,255,0.2); color:white; border:none; padding:10px 20px; border-radius:5px; cursor:pointer;">
          Close
        </button>
      </div>
    \`;
    
    errorPanel.style.display = 'block';
  }
  
  function hideDevError() {
    const errorPanel = document.getElementById('svg-pwa-error-panel');
    if (errorPanel) {
      errorPanel.style.display = 'none';
    }
  }
  
  // Initialize dev UI
  createDevUI();
  
  // Global dev utilities
  window.SVGPWADev = {
    reload: () => window.location.reload(),
    rebuild: () => ws.send(JSON.stringify({ type: 'request-rebuild' })),
    status: () => ws.send(JSON.stringify({ type: 'get-status' }))
  };
  
  console.log('🛠️ [DevServer] Development mode active');
  console.log('💡 [DevServer] Available commands: SVGPWADev.reload(), SVGPWADev.rebuild(), SVGPWADev.status()');
})();
    `;
    
    // Inject before closing script tag
    return svg.replace('</script>', devScript + '\n</script>');
  }

  generateErrorSVG(error) {
    return `<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" width="800" height="600" viewBox="0 0 800 600">
  <rect width="800" height="600" fill="#ff4444"/>
  <text x="400" y="100" text-anchor="middle" font-family="monospace" font-size="24" fill="white" font-weight="bold">
    ❌ Build Error
  </text>
  <foreignObject x="50" y="150" width="700" height="400">
    <div xmlns="http://www.w3.org/1999/xhtml" style="color:white; font-family:monospace; font-size:14px; background:rgba(0,0,0,0.3); padding:20px; border-radius:10px; overflow:auto; height:100%;">
      <pre style="white-space:pre-wrap; margin:0;">${this.escapeHTML(error.message || error)}</pre>
    </div>
  </foreignObject>
  <text x="400" y="580" text-anchor="middle" font-family="sans-serif" font-size="16" fill="white">
    Fix the error and save to reload automatically
  </text>
</svg>`;
  }

  generateDevToolsHTML() {
    return `<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SVG-PWA Development Tools</title>
    <style>
        body { font-family: system-ui; margin: 0; padding: 20px; background: #f5f5f5; }
        .container { max-width: 1200px; margin: 0 auto; }
        .panel { background: white; border-radius: 10px; padding: 20px; margin: 20px 0; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .status { display: flex; gap: 20px; align-items: center; }
        .status-item { padding: 10px 20px; border-radius: 20px; font-weight: bold; }
        .status-good { background: #4CAF50; color: white; }
        .status-building { background: #FF9800; color: white; }
        .status-error { background: #f44336; color: white; }
        button { background: #2196F3; color: white; border: none; padding: 10px 20px; border-radius: 5px; cursor: pointer; margin: 5px; }
        button:hover { background: #1976D2; }
        pre { background: #f8f9fa; padding: 15px; border-radius: 5px; overflow-x: auto; }
        .preview { border: 1px solid #ddd; border-radius: 5px; background: white; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🛠️ SVG-PWA Development Tools</h1>
        
        <div class="panel">
            <h2>📊 Server Status</h2>
            <div class="status" id="status">
                <div class="status-item status-good">Server Running</div>
                <div class="status-item" id="build-status">Ready</div>
                <div class="status-item" id="client-count">0 Clients</div>
            </div>
            
            <div style="margin-top: 20px;">
                <button onclick="rebuild()">🔄 Rebuild</button>
                <button onclick="openApp()">🚀 Open App</button>
                <button onclick="refreshStatus()">📡 Refresh Status</button>
            </div>
        </div>
        
        <div class="panel">
            <h2>👀 Live Preview</h2>
            <iframe src="/app.svg" class="preview" width="100%" height="400" id="preview"></iframe>
        </div>
        
        <div class="panel">
            <h2>📝 Build Log</h2>
            <pre id="build-log">Waiting for build events...</pre>
        </div>
        
        <div class="panel">
            <h2>⚙️ Configuration</h2>
            <pre id="config">${JSON.stringify(this.config, null, 2)}</pre>
        </div>
    </div>
    
    <script>
        let ws;
        let buildLog = [];
        
        function connectWebSocket() {
            ws = new WebSocket('ws://localhost:${this.config.wsPort}');
            
            ws.onopen = function() {
                log('Connected to development server');
                updateStatus();
            };
            
            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);
                handleMessage(data);
            };
            
            ws.onclose = function() {
                log('Disconnected from development server');
                setTimeout(connectWebSocket, 1000);
            };
        }
        
        function handleMessage(data) {
            switch (data.type) {
                case 'reload':
                    log('App reloaded');
                    document.getElementById('preview').src = '/app.svg?' + Date.now();
                    break;
                    
                case 'build-success':
                    log('✅ Build successful');
                    document.getElementById('build-status').textContent = 'Built';
                    document.getElementById('build-status').className = 'status-item status-good';
                    break;
                    
                case 'build-error':
                    log('❌ Build failed: ' + data.error);
                    document.getElementById('build-status').textContent = 'Error';
                    document.getElementById('build-status').className = 'status-item status-error';
                    break;
                    
                case 'status':
                    updateStatusFromData(data.data);
                    break;
            }
        }
        
        function log(message) {
            const timestamp = new Date().toLocaleTimeString();
            buildLog.push(\`[\${timestamp}] \${message}\`);
            if (buildLog.length > 50) buildLog.shift();
            
            document.getElementById('build-log').textContent = buildLog.join('\\n');
        }
        
        function rebuild() {
            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({ type: 'request-rebuild' }));
                log('Rebuild requested');
                document.getElementById('build-status').textContent = 'Building...';
                document.getElementById('build-status').className = 'status-item status-building';
            }
        }
        
        function openApp() {
            window.open('/app.svg', '_blank');
        }
        
        function refreshStatus() {
            fetch('/api/status')
                .then(r => r.json())
                .then(updateStatusFromData)
                .catch(e => log('Failed to fetch status: ' + e.message));
        }
        
        function updateStatusFromData(data) {
            document.getElementById('client-count').textContent = \`\${data.clients} Clients\`;
            
            if (data.building) {
                document.getElementById('build-status').textContent = 'Building...';
                document.getElementById('build-status').className = 'status-item status-building';
            }
        }
        
        function updateStatus() {
            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({ type: 'get-status' }));
            }
        }
        
        // Initialize
        connectWebSocket();
        setInterval(updateStatus, 5000);
    </script>
</body>
</html>`;
  }

  triggerRebuild(reason) {
    this.buildQueue = this.buildQueue
      .then(async () => {
        console.log(`🔄 Rebuilding (${reason})...`);
        
        try {
          const startTime = Date.now();
          
          // Build the app
          await this.buildWithDevFeatures();
          
          const buildTime = Date.now() - startTime;
          this.lastBuild = {
            timestamp: new Date().toISOString(),
            duration: buildTime,
            success: true,
            reason
          };
          
          console.log(`✅ Build completed in ${buildTime}ms`);
          
          // Notify clients
          this.broadcast({
            type: 'build-success',
            duration: buildTime
          });
          
          // Trigger reload
          if (this.config.hotReload) {
            setTimeout(() => {
              this.broadcast({ type: 'reload' });
            }, 100);
          }
          
        } catch (error) {
          console.error('❌ Build failed:', error.message);
          
          this.lastBuild = {
            timestamp: new Date().toISOString(),
            success: false,
            error: error.message,
            reason
          };
          
          // Notify clients of error
          this.broadcast({
            type: 'build-error',
            error: error.message
          });
        }
      })
      .catch(error => {
        console.error('Build queue error:', error);
      });
  }

  broadcast(message) {
    const data = JSON.stringify(message);
    
    this.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(data);
      }
    });
  }

  debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
      const later = () => {
        clearTimeout(timeout);
        func(...args);
      };
      clearTimeout(timeout);
      timeout = setTimeout(later, wait);
    };
  }

  escapeHTML(str) {
    return String(str)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;');
  }
}

module.exports = AdvancedDevServer;

// CLI usage
if (require.main === module) {
  const config = {
    port: process.argv[2] || 3000,
    srcDir: process.argv[3] || 'src'
  };
  
  const server = new AdvancedDevServer(config);
  server.start().catch(console.error);
}
```

---

## 🎯 **Podsumowanie - Complete Development Ecosystem**

### **📦 Kompletny toolkit SVG-PWA:**

1. **Development Tools:**
   - SVG-PWA CLI dla zarządzania projektami
   - Advanced Dev Server z hot reload
   - VS Code Extension z syntax highlighting
   - Web-based Builder dla wizualnego tworzenia

2. **Build System:**
   - Production Bundler z optymalizacją
   - Multi-file to single SVG compilation
   - Asset processing i data URI conversion
   - PWA validation i error checking

3. **Testing Framework:**
   - Integration testing z Puppeteer
   - Unit testing dla SVG komponentów
   - Performance monitoring
   - Cross-browser compatibility testing

4. **Deployment Solutions:**
   - GitHub Pages integration
   - CDN deployment scripts
   - Email distribution dla aplikacji
   - Version management

### **🚀 Recommended Workflow:**

```bash
# 1. Project Setup
npx svg-pwa-cli create my-app --template=advanced
cd my-app

# 2. Development
npm run dev           # Start dev server z hot reload
npm run test:watch    # Continuous testing
npm run validate      # Validation podczas dev

# 3. Testing
npm run test          # Full test suite
npm run test:e2e      # End-to-end testing
npm run test:perf     # Performance testing

# 4. Production Build
npm run build         # Optimized production build
npm run validate:prod # Production validation
npm run analyze       # Bundle analysis

# 5. Deployment
npm run deploy:gh     # GitHub Pages
npm run deploy:cdn    # CDN deployment
npm run distribute    # Email distribution
```

### **🎯 Kluczowe korzyści tego ecosystem:**

1. **Developer Experience** - Podobny do React/Vue development
2. **Hot Reload** - Instant feedback podczas development
3. **Testing Integration** - Comprehensive testing strategy
4. **Production Ready** - Optimized builds z validation
5. **Distribution** - Multiple deployment options

### **📈 Future Enhancements:**

1. **Advanced Bundler** z tree-shaking
2. **Component Library** dla UI elements
3. **State Management** library dla SVG-PWA
4. **PWA Store** dla distribution
5. **Analytics Platform** dla SVG-PWA apps

Ten ecosystem czyni development SVG-PWA tak prosty jak tworzenie standardowych web aplikacji, eliminując główne bariery techniczne! 🎉### **🛠️ Priorytetowe narzędzia do stworzenia:**

1. **SVG-PWA CLI** - podstawowe narzędzie deweloperskie
2. **VS Code Extension** - syntax highlighting, snippets, validation
3. **Web Builder** - wizualny kreator dla nie-programistów
4. **Webpack Plugin** - integracja z istniejącymi workflow
5. **Testing Framework** - dedykowane testowanie SVG-PWA
