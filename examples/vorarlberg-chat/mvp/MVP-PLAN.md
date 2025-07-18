# 📋 MVP Implementierungsplan - Vorarlberg Chat

## 🎯 MVP-Ziele

Ein funktionsfähiger Chatbot für Vorarlberg-Informationen, der:
- Vollständig im Browser läuft (Edge AI)
- Auf render.com gehostet wird
- Browser-Cache für Persistierung nutzt
- Ohne komplexe Backend-Infrastruktur auskommt

## 🏗️ Technische Architektur

### Core Stack
- **Frontend**: React 18 + TypeScript + Vite
- **AI Engine**: WebLLM (Browser-GPU)
- **Styling**: CSS Modules + vol.at Design
- **Daten**: Statische JSON-Dateien
- **Cache**: Browser LocalStorage/IndexedDB
- **Hosting**: render.com

### Datenfluss
```
Benutzer Input → Intent Detection → Statische Daten → WebLLM → Response → Browser Cache
```

## 📦 Implementierungsschritte

### Phase 1: Setup (Tag 1-2)
- [ ] Vite + React + TypeScript Setup
- [ ] render.yaml Konfiguration
- [ ] Basis-Komponenten erstellen
- [ ] vol.at Design-System implementieren

### Phase 2: Core Features (Tag 3-5)
- [ ] WebLLM Integration
- [ ] Chat-Interface
- [ ] Browser Cache Service
- [ ] Statische Vorarlberg-Daten erstellen

### Phase 3: Enhancement (Tag 6-7)
- [ ] Intent Detection (vereinfacht)
- [ ] Response Optimization
- [ ] Mobile Responsiveness
- [ ] Performance Tuning

### Phase 4: Deployment (Tag 8)
- [ ] render.com Setup
- [ ] Environment Variables
- [ ] Production Build
- [ ] Testing & Launch

## 🗂️ Datenstruktur

### Statische Daten (JSON)
```typescript
interface VorarlbergData {
  locations: Location[]
  restaurants: Restaurant[]
  events: Event[]
  faqs: FAQ[]
  general: GeneralInfo[]
}

interface Location {
  id: string
  name: string
  description: string
  category: string
  city: string
  coordinates?: [number, number]
  tags: string[]
}
```

### Browser Cache Schema
```typescript
interface ChatHistory {
  id: string
  messages: Message[]
  timestamp: Date
  lastActive: Date
}

interface Message {
  id: string
  role: 'user' | 'assistant'
  content: string
  timestamp: Date
  sources?: string[]
}
```

## 🎨 UI/UX Design

### vol.at Design-Inspiration
```css
:root {
  --primary-color: #FFD200;    /* vol.at Gelb */
  --secondary-color: #1A1603;  /* Schwarz */
  --background: #F7F6F0;       /* Hellgrau */
  --accent: #E3051B;           /* Rot */
}
```

### Komponenten-Hierarchie
```
App
├── Header (Logo + Status)
├── ChatContainer
│   ├── MessageList
│   │   └── Message (User/Assistant)
│   ├── QuickActions (Suggestion Cards)
│   └── LoadingIndicator
├── InputArea
│   ├── TextInput
│   └── SendButton
└── Footer (powered by WebLLM)
```

## 🔧 Technische Details

### WebLLM Konfiguration
```typescript
const modelConfig = {
  model: "Llama-3.2-1B-Instruct-q4f32_1-MLC",
  temperature: 0.7,
  max_tokens: 512,
  system_prompt: `Du bist ein hilfreicher Assistent für Vorarlberg-Informationen.
  Antworte auf Deutsch und sei präzise und freundlich.`
}
```

### Browser Cache Service
```typescript
class CacheService {
  private storage = window.localStorage
  
  saveChatHistory(history: ChatHistory): void
  loadChatHistory(): ChatHistory[]
  clearOldHistory(days: number): void
  saveUserPreferences(prefs: UserPreferences): void
}
```

