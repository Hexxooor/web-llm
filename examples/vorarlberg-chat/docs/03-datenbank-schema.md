# 🗄️ Datenbank-Schema: Ländle Chatbot

## Übersicht

Das Datenbank-Schema ist für Supabase (PostgreSQL) optimiert und verwendet den Präfix `voli_` für alle Tabellen. Die Struktur unterstützt effiziente Abfragen für lokale Informationen über Vorarlberg.

## 📊 Haupt-Tabellen

### voli_locations (Sehenswürdigkeiten & Orte)
```sql
CREATE TABLE voli_locations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  name_normalized VARCHAR(255) GENERATED ALWAYS AS (lower(unaccent(name))) STORED,
  description TEXT,
  category VARCHAR(100) NOT NULL,
  sub_category VARCHAR(100),
  
  -- Adresse
  street VARCHAR(255),
  house_number VARCHAR(20),
  postal_code VARCHAR(10),
  city VARCHAR(100) NOT NULL,
  district VARCHAR(100),
  
  -- Geodaten
  latitude DECIMAL(10, 8),
  longitude DECIMAL(11, 8),
  altitude INTEGER,
  
  -- Kontakt
  phone VARCHAR(50),
  email VARCHAR(255),
  website VARCHAR(500),
  
  -- Öffnungszeiten als JSON
  opening_hours JSONB,
  /* Beispiel:
  {
    "monday": {"open": "09:00", "close": "18:00"},
    "tuesday": {"open": "09:00", "close": "18:00"},
    "special": [
      {"date": "2025-12-24", "open": "09:00", "close": "14:00"}
    ]
  }
  */
  
  -- Medien
  images TEXT[],
  main_image VARCHAR(500),
  
  -- Zusatzinfos
  accessibility JSONB,
  parking JSONB,
  public_transport JSONB,
  
  -- Metadaten
  tags TEXT[],
  popularity_score INTEGER DEFAULT 0,
  verified BOOLEAN DEFAULT false,
  active BOOLEAN DEFAULT true,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  created_by UUID REFERENCES auth.users(id),
  
  -- Full-Text Search
  search_vector tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('german', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('german', coalesce(description, '')), 'B') ||
    setweight(to_tsvector('german', coalesce(array_to_string(tags, ' '), '')), 'C')
  ) STORED
);

-- Indizes
CREATE INDEX idx_voli_locations_search ON voli_locations USING GIN(search_vector);
CREATE INDEX idx_voli_locations_category ON voli_locations(category, active);
CREATE INDEX idx_voli_locations_city ON voli_locations(city, active);
CREATE INDEX idx_voli_locations_geo ON voli_locations USING GIST(
  ll_to_earth(latitude, longitude)
);
```

### voli_restaurants (Restaurants & Gastronomie)
```sql
CREATE TABLE voli_restaurants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  location_id UUID REFERENCES voli_locations(id),
  
  -- Basis-Info
  name VARCHAR(255) NOT NULL,
  description TEXT,
  cuisine_types TEXT[],
  
  -- Details
  price_range INTEGER CHECK (price_range BETWEEN 1 AND 4),
  capacity INTEGER,
  
  -- Features
  features JSONB,
  /* Beispiel:
  {
    "vegetarian": true,
    "vegan": true,
    "gluten_free": true,
    "outdoor_seating": true,
    "wifi": true,
    "parking": true,
    "reservation_required": false,
    "takeaway": true,
    "delivery": true
  }
  */
  
  -- Speisekarte
  menu_url VARCHAR(500),
  menu_highlights TEXT[],
  specialties TEXT[],
  
  -- Bewertungen
  rating_average DECIMAL(2,1),
  rating_count INTEGER DEFAULT 0,
  
  -- Metadaten
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_voli_restaurants_cuisine ON voli_restaurants USING GIN(cuisine_types);
CREATE INDEX idx_voli_restaurants_features ON voli_restaurants USING GIN(features);
```

