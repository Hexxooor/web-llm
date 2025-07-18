# 🛠️ Implementierungsanleitung: Ländle Chatbot

## Voraussetzungen

- Node.js 18+ und npm/yarn
- Supabase Account
- Google Cloud Account (für Search API)
- Git

## 📦 Phase 1: Projekt-Setup

### 1.1 Repository klonen und Dependencies
```bash
# Repository klonen
git clone [repository-url]
cd vorarlberg-chat

# Dependencies installieren
npm install

# Zusätzliche Packages
npm install @supabase/supabase-js
npm install axios
npm install react-speech-kit
npm install @types/node --save-dev
```

### 1.2 Environment Variables
```bash
# .env.local erstellen
cat > .env.local << EOF
# Supabase
VITE_SUPABASE_URL=https://[PROJECT_ID].supabase.co
VITE_SUPABASE_ANON_KEY=your_anon_key

# Google Search API
VITE_GOOGLE_API_KEY=your_google_api_key
VITE_GOOGLE_SEARCH_ENGINE_ID=your_engine_id

# Optional: OpenWeather
VITE_WEATHER_API_KEY=your_weather_key
EOF
```

### 1.3 TypeScript Konfiguration
```typescript
// tsconfig.json anpassen
{
  "compilerOptions": {
    "baseUrl": "./src",
    "paths": {
      "@/*": ["*"],
      "@components/*": ["components/*"],
      "@services/*": ["services/*"],
      "@types/*": ["types/*"]
    }
  }
}
```

## 🗄️ Phase 2: Supabase Setup

