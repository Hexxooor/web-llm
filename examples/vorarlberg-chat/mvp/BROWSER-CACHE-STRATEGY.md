# 🗄️ Browser-Cache-Strategie für Vorarlberg Chat MVP

## Überblick

Da das MVP ohne Backend-Datenbank auskommt, nutzen wir verschiedene Browser-Storage-Technologien für die Datenpersistierung. Diese Strategie ermöglicht es, Chatverläufe, Benutzereinstellungen und gecachte Antworten lokal zu speichern.

## 🏗️ Storage-Architektur

### Storage-Hierarchie
```
Browser Storage
├── LocalStorage (einfache Daten)
│   ├── User Preferences
│   ├── App Settings
│   └── Cache Metadata
├── SessionStorage (temporäre Daten)
│   ├── Current Session
│   └── Temp Variables
└── IndexedDB (komplexe Daten)
    ├── Chat History
    ├── AI Responses Cache
    └── Static Data Cache
```

## 📦 Datenstrukturen

### TypeScript Interfaces
```typescript
// Chat-Verlauf
interface ChatHistory {
  id: string
  messages: Message[]
  timestamp: Date
  lastActive: Date
  title: string
  messageCount: number
}

interface Message {
  id: string
  role: 'user' | 'assistant'
  content: string
  timestamp: Date
  sources?: string[]
  intent?: string
  confidence?: number
}

// Benutzer-Einstellungen
interface UserPreferences {
  theme: 'light' | 'dark' | 'auto'
  language: 'de' | 'en'
  maxHistoryDays: number
  cacheEnabled: boolean
  modelPreference: 'fast' | 'accurate'
}

// Cache-Eintrag
interface CacheEntry {
  id: string
  query: string
  response: string
  sources: string[]
  timestamp: Date
  expiresAt: Date
  hitCount: number
}
```

## 🔧 Cache-Services

### LocalStorage Service
```typescript
class LocalStorageService {
  private readonly prefix = 'vorarlberg_chat_'
  
  // Benutzer-Einstellungen
  saveUserPreferences(prefs: UserPreferences): void {
    const key = `${this.prefix}user_prefs`
    localStorage.setItem(key, JSON.stringify(prefs))
  }
  
  loadUserPreferences(): UserPreferences | null {
    const key = `${this.prefix}user_prefs`
    const data = localStorage.getItem(key)
    return data ? JSON.parse(data) : null
  }
  
  // App-Metadaten
  saveAppMetadata(metadata: AppMetadata): void {
    const key = `${this.prefix}app_metadata`
    localStorage.setItem(key, JSON.stringify(metadata))
  }
  
  // Cache-Statistiken
  getCacheStats(): CacheStats {
    const entries = this.getAllCacheEntries()
    return {
      totalEntries: entries.length,
      totalSize: this.calculateStorageSize(),
      oldestEntry: entries.reduce((oldest, entry) => 
        entry.timestamp < oldest.timestamp ? entry : oldest
      ),
      hitRate: this.calculateHitRate()
    }
  }
}
```

### IndexedDB Service
```typescript
class IndexedDBService {
  private dbName = 'VorarlbergChatDB'
  private version = 1
  private db: IDBDatabase | null = null
  
  async initialize(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, this.version)
      
      request.onerror = () => reject(request.error)
      request.onsuccess = () => {
        this.db = request.result
        resolve()
      }
      
      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result
        
        // Chat History Store
        if (!db.objectStoreNames.contains('chatHistory')) {
          const chatStore = db.createObjectStore('chatHistory', { keyPath: 'id' })
          chatStore.createIndex('timestamp', 'timestamp')
          chatStore.createIndex('lastActive', 'lastActive')
        }
        
        // Response Cache Store
        if (!db.objectStoreNames.contains('responseCache')) {
          const cacheStore = db.createObjectStore('responseCache', { keyPath: 'id' })
          cacheStore.createIndex('query', 'query')
          cacheStore.createIndex('expiresAt', 'expiresAt')
        }
      }
    })
  }
  
  // Chat-Verlauf speichern
  async saveChatHistory(history: ChatHistory): Promise<void> {
    const transaction = this.db!.transaction(['chatHistory'], 'readwrite')
    const store = transaction.objectStore('chatHistory')
    await store.put(history)
  }
  
  // Chat-Verlauf laden
  async loadChatHistory(): Promise<ChatHistory[]> {
    const transaction = this.db!.transaction(['chatHistory'], 'readonly')
    const store = transaction.objectStore('chatHistory')
    const request = store.getAll()
    
    return new Promise((resolve, reject) => {
      request.onsuccess = () => resolve(request.result)
      request.onerror = () => reject(request.error)
    })
  }
  
  // Response cachen
  async cacheResponse(entry: CacheEntry): Promise<void> {
    const transaction = this.db!.transaction(['responseCache'], 'readwrite')
    const store = transaction.objectStore('responseCache')
    await store.put(entry)
  }
  
  // Gecachte Response laden
  async getCachedResponse(query: string): Promise<CacheEntry | null> {
    const transaction = this.db!.transaction(['responseCache'], 'readonly')
    const store = transaction.objectStore('responseCache')
    const index = store.index('query')
    const request = index.get(query)
    
    return new Promise((resolve, reject) => {
      request.onsuccess = () => {
        const result = request.result
        if (result && result.expiresAt > new Date()) {
          // Hit Count erhöhen
          result.hitCount += 1
          store.put(result)
          resolve(result)
        } else {
          resolve(null)
        }
      }
      request.onerror = () => reject(request.error)
    })
  }
}
```

