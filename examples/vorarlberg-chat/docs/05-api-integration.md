# 🔌 API Integration Guide: Ländle Chatbot

## Übersicht

Dieses Dokument beschreibt die Integration externer APIs und Services in den Ländle Chatbot. Alle APIs werden sicher und effizient eingebunden mit Fokus auf Performance und Fehlerbehandlung.

## 🔑 API-Verwaltung

### Environment Variables
```env
# .env.local - Entwicklung
VITE_SUPABASE_URL=https://xxx.supabase.co
VITE_SUPABASE_ANON_KEY=xxx
VITE_GOOGLE_API_KEY=xxx
VITE_GOOGLE_SEARCH_ENGINE_ID=xxx
VITE_WEATHER_API_KEY=xxx
VITE_MAPS_API_KEY=xxx

# Zusätzliche Sicherheit
VITE_API_RATE_LIMIT=100
VITE_API_TIMEOUT=5000
```

### API Key Manager
```typescript
// src/services/apiKeyManager.ts
class APIKeyManager {
  private keys: Map<string, string> = new Map()
  private rateLimits: Map<string, RateLimit> = new Map()
  
  constructor() {
    this.loadKeys()
    this.initializeRateLimits()
  }
  
  private loadKeys() {
    // Sichere API-Keys aus Environment laden
    this.keys.set('google', import.meta.env.VITE_GOOGLE_API_KEY || '')
    this.keys.set('weather', import.meta.env.VITE_WEATHER_API_KEY || '')
    this.keys.set('maps', import.meta.env.VITE_MAPS_API_KEY || '')
  }
  
  getKey(service: string): string {
    const key = this.keys.get(service)
    if (!key) throw new Error(`API key for ${service} not found`)
    return key
  }
  
  async checkRateLimit(service: string): Promise<boolean> {
    const limit = this.rateLimits.get(service)
    if (!limit) return true
    
    return limit.canMakeRequest()
  }
}

export const apiKeyManager = new APIKeyManager()
```

## 🌐 Google Search API

### Konfiguration
```typescript
// src/services/apis/googleSearch.ts
import { apiKeyManager } from '../apiKeyManager'

interface GoogleSearchConfig {
  apiKey: string
  engineId: string
  baseUrl: string
  defaultParams: {
    hl: string // Sprache
    gl: string // Land
    num: number // Ergebnisse
  }
}

const config: GoogleSearchConfig = {
  apiKey: apiKeyManager.getKey('google'),
  engineId: import.meta.env.VITE_GOOGLE_SEARCH_ENGINE_ID,
  baseUrl: 'https://www.googleapis.com/customsearch/v1',
  defaultParams: {
    hl: 'de',
    gl: 'at',
    num: 5
  }
}
```

### Implementierung
```typescript
export class GoogleSearchService {
  private cache = new Map<string, CachedResult>()
  
  async search(
    query: string, 
    options?: SearchOptions
  ): Promise<SearchResult[]> {
    // Cache prüfen
    const cacheKey = this.getCacheKey(query, options)
    const cached = this.cache.get(cacheKey)
    if (cached && !this.isExpired(cached)) {
      return cached.data
    }
    
    // Rate Limit prüfen
    if (!await apiKeyManager.checkRateLimit('google')) {
      throw new Error('Rate limit exceeded')
    }
    
    try {
      const response = await axios.get(config.baseUrl, {
        params: {
          key: config.apiKey,
          cx: config.engineId,
          q: this.enhanceQuery(query),
          ...config.defaultParams,
          ...options
        },
        timeout: 5000
      })
      
      const results = this.parseResults(response.data)
      
      // Cache aktualisieren
      this.cache.set(cacheKey, {
        data: results,
        timestamp: Date.now()
      })
      
      return results
    } catch (error) {
      return this.handleError(error)
    }
  }
  
  private enhanceQuery(query: string): string {
    // Vorarlberg-spezifische Verbesserungen
    const enhanced = `${query} Vorarlberg`
    
    // Lokale Websites priorisieren
    const localSites = [
      'site:vorarlberg.at',
      'site:vol.at',
      'site:laendle.at',
      'site:vorarlberg.travel'
    ].join(' OR ')
    
    return `${enhanced} (${localSites})`
  }
  
  private parseResults(data: any): SearchResult[] {
    return data.items?.map((item: any) => ({
      title: item.title,
      snippet: this.cleanSnippet(item.snippet),
      url: item.link,
      source: new URL(item.link).hostname,
      publishedDate: item.pagemap?.metatags?.[0]?.['article:published_time'],
      imageUrl: item.pagemap?.cse_image?.[0]?.src
    })) || []
  }
}
```

## 🗄️ Supabase Integration