### 2.1 Supabase Projekt erstellen
1. Auf [supabase.com](https://supabase.com) einloggen
2. Neues Projekt "vorarlberg-chat" erstellen
3. Region: Frankfurt (eu-central-1)
4. Passwort sicher speichern

### 2.2 Datenbank initialisieren
```sql
-- In Supabase SQL Editor ausführen
-- 1. Extensions aktivieren
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "unaccent"; 
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- 2. Schema aus 03-datenbank-schema.md ausführen
-- (Tabellen in der richtigen Reihenfolge erstellen)
```

### 2.3 Supabase Client
```typescript
// src/services/supabase.ts
import { createClient } from '@supabase/supabase-js'
import type { Database } from '@/types/database'

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY

if (!supabaseUrl || !supabaseAnonKey) {
  throw new Error('Missing Supabase environment variables')
}

export const supabase = createClient<Database>(
  supabaseUrl, 
  supabaseAnonKey,
  {
    auth: { persistSession: true },
    realtime: { enabled: false }
  }
)

// Hilfsfunktionen
export async function searchLocations(query: string) {
  const { data, error } = await supabase
    .from('voli_locations')
    .select('*')
    .textSearch('search_vector', query, {
      type: 'websearch',
      config: 'german'
    })
    .eq('active', true)
    .limit(5)

  if (error) throw error
  return data
}
```

## 🧩 Phase 3: Core Components

### 3.1 Answer Pipeline
```typescript
// src/services/answerPipeline.ts
import { supabase, searchLocations } from './supabase'
import { searchWeb } from './webSearch'
import { detectIntent } from './intentDetection'

export interface PipelineResult {
  answer: string
  sources: Source[]
  intent: IntentType
  confidence: number
}

export async function processQuery(
  query: string, 
  engine: any
): Promise<PipelineResult> {
  // 1. Intent erkennen
  const intent = await detectIntent(query)
  
  // 2. Cache prüfen
  const cached = await checkCache(query)
  if (cached) return cached
  
  // 3. Datenquellen abfragen
  let contextData: any[] = []
  let sources: Source[] = []
  
  try {
    // Datenbank zuerst
    if (intent.useDatabase) {
      const dbResults = await queryDatabase(query, intent)
      if (dbResults.length > 0) {
        contextData = dbResults
        sources.push({ type: 'database', items: dbResults })
      }
    }
    
    // Web-Suche als Fallback
    if (contextData.length === 0 && intent.useWebSearch) {
      const webResults = await searchWeb(query)
      if (webResults.length > 0) {
        contextData = webResults
        sources.push({ type: 'web', items: webResults })
      }
    }
  } catch (error) {
    console.error('Data fetch error:', error)
  }
  
  // 4. Kontext aufbereiten
  const context = prepareContext(contextData, intent)
  
  // 5. LLM-Antwort generieren
  const answer = await generateAnswer(
    query, 
    context, 
    engine,
    intent
  )
  
  // 6. Ergebnis cachen
  await updateCache(query, answer, sources)
  
  return {
    answer,
    sources,
    intent: intent.type,
    confidence: intent.confidence
  }
}
```

### 3.2 Model Manager
```typescript
// src/services/modelManager.ts
import * as webllm from '@mlc-ai/web-llm'

export class ModelManager {
  private smallEngine: webllm.MLCEngine | null = null
  private largeEngine: webllm.MLCEngine | null = null
  private currentEngine: 'small' | 'large' = 'small'
  private loadingLarge = false
  
  async initialize(
    onProgress: (progress: any) => void
  ) {
    // Kleines Modell sofort laden
    this.smallEngine = new webllm.MLCEngine({ 
      appConfig: {
        model_list: [{
          model_id: "Llama-3.2-1B-Instruct-q4f32_1-MLC",
          // ... config
        }]
      }
    })
    
    this.smallEngine.setInitProgressCallback(onProgress)
    await this.smallEngine.reload("Llama-3.2-1B-Instruct-q4f32_1-MLC")
    
    // Großes Modell im Hintergrund
    this.loadLargeModelInBackground()
  }
  
  private async loadLargeModelInBackground() {
    if (this.loadingLarge) return
    this.loadingLarge = true
    
    setTimeout(async () => {
      try {
        this.largeEngine = new webllm.MLCEngine({
          appConfig: {
            model_list: [{
              model_id: "Llama-3.2-3B-Instruct-q4f32_1-MLC",
              // ... config
            }]
          }
        })
        
        await this.largeEngine.reload("Llama-3.2-3B-Instruct-q4f32_1-MLC")
        this.currentEngine = 'large'
        console.log('Large model loaded successfully')
      } catch (error) {
        console.error('Failed to load large model:', error)
      }
    }, 5000) // 5 Sekunden Verzögerung
  }
  
  getCurrentEngine() {
    return this.currentEngine === 'large' && this.largeEngine 
      ? this.largeEngine 
      : this.smallEngine
  }
}
```

### 3.3 Voice Input Component
```typescript
// src/components/VoiceInput.tsx
import React, { useState, useEffect } from 'react'
import { useSpeechRecognition } from 'react-speech-kit'

interface VoiceInputProps {
  onTranscript: (text: string) => void
  disabled?: boolean
}

export const VoiceInput: React.FC<VoiceInputProps> = ({ 
  onTranscript, 
  disabled 
}) => {
  const [isListening, setIsListening] = useState(false)
  
  const { listen, listening, stop, supported } = useSpeechRecognition({
    onResult: (result: string) => {
      onTranscript(result)
      setIsListening(false)
    },
    onError: (error: any) => {
      console.error('Speech recognition error:', error)
      setIsListening(false)
    }
  })
  
  const toggleListening = () => {
    if (isListening) {
      stop()
    } else {
      listen({ lang: 'de-DE', continuous: false })
    }
    setIsListening(!isListening)
  }
  
  if (!supported) {
    return null // Browser unterstützt keine Spracherkennung
  }
  
  return (
    <button
      onClick={toggleListening}
      disabled={disabled}
      className={`voice-button ${isListening ? 'listening' : ''}`}
      aria-label="Spracherkennung"
    >
      {isListening ? '🎤' : '🎙️'}
    </button>
  )
}
```

## 🔌 Phase 4: API Integration

### 4.1 Web Search Service
```typescript
// src/services/webSearch.ts
import axios from 'axios'

const GOOGLE_API_KEY = import.meta.env.VITE_GOOGLE_API_KEY
const SEARCH_ENGINE_ID = import.meta.env.VITE_GOOGLE_SEARCH_ENGINE_ID

export interface WebSearchResult {
  title: string
  snippet: string
  link: string
  source: string
}

export async function searchWeb(
  query: string
): Promise<WebSearchResult[]> {
  try {
    const response = await axios.get(
      'https://www.googleapis.com/customsearch/v1',
      {
        params: {
          key: GOOGLE_API_KEY,
          cx: SEARCH_ENGINE_ID,
          q: `${query} Vorarlberg`,
          hl: 'de',
          gl: 'at',
          num: 5
        }
      }
    )
    
    return response.data.items?.map((item: any) => ({
      title: item.title,
      snippet: item.snippet,
      link: item.link,
      source: new URL(item.link).hostname
    })) || []
  } catch (error) {
    console.error('Web search error:', error)
    return []
  }
}

// Alternative: DuckDuckGo (keine API-Key nötig)
export async function searchDuckDuckGo(
  query: string
): Promise<WebSearchResult[]> {
  // Implementierung mit DuckDuckGo Instant Answer API
  // oder Web Scraping (mit Einschränkungen)
}
```

### 4.2 Intent Detection
```typescript
// src/services/intentDetection.ts
export enum IntentType {
  LOCATION = 'location',
  RESTAURANT = 'restaurant', 
  EVENT = 'event',
  HIKING = 'hiking',
  ACCOMMODATION = 'accommodation',
  WEATHER = 'weather',
  GENERAL = 'general'
}

const INTENT_KEYWORDS = {
  [IntentType.LOCATION]: [
    'sehenswürdigkeit', 'museum', 'kirche', 'schloss',
    'ort', 'platz', 'besichtigen', 'anschauen'
  ],
  [IntentType.RESTAURANT]: [
    'restaurant', 'essen', 'gasthaus', 'wirtshaus',
    'speisen', 'küche', 'lokal', 'café'
  ],
  [IntentType.HIKING]: [
    'wandern', 'wanderung', 'berg', 'tour',
    'route', 'weg', 'gipfel', 'alm'
  ]
  // ... weitere Intent-Keywords
}

export async function detectIntent(query: string) {
  const normalizedQuery = query.toLowerCase()
  let bestMatch = { type: IntentType.GENERAL, confidence: 0 }
  
  for (const [intent, keywords] of Object.entries(INTENT_KEYWORDS)) {
    let score = 0
    for (const keyword of keywords) {
      if (normalizedQuery.includes(keyword)) {
        score += 1
      }
    }
    
    const confidence = score / keywords.length
    if (confidence > bestMatch.confidence) {
      bestMatch = { 
        type: intent as IntentType, 
        confidence 
      }
    }
  }
  
  return {
    type: bestMatch.type,
    confidence: bestMatch.confidence,
    useDatabase: bestMatch.confidence > 0.3,
    useWebSearch: bestMatch.confidence < 0.5
  }
}
```

## 🎨 Phase 5: UI Components

### 5.1 Source Badge Component
```typescript
// src/components/SourceBadge.tsx
import React from 'react'

interface SourceBadgeProps {
  type: 'database' | 'web' | 'ai'
  count?: number
}

export const SourceBadge: React.FC<SourceBadgeProps> = ({ 
  type, 
  count 
}) => {
  const badges = {
    database: { icon: '🗄️', label: 'Lokal', color: '#4CAF50' },
    web: { icon: '🌐', label: 'Web', color: '#2196F3' },
    ai: { icon: '🤖', label: 'KI', color: '#FF9800' }
  }
  
  const badge = badges[type]
  
  return (
    <span 
      className="source-badge"
      style={{ backgroundColor: badge.color }}
    >
      {badge.icon} {badge.label}
      {count && ` (${count})`}
    </span>
  )
}
```

### 5.2 Model Status Component
```typescript
// src/components/ModelStatus.tsx
import React from 'react'

interface ModelStatusProps {
  smallLoaded: boolean
  largeLoaded: boolean
  largeLoading: boolean
  currentModel: 'small' | 'large'
}

export const ModelStatus: React.FC<ModelStatusProps> = ({
  smallLoaded,
  largeLoaded,
  largeLoading,
  currentModel
}) => {
  return (
    <div className="model-status">
      <div className="model-indicator">
        <span className={smallLoaded ? 'active' : ''}>
          1B {currentModel === 'small' && '✓'}
        </span>
        {largeLoading && (
          <span className="loading">3B lädt...</span>
        )}
        {largeLoaded && (
          <span className={largeLoaded ? 'active' : ''}>
            3B {currentModel === 'large' && '✓'}
          </span>
        )}
      </div>
    </div>
  )
}
```

## 🧪 Phase 6: Testing

### 6.1 Unit Tests
```typescript
// src/services/__tests__/intentDetection.test.ts
import { detectIntent, IntentType } from '../intentDetection'

describe('Intent Detection', () => {
  test('detects restaurant intent', async () => {
    const result = await detectIntent('Wo kann ich gut essen gehen?')
    expect(result.type).toBe(IntentType.RESTAURANT)
    expect(result.confidence).toBeGreaterThan(0.5)
  })
  
  test('detects hiking intent', async () => {
    const result = await detectIntent('Schöne Wanderung im Montafon')
    expect(result.type).toBe(IntentType.HIKING)
  })
})
```

### 6.2 Integration Tests
```typescript
// src/services/__tests__/answerPipeline.test.ts
import { processQuery } from '../answerPipeline'

describe('Answer Pipeline', () => {
  test('processes location query', async () => {
    const result = await processQuery(
      'Was gibt es in Bregenz zu sehen?',
      mockEngine
    )
    
    expect(result.answer).toBeTruthy()
    expect(result.sources.length).toBeGreaterThan(0)
    expect(result.intent).toBe('location')
  })
})
```

## 🚀 Phase 7: Deployment

### 7.1 Build für Production
```bash
# Environment variables setzen
cp .env.local .env.production

# Production build
npm run build

# Build testen
npm run preview
```

### 7.2 Vercel Deployment
```bash
# Vercel CLI installieren
npm i -g vercel

# Deploy
vercel --prod

# Environment Variables in Vercel setzen
vercel env add VITE_SUPABASE_URL
vercel env add VITE_SUPABASE_ANON_KEY
vercel env add VITE_GOOGLE_API_KEY
```

## 📝 Checkliste

### Vor Go-Live
- [ ] Alle Environment Variables gesetzt
- [ ] Supabase RLS aktiviert
- [ ] API Rate Limits konfiguriert
- [ ] Error Tracking (Sentry) eingerichtet
- [ ] Analytics eingebaut
- [ ] Performance getestet
- [ ] Mobile Experience getestet
- [ ] Accessibility geprüft
- [ ] Security Audit durchgeführt
- [ ] Backup-Strategie definiert

### Nach Go-Live
- [ ] Monitoring aktiviert
- [ ] User Feedback sammeln
- [ ] Performance überwachen
- [ ] Datenqualität prüfen
- [ ] Iterative Verbesserungen

---
*Implementierungs-Version: 1.0 | Stand: Januar 2025*