## 🧹 Cache-Management

### Automatische Bereinigung
```typescript
class CacheManager {
  private localStorage = new LocalStorageService()
  private indexedDB = new IndexedDBService()
  
  // Täglich automatisch ausführen
  async cleanupExpiredEntries(): Promise<void> {
    const now = new Date()
    
    // Expired Response Cache löschen
    const transaction = this.indexedDB.db!.transaction(['responseCache'], 'readwrite')
    const store = transaction.objectStore('responseCache')
    const index = store.index('expiresAt')
    
    const request = index.openCursor(IDBKeyRange.upperBound(now))
    request.onsuccess = (event) => {
      const cursor = (event.target as IDBRequest).result
      if (cursor) {
        cursor.delete()
        cursor.continue()
      }
    }
  }
  
  // Alte Chat-Verläufe archivieren
  async archiveOldChats(maxDays: number = 30): Promise<void> {
    const cutoffDate = new Date()
    cutoffDate.setDate(cutoffDate.getDate() - maxDays)
    
    const allChats = await this.indexedDB.loadChatHistory()
    const oldChats = allChats.filter(chat => chat.lastActive < cutoffDate)
    
    // In separate Archive-Tabelle verschieben
    for (const chat of oldChats) {
      await this.archiveChat(chat)
      await this.deleteChatHistory(chat.id)
    }
  }
  
  // Storage-Limits überwachen
  async checkStorageLimits(): Promise<StorageStatus> {
    const estimate = await navigator.storage.estimate()
    const usage = estimate.usage || 0
    const quota = estimate.quota || 0
    
    return {
      usedBytes: usage,
      totalBytes: quota,
      usagePercent: (usage / quota) * 100,
      warning: (usage / quota) > 0.8, // Warnung bei 80%
      critical: (usage / quota) > 0.95 // Kritisch bei 95%
    }
  }
}
```

## 🔄 Sync-Strategien

### Offline-First Ansatz
```typescript
class OfflineManager {
  private cacheManager = new CacheManager()
  
  // Offline-Erkennung
  isOnline(): boolean {
    return navigator.onLine
  }
  
  // Queue für Offline-Aktionen
  private offlineQueue: OfflineAction[] = []
  
  async queueOfflineAction(action: OfflineAction): Promise<void> {
    this.offlineQueue.push(action)
    this.localStorage.saveOfflineQueue(this.offlineQueue)
  }
  
  // Synchronisierung wenn wieder online
  async syncWhenOnline(): Promise<void> {
    if (!this.isOnline()) return
    
    const queue = this.offlineQueue
    this.offlineQueue = []
    
    for (const action of queue) {
      try {
        await this.executeAction(action)
      } catch (error) {
        console.error('Sync error:', error)
        // Zurück in die Queue
        this.offlineQueue.push(action)
      }
    }
  }
}
```

## 📊 Performance-Optimierungen

### Lazy Loading
```typescript
class DataLoader {
  private loadedChats = new Set<string>()
  
  // Chat-Verlauf lazy laden
  async loadChatHistoryLazy(chatId: string): Promise<ChatHistory> {
    if (this.loadedChats.has(chatId)) {
      return this.getCachedChat(chatId)
    }
    
    const chat = await this.indexedDB.loadChatHistory(chatId)
    this.loadedChats.add(chatId)
    return chat
  }
  
  // Paginierung für große Chat-Listen
  async loadChatPage(page: number, limit: number = 20): Promise<ChatHistory[]> {
    const offset = page * limit
    const transaction = this.indexedDB.db!.transaction(['chatHistory'], 'readonly')
    const store = transaction.objectStore('chatHistory')
    const index = store.index('lastActive')
    
    const request = index.openCursor(null, 'prev') // Neueste zuerst
    const results: ChatHistory[] = []
    
    return new Promise((resolve) => {
      let count = 0
      request.onsuccess = (event) => {
        const cursor = (event.target as IDBRequest).result
        if (cursor && results.length < limit) {
          if (count >= offset) {
            results.push(cursor.value)
          }
          count++
          cursor.continue()
        } else {
          resolve(results)
        }
      }
    })
  }
}
```