### Intent Detection (Vereinfacht)
```typescript
const intentKeywords = {
  'restaurant': ['essen', 'restaurant', 'gasthaus', 'küche'],
  'location': ['sehenswürdigkeit', 'museum', 'ort', 'besichtigen'],
  'event': ['veranstaltung', 'event', 'konzert', 'festival'],
  'hiking': ['wandern', 'berg', 'tour', 'route']
}
```

## 🚀 render.com Konfiguration

### render.yaml
```yaml
services:
  - type: web
    name: vorarlberg-chat
    runtime: static
    buildCommand: npm run build
    staticPublishPath: ./dist
    envVars:
      - key: NODE_ENV
        value: production
      - key: VITE_APP_VERSION
        value: mvp-1.0
    headers:
      - path: /*
        name: Cache-Control
        value: public, max-age=3600
```

### Environment Variables
```bash
# Render.com Dashboard
VITE_APP_NAME=Vorarlberg Chat
VITE_APP_VERSION=1.0.0
VITE_MODEL_NAME=Llama-3.2-1B-Instruct-q4f32_1-MLC
```

## 📊 Performance Ziele

### Metriken
- **First Contentful Paint**: < 2s
- **Model Load Time**: < 30s
- **Response Time**: < 3s
- **Bundle Size**: < 2MB (ohne Model)
- **Memory Usage**: < 2GB (mit Model)

### Optimierungen
- Code Splitting für WebLLM
- Lazy Loading für Komponenten
- Aggressive Caching
- Minimal Bundle Size

## 🗃️ Datensammlung

### Vorarlberg-Basisdaten
```json
{
  "locations": [
    {
      "id": "schattenburg",
      "name": "Schattenburg",
      "description": "Historische Burg in Feldkirch",
      "category": "museum",
      "city": "Feldkirch",
      "tags": ["burg", "museum", "geschichte"]
    }
  ],
  "restaurants": [
    {
      "id": "gasthaus-krone",
      "name": "Gasthaus Krone",
      "description": "Traditionelle Vorarlberger Küche",
      "category": "gasthaus",
      "city": "Bregenz",
      "tags": ["traditionell", "vorarlberger küche"]
    }
  ]
}
```

### FAQ-Daten
```json
{
  "faqs": [
    {
      "question": "Was kann man in Vorarlberg unternehmen?",
      "answer": "Vorarlberg bietet viele Aktivitäten: Wandern, Skifahren, kulturelle Sehenswürdigkeiten und kulinarische Erlebnisse.",
      "tags": ["aktivitäten", "tourismus"]
    }
  ]
}
```

## 🔄 Workflow

### Development
```bash
# Lokale Entwicklung
npm run dev

# Build für Production
npm run build

# Preview Build
npm run preview
```

### Deployment
```bash
# Automatisch bei Git Push
git push origin main

# Render.com deployed automatisch
# URL: https://vorarlberg-chat.onrender.com
```

## 🎯 Erfolgsmetriken

### Technische KPIs
- Model lädt erfolgreich: 100%
- Chat funktioniert: 100%
- Cache persistiert: 100%
- Mobile responsive: 100%

### User Experience
- Erste Antwort: < 5s
- Folgende Antworten: < 3s
- Intuitive Bedienung: Ja
- Fehlerbehandlung: Robust

## 🚨 Risiken & Mitigations

### Technische Risiken
| Risiko | Mitigation |
|--------|------------|
| WebLLM lädt nicht | Fallback-Nachrichten |
| Browser-Kompatibilität | Progressive Enhancement |
| Performance-Probleme | Lazy Loading |

### Deployment-Risiken
| Risiko | Mitigation |
|--------|------------|
| render.com Limits | Optimierte Bundle-Größe |
| Build-Fehler | Umfassende Tests |
| Environment-Probleme | Dokumentierte Konfiguration |

## 📝 Nächste Schritte

1. **Sofort**: Vite-Projekt erstellen
2. **Diese Woche**: Core-Features implementieren
3. **Nächste Woche**: render.com Deployment
4. **Danach**: User-Testing und Feedback

---
*MVP-Plan Version 1.0 - Fokus auf schnelle Umsetzung*