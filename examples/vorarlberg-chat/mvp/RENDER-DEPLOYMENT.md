# 🚀 Render.com Deployment Guide

## render.yaml Konfiguration

Erstelle eine `render.yaml` Datei im Root-Verzeichnis mit folgendem Inhalt:

```yaml
# render.yaml - Konfiguration für render.com Deployment

services:
  - type: web
    name: vorarlberg-chat-mvp
    runtime: static
    buildCommand: npm ci && npm run build
    staticPublishPath: ./dist
    
    # Environment Variables (über Dashboard setzen)
    envVars:
      - key: NODE_ENV
        value: production
      - key: VITE_APP_NAME
        value: Vorarlberg Chat
      - key: VITE_APP_VERSION
        value: "1.0.0"
      - key: VITE_MODEL_NAME
        value: Llama-3.2-1B-Instruct-q4f32_1-MLC
      - key: VITE_CACHE_VERSION
        value: "1.0"
      - key: VITE_DEBUG
        value: "false"
    
    # Custom Headers für WebLLM/WebGPU Support
    headers:
      - path: /*
        name: Cache-Control
        value: public, max-age=3600
      - path: /assets/*
        name: Cache-Control
        value: public, max-age=31536000
      - path: "*.js"
        name: Cache-Control
        value: public, max-age=31536000
      - path: "*.css"
        name: Cache-Control
        value: public, max-age=31536000
      - path: "*.wasm"
        name: Cache-Control
        value: public, max-age=31536000
      # Wichtig für WebLLM/SharedArrayBuffer Support
      - path: /*
        name: Cross-Origin-Embedder-Policy
        value: require-corp
      - path: /*
        name: Cross-Origin-Opener-Policy
        value: same-origin
    
    # SPA Routing
    routes:
      - type: redirect
        source: /*
        destination: /index.html
```

## package.json Konfiguration

```json
{
  "name": "vorarlberg-chat-mvp",
  "version": "1.0.0",
  "description": "MVP für Vorarlberg Chat mit Edge AI",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "type-check": "tsc --noEmit",
    "clean": "rm -rf dist node_modules/.vite"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@mlc-ai/web-llm": "^0.2.46"
  },
  "devDependencies": {
    "@types/react": "^18.2.43",
    "@types/react-dom": "^18.2.17",
    "@typescript-eslint/eslint-plugin": "^6.14.0",
    "@typescript-eslint/parser": "^6.14.0",
    "@vitejs/plugin-react": "^4.2.1",
    "eslint": "^8.55.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.4.5",
    "typescript": "^5.2.2",
    "vite": "^5.0.8"
  },
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  },
  "browserslist": [
    "Chrome >= 90",
    "Firefox >= 90",
    "Safari >= 14",
    "Edge >= 90"
  ]
}
```

## vite.config.ts Konfiguration

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { resolve } from 'path'

export default defineConfig({
  plugins: [react()],
  
  base: './',
  
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
      '@components': resolve(__dirname, './src/components'),
      '@services': resolve(__dirname, './src/services'),
      '@utils': resolve(__dirname, './src/utils'),
      '@data': resolve(__dirname, './src/data'),
      '@styles': resolve(__dirname, './src/styles'),
      '@types': resolve(__dirname, './src/types')
    }
  },
  
  server: {
    port: 3000,
    host: true,
    open: true,
    // Wichtig für WebLLM/WebGPU
    headers: {
      'Cross-Origin-Embedder-Policy': 'require-corp',
      'Cross-Origin-Opener-Policy': 'same-origin'
    }
  },
  
  build: {
    outDir: 'dist',
    assetsDir: 'assets',
    sourcemap: false,
    target: 'es2022',
    
    rollupOptions: {
      output: {
        manualChunks: {
          'webllm': ['@mlc-ai/web-llm'],
          'react': ['react', 'react-dom'],
          'vendor': ['@mlc-ai/web-llm']
        }
      }
    },
    
    chunkSizeWarningLimit: 10000
  },
  
  optimizeDeps: {
    include: ['react', 'react-dom'],
    exclude: ['@mlc-ai/web-llm']
  },
  
  worker: {
    format: 'es'
  }
})
```

## Environment Variables

### .env.example
```bash
# Vorarlberg Chat MVP - Environment Variables

