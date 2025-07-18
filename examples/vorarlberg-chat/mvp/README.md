# 🚀 Vorarlberg Chat MVP

## Überblick

Dieses MVP (Minimum Viable Product) implementiert einen funktionsfähigen Chatbot für Vorarlberg mit Edge AI, der vollständig im Browser läuft und render.com als Hosting-Plattform nutzt.

## 🎯 MVP-Fokus

### Was ist DRIN (Core Features)
- ✅ **Chat-Interface** - Einfache, responsive UI
- ✅ **Edge AI** - WebLLM mit Llama-3.2-1B im Browser
- ✅ **Browser-Cache** - Lokale Speicherung von Chatverläufen
- ✅ **Statische Daten** - Vorarlberg-Infos als JSON-Dateien
- ✅ **Render.com Ready** - Optimiert für render.com Deployment

### Was ist NICHT DRIN (für spätere Versionen)
- ❌ Supabase/PostgreSQL Datenbank
- ❌ Benutzerauthentifizierung
- ❌ Komplexe API-Integrationen
- ❌ Spracherkennung
- ❌ Mehrsprachigkeit
- ❌ Erweiterte Sicherheitsfeatures

## 🏗️ Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser                               │
├─────────────────────────────────────────────────────────────┤
│  React App                                                  │
│  ├── Chat UI                                                │
│  ├── WebLLM Engine (Edge AI)                               │
│  ├── Browser Cache (Chat History)                          │
│  └── Static Data (Vorarlberg JSON)                         │
├─────────────────────────────────────────────────────────────┤
│                     Render.com                             │
│  ├── Static File Serving                                   │
│  ├── Environment Variables                                 │
│  └── Auto-Deploy from Git                                  │
└─────────────────────────────────────────────────────────────┘
```

## 📁 Projektstruktur

```
mvp/
├── src/
│   ├── components/        # React-Komponenten
│   ├── services/         # WebLLM, Cache, Data
│   ├── data/             # Statische Vorarlberg-Daten
│   ├── styles/           # CSS/SCSS
│   └── utils/            # Hilfsfunktionen
├── public/
│   ├── index.html
│   └── favicon.ico
├── package.json
├── vite.config.js
├── render.yaml           # Render.com Konfiguration
├── .env.example
└── README.md
```

## 🚀 Quick Start

1. **Projekt klonen**
   ```bash
   git clone <repository>
   cd vorarlberg-chat/mvp
   ```

2. **Dependencies installieren**
   ```bash
   npm install
   ```

3. **Development starten**
   ```bash
   npm run dev
   ```

4. **Auf render.com deployen**
   - GitHub Repository verknüpfen
   - render.yaml wird automatisch erkannt
   - Environment Variables setzen

## 🎲 Nächste Schritte

1. Implementiere React-Komponenten
2. Integriere WebLLM
3. Implementiere Browser-Cache
4. Erstelle Vorarlberg-Daten
5. Deploy auf render.com

---
*MVP Version 1.0 - Fokus auf Funktionalität*