### Client Setup
```typescript
// src/services/apis/supabase.ts
import { createClient } from '@supabase/supabase-js'
import type { Database } from '@/types/database'

class SupabaseService {
  private client: ReturnType<typeof createClient<Database>>
  
  constructor() {
    this.client = createClient<Database>(
      import.meta.env.VITE_SUPABASE_URL,
      import.meta.env.VITE_SUPABASE_ANON_KEY,
      {
        auth: { persistSession: true },
        global: {
          headers: { 'x-region': 'vorarlberg' }
        }
      }
    )
  }
  
  // Locations
  async searchLocations(params: LocationSearchParams) {
    const query = this.client
      .from('voli_locations')
      .select(`
        *,
        restaurants:voli_restaurants(count),
        events:voli_events(count)
      `)
      .eq('active', true)
    
    // Text-Suche
    if (params.searchText) {
      query.textSearch('search_vector', params.searchText, {
        type: 'websearch',
        config: 'german'
      })
    }
    
    // Geo-Filter
    if (params.nearLocation) {
      query.rpc('find_nearby', {
        lat: params.nearLocation.lat,
        lng: params.nearLocation.lng,
        radius_km: params.radius || 10
      })
    }
    
    // Kategorie-Filter
    if (params.category) {
      query.eq('category', params.category)
    }
    
    const { data, error } = await query.limit(params.limit || 10)
    
    if (error) throw error
    return data
  }
  
  // Events
  async getUpcomingEvents(days: number = 7) {
    const today = new Date().toISOString().split('T')[0]
    const future = new Date()
    future.setDate(future.getDate() + days)
    
    const { data, error } = await this.client
      .from('voli_events')
      .select(`
        *,
        location:voli_locations(name, city, latitude, longitude)
      `)
      .gte('start_date', today)
      .lte('start_date', future.toISOString().split('T')[0])
      .eq('status', 'scheduled')
      .order('start_date', { ascending: true })
    
    if (error) throw error
    return data
  }
  
  // Real-time Subscriptions
  subscribeToEvents(callback: (event: any) => void) {
    return this.client
      .channel('events')
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'voli_events'
        },
        callback
      )
      .subscribe()
  }
}

export const supabaseService = new SupabaseService()
```

## 🌤️ Weather API

### OpenWeatherMap Integration
```typescript
// src/services/apis/weather.ts
interface WeatherConfig {
  apiKey: string
  baseUrl: string
  units: 'metric' | 'imperial'
  lang: string
}

class WeatherService {
  private config: WeatherConfig = {
    apiKey: apiKeyManager.getKey('weather'),
    baseUrl: 'https://api.openweathermap.org/data/2.5',
    units: 'metric',
    lang: 'de'
  }
  
  async getCurrentWeather(location: string): Promise<WeatherData> {
    try {
      const response = await axios.get(`${this.config.baseUrl}/weather`, {
        params: {
          q: `${location},AT`,
          appid: this.config.apiKey,
          units: this.config.units,
          lang: this.config.lang
        }
      })
      
      return this.formatWeatherData(response.data)
    } catch (error) {
      throw new Error('Wetterdaten nicht verfügbar')
    }
  }
  
  async getForecast(location: string, days: number = 5): Promise<ForecastData[]> {
    const response = await axios.get(`${this.config.baseUrl}/forecast`, {
      params: {
        q: `${location},AT`,
        appid: this.config.apiKey,
        units: this.config.units,
        lang: this.config.lang,
        cnt: days * 8 // 8 Messungen pro Tag (alle 3 Stunden)
      }
    })
    
    return this.formatForecastData(response.data)
  }
  
  private formatWeatherData(data: any): WeatherData {
    return {
      temperature: Math.round(data.main.temp),
      feelsLike: Math.round(data.main.feels_like),
      description: data.weather[0].description,
      icon: this.getWeatherEmoji(data.weather[0].id),
      humidity: data.main.humidity,
      windSpeed: data.wind.speed,
      location: data.name
    }
  }
  
  private getWeatherEmoji(code: number): string {
    if (code >= 200 && code < 300) return '⛈️'
    if (code >= 300 && code < 400) return '🌦️'
    if (code >= 500 && code < 600) return '🌧️'
    if (code >= 600 && code < 700) return '❄️'
    if (code >= 700 && code < 800) return '🌫️'
    if (code === 800) return '☀️'
    if (code > 800) return '☁️'
    return '🌈'
  }
}
```

## 🗺️ Maps Integration