# App-Informationen
VITE_APP_NAME=Vorarlberg Chat
VITE_APP_VERSION=1.0.0
VITE_APP_DESCRIPTION=Dein KI-Assistent für Vorarlberg

# WebLLM-Konfiguration
VITE_MODEL_NAME=Llama-3.2-1B-Instruct-q4f32_1-MLC
VITE_MODEL_TEMPERATURE=0.7
VITE_MODEL_MAX_TOKENS=512

# Cache-Konfiguration
VITE_CACHE_VERSION=1.0
VITE_CACHE_MAX_AGE=86400
VITE_CACHE_MAX_ENTRIES=1000

# Debug-Einstellungen
VITE_DEBUG=false
VITE_LOG_LEVEL=warn

# UI-Konfiguration
VITE_THEME=vol-at
VITE_LANGUAGE=de
VITE_MAX_MESSAGES=100
```

## Deployment-Schritte

### 1. Repository Setup
```bash
# Git Repository erstellen
git init
git add .
git commit -m "Initial MVP setup"
git branch -M main
git remote add origin https://github.com/username/vorarlberg-chat.git
git push -u origin main
```

### 2. Render.com Setup
1. **Account erstellen**: Auf [render.com](https://render.com) registrieren
2. **Repository verbinden**: GitHub Repository auswählen
3. **Service konfigurieren**:
   - Service Type: `Static Site`
   - Build Command: `npm ci && npm run build`
   - Publish Directory: `dist`
   - Auto-Deploy: `Yes`

### 3. Environment Variables setzen
Im Render.com Dashboard unter "Environment":
```
NODE_ENV=production
VITE_APP_NAME=Vorarlberg Chat
VITE_APP_VERSION=1.0.0
VITE_MODEL_NAME=Llama-3.2-1B-Instruct-q4f32_1-MLC
VITE_CACHE_VERSION=1.0
VITE_DEBUG=false
```

### 4. Custom Domain (Optional)
- Im Dashboard unter "Settings"
- Custom Domain hinzufügen
- SSL wird automatisch bereitgestellt

## Wichtige Besonderheiten für render.com

### Dynamische Ports
- Render.com setzt automatisch `PORT` Environment Variable
- Für Static Sites nicht relevant, da nur statische Dateien served werden
- Ports sind automatisch dynamisch und werden von render.com verwaltet

### WebLLM/WebGPU Support
- **COOP/COEP Headers** sind essentiell für SharedArrayBuffer
- Diese werden über die `headers` Sektion in render.yaml gesetzt
- Ohne diese Headers funktioniert WebLLM nicht

### Build-Optimierungen
- **Chunk Splitting** für bessere Performance
- **Aggressive Caching** für statische Assets
- **Bundle Size Limits** erhöht für WebLLM

## Monitoring & Debugging

### Render.com Dashboard
- **Deployment Logs**: Zeigt Build-Prozess
- **Service Logs**: Runtime-Informationen
- **Metrics**: Performance-Daten

### Browser-Debugging
```javascript
// Console-Debugging für WebLLM
console.log('WebLLM Status:', window.webllm_status)
console.log('Cache Status:', localStorage.getItem('vorarlberg_chat_cache'))
```

## Troubleshooting

### Häufige Probleme

1. **Build-Fehler**
   - TypeScript-Fehler beheben
   - Dependencies prüfen
   - Build-Command anpassen

2. **WebLLM lädt nicht**
   - COOP/COEP Headers prüfen
   - Browser-Kompatibilität prüfen
   - Netzwerk-Debugging

3. **Cache-Probleme**
   - Cache-Version erhöhen
   - LocalStorage leeren
   - Service Worker prüfen

### Debug-Commands
```bash
# Lokaler Build-Test
npm run build
npm run preview

# Dependency-Check
npm ls --depth=0

# TypeScript-Check
npm run type-check
```

## Performance-Monitoring

### Metriken überwachen
- **Bundle Size**: < 2MB (ohne WebLLM)
- **Load Time**: < 30s für Model
- **Response Time**: < 3s für Chat
- **Memory Usage**: < 2GB total

### Optimierungen
- **Lazy Loading** für WebLLM
- **Code Splitting** für bessere Performance
- **Aggressive Caching** für statische Assets

---
*Render.com Deployment Guide - Version 1.0*