### voli_events (Veranstaltungen)
```sql
CREATE TABLE voli_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  location_id UUID REFERENCES voli_locations(id),
  
  -- Basis-Info
  title VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(100) NOT NULL,
  
  -- Zeitangaben
  start_date DATE NOT NULL,
  end_date DATE,
  start_time TIME,
  end_time TIME,
  
  -- Wiederholung
  recurrence JSONB,
  /* Beispiel:
  {
    "type": "weekly",
    "days": ["monday", "wednesday"],
    "until": "2025-12-31"
  }
  */
  
  -- Details
  organizer VARCHAR(255),
  contact_info JSONB,
  
  -- Tickets
  ticket_info JSONB,
  /* Beispiel:
  {
    "price_from": 15.00,
    "price_to": 45.00,
    "currency": "EUR",
    "booking_url": "https://...",
    "available": true
  }
  */
  
  -- Medien
  images TEXT[],
  
  -- Status
  status VARCHAR(50) DEFAULT 'scheduled',
  capacity INTEGER,
  
  -- Metadaten
  tags TEXT[],
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_voli_events_dates ON voli_events(start_date, end_date);
CREATE INDEX idx_voli_events_category ON voli_events(category, status);
```

### voli_hiking_routes (Wanderwege)
```sql
CREATE TABLE voli_hiking_routes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Basis-Info
  name VARCHAR(255) NOT NULL,
  description TEXT,
  
  -- Route Details
  difficulty VARCHAR(50) CHECK (difficulty IN ('leicht', 'mittel', 'schwer', 'sehr_schwer')),
  distance_km DECIMAL(5,2),
  duration_minutes INTEGER,
  elevation_gain INTEGER,
  elevation_loss INTEGER,
  
  -- Start/Ziel
  start_point JSONB,
  end_point JSONB,
  is_circular BOOLEAN DEFAULT false,
  
  -- GPS Track
  gpx_url VARCHAR(500),
  waypoints JSONB[],
  
  -- Bedingungen
  season_from INTEGER CHECK (season_from BETWEEN 1 AND 12),
  season_to INTEGER CHECK (season_to BETWEEN 1 AND 12),
  
  -- Features
  features JSONB,
  /* Beispiel:
  {
    "family_friendly": true,
    "dog_allowed": true,
    "stroller_suitable": false,
    "alpine_hut": true,
    "viewpoint": true,
    "waterfall": false
  }
  */
  
  -- Sicherheit
  safety_info TEXT,
  equipment_needed TEXT[],
  
  -- Bewertungen
  rating_average DECIMAL(2,1),
  rating_count INTEGER DEFAULT 0,
  
  -- Metadaten
  tags TEXT[],
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### voli_accommodations (Unterkünfte)
```sql
CREATE TABLE voli_accommodations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  location_id UUID REFERENCES voli_locations(id),
  
  -- Basis-Info
  name VARCHAR(255) NOT NULL,
  type VARCHAR(100) NOT NULL, -- hotel, pension, ferienwohnung, etc.
  category INTEGER CHECK (category BETWEEN 1 AND 5), -- Sterne
  
  -- Kapazität
  rooms INTEGER,
  beds INTEGER,
  
  -- Preise
  price_from DECIMAL(10,2),
  price_to DECIMAL(10,2),
  price_info JSONB,
  
  -- Ausstattung
  amenities JSONB,
  /* Beispiel:
  {
    "wifi": true,
    "parking": true,
    "breakfast": true,
    "spa": true,
    "pool": false,
    "gym": true,
    "restaurant": true,
    "bar": true,
    "pets_allowed": false
  }
  */
  
  -- Buchung
  booking_url VARCHAR(500),
  booking_email VARCHAR(255),
  booking_phone VARCHAR(50),
  
  -- Bewertungen
  rating_average DECIMAL(2,1),
  rating_count INTEGER DEFAULT 0,
  
  -- Metadaten
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 🔗 Beziehungs-Tabellen

### voli_location_categories
```sql
CREATE TABLE voli_location_categories (
  id SERIAL PRIMARY KEY,
  category VARCHAR(100) UNIQUE NOT NULL,
  parent_category VARCHAR(100),
  icon VARCHAR(50),
  sort_order INTEGER DEFAULT 0,
  active BOOLEAN DEFAULT true
);

-- Beispieldaten
INSERT INTO voli_location_categories (category, parent_category, icon) VALUES
('Sehenswürdigkeit', NULL, '🏛️'),
('Museum', 'Sehenswürdigkeit', '🖼️'),
('Kirche', 'Sehenswürdigkeit', '⛪'),
('Natur', NULL, '🌲'),
('See', 'Natur', '🏞️'),
('Berg', 'Natur', '⛰️');
```