## 🔐 Datenschutz & Sicherheit

### Lokale Daten-Verschlüsselung
```typescript
class EncryptionService {
  private key: CryptoKey | null = null
  
  async generateKey(): Promise<CryptoKey> {
    return await crypto.subtle.generateKey(
      { name: 'AES-GCM', length: 256 },
      true,
      ['encrypt', 'decrypt']
    )
  }
  
  async encryptData(data: string): Promise<string> {
    const encoder = new TextEncoder()
    const dataBuffer = encoder.encode(data)
    const iv = crypto.getRandomValues(new Uint8Array(12))
    
    const encrypted = await crypto.subtle.encrypt(
      { name: 'AES-GCM', iv },
      this.key!,
      dataBuffer
    )
    
    return btoa(String.fromCharCode(...new Uint8Array(encrypted)))
  }
  
  async decryptData(encryptedData: string): Promise<string> {
    const dataBuffer = new Uint8Array(atob(encryptedData).split('').map(c => c.charCodeAt(0)))
    
    const decrypted = await crypto.subtle.decrypt(
      { name: 'AES-GCM', iv: this.iv },
      this.key!,
      dataBuffer
    )
    
    return new TextDecoder().decode(decrypted)
  }
}
```

## 📱 Mobile Optimierungen

### Storage-Limits für Mobile
```typescript
class MobileStorageManager {
  // Reduzierte Limits für mobile Geräte
  private mobileConfig = {
    maxChatHistory: 50,
    maxCacheEntries: 200,
    maxCacheDays: 7,
    maxMessageLength: 1000
  }
  
  isMobile(): boolean {
    return /Android|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)
  }
  
  getStorageConfig(): StorageConfig {
    return this.isMobile() ? this.mobileConfig : this.desktopConfig
  }
  
  // Aggressive Bereinigung auf mobilen Geräten
  async aggressiveCleanup(): Promise<void> {
    if (!this.isMobile()) return
    
    const config = this.getStorageConfig()
    
    // Nur neueste Chats behalten
    const allChats = await this.indexedDB.loadChatHistory()
    const sortedChats = allChats.sort((a, b) => b.lastActive.getTime() - a.lastActive.getTime())
    const chatsToDelete = sortedChats.slice(config.maxChatHistory)
    
    for (const chat of chatsToDelete) {
      await this.indexedDB.deleteChatHistory(chat.id)
    }
  }
}
```

## 🛠️ Implementierung

### Service Integration
```typescript
class CacheService {
  private localStorage = new LocalStorageService()
  private indexedDB = new IndexedDBService()
  private cacheManager = new CacheManager()
  private offlineManager = new OfflineManager()
  
  async initialize(): Promise<void> {
    await this.indexedDB.initialize()
    
    // Automatische Bereinigung starten
    setInterval(() => {
      this.cacheManager.cleanupExpiredEntries()
    }, 24 * 60 * 60 * 1000) // Täglich
    
    // Online/Offline Event-Listener
    window.addEventListener('online', () => {
      this.offlineManager.syncWhenOnline()
    })
  }
  
  // Einheitliche API für Components
  async saveMessage(message: Message): Promise<void> {
    const currentChat = await this.getCurrentChat()
    currentChat.messages.push(message)
    currentChat.lastActive = new Date()
    
    await this.indexedDB.saveChatHistory(currentChat)
  }
  
  async getCachedResponse(query: string): Promise<string | null> {
    const cached = await this.indexedDB.getCachedResponse(query)
    return cached?.response || null
  }
}
```

## 🎯 Best Practices

### 1. Datenschutz
- Sensible Daten verschlüsseln
- Regelmäßige Bereinigung
- Benutzer-Kontrolle über Daten

### 2. Performance
- Lazy Loading für große Datasets
- Paginierung für Listen
- Indexierung für schnelle Suche

### 3. Benutzerfreundlichkeit
- Transparente Cache-Verwaltung
- Offline-Funktionalität
- Einstellungen für Speicher-Limits

### 4. Fehlerbehandlung
- Graceful Degradation
- Fallback-Strategien
- Error Recovery

## 📈 Monitoring

### Storage-Metriken
```typescript
class StorageMonitor {
  async getStorageMetrics(): Promise<StorageMetrics> {
    const estimate = await navigator.storage.estimate()
    const cacheStats = await this.cacheManager.getCacheStats()
    
    return {
      totalUsage: estimate.usage || 0,
      totalQuota: estimate.quota || 0,
      chatHistorySize: await this.getChatHistorySize(),
      cacheSize: await this.getCacheSize(),
      hitRate: cacheStats.hitRate,
      oldestEntry: cacheStats.oldestEntry
    }
  }
}
```

---
*Browser-Cache-Strategie Version 1.0 - Optimiert für MVP*