### Google Maps für Standorte
```typescript
// src/services/apis/maps.ts
class MapsService {
  private apiKey = apiKeyManager.getKey('maps')
  
  async geocodeAddress(address: string): Promise<Coordinates> {
    const response = await axios.get(
      'https://maps.googleapis.com/maps/api/geocode/json',
      {
        params: {
          address: `${address}, Vorarlberg, Austria`,
          key: this.apiKey,
          language: 'de'
        }
      }
    )
    
    const location = response.data.results[0]?.geometry.location
    if (!location) throw new Error('Adresse nicht gefunden')
    
    return {
      lat: location.lat,
      lng: location.lng
    }
  }
  
  async getDirections(
    origin: string, 
    destination: string,
    mode: 'driving' | 'walking' | 'transit' = 'driving'
  ): Promise<DirectionsResult> {
    const response = await axios.get(
      'https://maps.googleapis.com/maps/api/directions/json',
      {
        params: {
          origin,
          destination,
          mode,
          key: this.apiKey,
          language: 'de',
          region: 'at'
        }
      }
    )
    
    return this.parseDirections(response.data)
  }
  
  generateStaticMapUrl(
    location: Coordinates,
    options: MapOptions = {}
  ): string {
    const params = new URLSearchParams({
      center: `${location.lat},${location.lng}`,
      zoom: options.zoom?.toString() || '14',
      size: options.size || '400x300',
      maptype: options.mapType || 'roadmap',
      markers: `color:red|${location.lat},${location.lng}`,
      key: this.apiKey
    })
    
    return `https://maps.googleapis.com/maps/api/staticmap?${params}`
  }
}
```

## 🔄 Fallback-Strategien

### API-Fehlerbehandlung
```typescript
// src/services/apis/fallbackHandler.ts
class FallbackHandler {
  async handleAPIFailure(
    service: string,
    error: any,
    context: any
  ): Promise<any> {
    console.error(`API ${service} failed:`, error)
    
    // Service-spezifische Fallbacks
    switch (service) {
      case 'google-search':
        return this.fallbackToLocalSearch(context.query)
      
      case 'weather':
        return this.fallbackToLastKnownWeather(context.location)
      
      case 'maps':
        return this.fallbackToStaticCoordinates(context.address)
      
      default:
        return this.genericFallback(service)
    }
  }
  
  private async fallbackToLocalSearch(query: string) {
    // Nur Datenbank durchsuchen
    try {
      const results = await supabaseService.searchLocations({
        searchText: query,
        limit: 10
      })
      
      return results.map(r => ({
        title: r.name,
        snippet: r.description,
        url: '#',
        source: 'Lokale Datenbank'
      }))
    } catch (error) {
      return []
    }
  }
  
  private async fallbackToLastKnownWeather(location: string) {
    // Aus Cache oder Default
    const cached = await getCachedWeather(location)
    if (cached) return cached
    
    return {
      temperature: 15,
      description: 'Wetterdaten momentan nicht verfügbar',
      icon: '🌤️',
      location
    }
  }
}
```

## 📊 API Monitoring

### Usage Tracking
```typescript
// src/services/apis/apiMonitor.ts
interface APIUsage {
  service: string
  endpoint: string
  timestamp: Date
  duration: number
  success: boolean
  error?: string
}

class APIMonitor {
  private usage: APIUsage[] = []
  
  async trackAPICall(
    service: string,
    endpoint: string,
    call: () => Promise<any>
  ): Promise<any> {
    const start = Date.now()
    let success = true
    let error: string | undefined
    
    try {
      const result = await call()
      return result
    } catch (err) {
      success = false
      error = err.message
      throw err
    } finally {
      this.usage.push({
        service,
        endpoint,
        timestamp: new Date(),
        duration: Date.now() - start,
        success,
        error
      })
      
      // Analytics senden
      this.sendAnalytics()
    }
  }
  
  getUsageStats(service?: string): UsageStats {
    const filtered = service 
      ? this.usage.filter(u => u.service === service)
      : this.usage
    
    return {
      totalCalls: filtered.length,
      successRate: filtered.filter(u => u.success).length / filtered.length,
      avgDuration: filtered.reduce((sum, u) => sum + u.duration, 0) / filtered.length,
      errors: filtered.filter(u => !u.success).map(u => u.error)
    }
  }
}
```

## 🚀 Best Practices

### 1. Caching
- Aggressives Caching für statische Daten
- TTL basierend auf Datentyp
- Invalidierung bei Updates

### 2. Rate Limiting
- Client-seitiges Rate Limiting
- Exponential Backoff bei Fehlern
- Request Batching wo möglich

### 3. Error Recovery
- Graceful Degradation
- Fallback auf lokale Daten
- User-freundliche Fehlermeldungen

### 4. Security
- API Keys niemals im Frontend exponieren
- CORS richtig konfigurieren
- Request Signing für kritische APIs

---
*API Integration Version: 1.0 | Stand: Januar 2025*