## 📈 Hilfs-Tabellen

### voli_cache (Query Cache)
```sql
CREATE TABLE voli_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cache_key VARCHAR(255) UNIQUE NOT NULL,
  query_text TEXT NOT NULL,
  result JSONB NOT NULL,
  hit_count INTEGER DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_voli_cache_key ON voli_cache(cache_key, expires_at);
```

### voli_faqs (Häufige Fragen)
```sql
CREATE TABLE voli_faqs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  question TEXT NOT NULL,
  answer TEXT NOT NULL,
  category VARCHAR(100),
  keywords TEXT[],
  view_count INTEGER DEFAULT 0,
  helpful_count INTEGER DEFAULT 0,
  sort_order INTEGER DEFAULT 0,
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### voli_data_sources (Datenquellen)
```sql
CREATE TABLE voli_data_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50), -- api, scraper, manual
  url VARCHAR(500),
  last_sync TIMESTAMPTZ,
  sync_frequency INTERVAL,
  active BOOLEAN DEFAULT true,
  config JSONB
);
```

## 🔒 Row Level Security (RLS)

```sql
-- Lesezugriff für alle
ALTER TABLE voli_locations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Locations sind öffentlich lesbar" 
  ON voli_locations FOR SELECT 
  USING (active = true);

-- Schreibzugriff nur für Admins
CREATE POLICY "Nur Admins können Locations bearbeiten" 
  ON voli_locations FOR ALL 
  USING (auth.jwt() ->> 'role' = 'admin');

-- Ähnliche Policies für alle anderen Tabellen
```

## 🚀 Performance-Optimierungen

### Materialized Views
```sql
-- Beliebte Orte
CREATE MATERIALIZED VIEW voli_popular_locations AS
SELECT 
  l.*,
  COUNT(DISTINCT e.id) as event_count,
  AVG(r.rating_average) as avg_rating
FROM voli_locations l
LEFT JOIN voli_events e ON l.id = e.location_id
LEFT JOIN voli_restaurants r ON l.id = r.location_id
WHERE l.active = true
GROUP BY l.id
ORDER BY l.popularity_score DESC;

-- Refresh täglich
CREATE OR REPLACE FUNCTION refresh_popular_locations()
RETURNS void AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY voli_popular_locations;
END;
$$ LANGUAGE plpgsql;
```

### Funktionen
```sql
-- Umkreissuche
CREATE OR REPLACE FUNCTION voli_find_nearby(
  lat DECIMAL,
  lng DECIMAL,
  radius_km INTEGER DEFAULT 10
)
RETURNS TABLE(
  id UUID,
  name VARCHAR,
  distance_km DECIMAL
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    l.id,
    l.name,
    earth_distance(
      ll_to_earth(lat, lng),
      ll_to_earth(l.latitude, l.longitude)
    ) / 1000 as distance_km
  FROM voli_locations l
  WHERE 
    l.active = true AND
    earth_box(ll_to_earth(lat, lng), radius_km * 1000) @> ll_to_earth(l.latitude, l.longitude)
  ORDER BY distance_km;
END;
$$ LANGUAGE plpgsql;
```

## 📝 Migrations

### Initial Migration
```sql
-- 001_initial_schema.sql
BEGIN;

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "unaccent";
CREATE EXTENSION IF NOT EXISTS "earthdistance" CASCADE;

-- Tabellen erstellen (siehe oben)

COMMIT;
```

### Seed Data
```sql
-- 002_seed_data.sql
BEGIN;

-- Kategorien
INSERT INTO voli_location_categories (category, icon) VALUES
('Restaurant', '🍽️'),
('Hotel', '🏨'),
('Wanderweg', '🥾'),
('Museum', '🏛️'),
('Veranstaltung', '🎭');

-- Beispiel-Locations
INSERT INTO voli_locations (name, city, category) VALUES
('Schattenburg', 'Feldkirch', 'Sehenswürdigkeit'),
('Bodensee', 'Bregenz', 'Natur'),
('Montafon', 'Schruns', 'Region');

COMMIT;
```

---
*Schema-Version: 1.0 | Stand: Januar 2025*