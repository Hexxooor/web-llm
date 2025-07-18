# 🎨 UI/UX Design Guide: Ländle Chatbot

## Design-Prinzipien

### vol.at Design-DNA
- **Farben**: Gelb (#FFD200), Schwarz (#1A1603), Weiß (#FFFFFF)
- **Typografie**: Clean, Bold, Uppercase für Headlines
- **Layout**: Klar strukturiert, viel Whitespace
- **Interaktion**: Direkt, schnell, informativ

### Benutzerzentrierung
1. **Einfachheit**: Intuitive Bedienung ohne Erklärung
2. **Geschwindigkeit**: Sofortige Reaktion auf Eingaben
3. **Klarheit**: Eindeutige visuelle Hierarchie
4. **Zugänglichkeit**: WCAG 2.1 AA konform

## 🎨 Farbpalette

```css
:root {
  /* Primärfarben */
  --color-primary: #FFD200;      /* vol.at Gelb */
  --color-secondary: #1A1603;    /* Fast Schwarz */
  --color-accent: #E3051B;       /* Rot für Akzente */
  
  /* Neutralfarben */
  --color-bg-main: #F7F6F0;      /* Heller Hintergrund */
  --color-bg-white: #FFFFFF;     /* Weiß */
  --color-bg-gray: #F8F8F8;      /* Hellgrau */
  
  /* Textfarben */
  --color-text-primary: #1A1603; /* Haupttext */
  --color-text-secondary: #666;   /* Sekundärtext */
  --color-text-light: #999;      /* Deaktiviert */
  
  /* Statusfarben */
  --color-success: #4CAF50;      /* Erfolg */
  --color-error: #E3051B;        /* Fehler */
  --color-info: #2196F3;         /* Info */
  --color-warning: #FF9800;      /* Warnung */
  
  /* Schatten */
  --shadow-sm: 0 2px 8px rgba(0,0,0,0.05);
  --shadow-md: 0 4px 16px rgba(0,0,0,0.1);
  --shadow-lg: 0 8px 32px rgba(0,0,0,0.15);
}
```

## 📐 Layout-System

### Grid-System
```scss
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
  
  @media (max-width: 768px) {
    padding: 0 15px;
  }
}

.grid {
  display: grid;
  gap: 16px;
  
  &--2cols {
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  }
  
  &--3cols {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  }
}
```

### Spacing-System
```scss
$spacing: (
  xs: 4px,
  sm: 8px,
  md: 16px,
  lg: 24px,
  xl: 32px,
  xxl: 48px
);
```

## 🖼️ Komponenten-Design

### Chat-Nachrichten
```scss
.message {
  max-width: 80%;
  margin-bottom: 16px;
  
  &.user {
    margin-left: auto;
    
    .message-content {
      background: var(--color-primary);
      color: var(--color-secondary);
      border-radius: 12px 12px 4px 12px;
    }
  }
  
  &.assistant {
    margin-right: auto;
    
    .message-content {
      background: white;
      border: 1px solid #E0E0E0;
      border-radius: 12px 12px 12px 4px;
      box-shadow: var(--shadow-sm);
    }
  }
  
  &-content {
    padding: 16px 20px;
    font-size: 15px;
    line-height: 1.5;
  }
}
```

### Vorschlags-Karten
```scss
.suggestion-card {
  background: white;
  border: 2px solid #E0E0E0;
  border-radius: 8px;
  padding: 20px;
  cursor: pointer;
  transition: all 0.2s ease;
  position: relative;
  overflow: hidden;
  
  &:hover {
    border-color: var(--color-primary);
    transform: translateY(-2px);
    box-shadow: var(--shadow-md);
    
    &::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 4px;
      background: var(--color-primary);
    }
  }
  
  h3 {
    font-size: 16px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    margin-bottom: 8px;
  }
  
  p {
    font-size: 14px;
    color: var(--color-text-secondary);
    line-height: 1.4;
  }
}
```

### Input-Bereich
```scss
.chat-input {
  display: flex;
  gap: 12px;
  padding: 20px;
  background: white;
  border-top: 1px solid #E0E0E0;
  
  input {
    flex: 1;
    padding: 14px 18px;
    border: 2px solid #E0E0E0;
    border-radius: 8px;
    font-size: 16px;
    transition: border-color 0.3s;
    
    &:focus {
      outline: none;
      border-color: var(--color-primary);
    }
    
    &::placeholder {
      color: var(--color-text-light);
    }
  }
  
  button {
    padding: 14px 24px;
    background: var(--color-primary);
    color: var(--color-secondary);
    border: none;
    border-radius: 8px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    cursor: pointer;
    transition: all 0.2s;
    
    &:hover:not(:disabled) {
      background: #E6BE00;
      transform: translateY(-1px);
    }
    
    &:active {
      transform: translateY(0);
    }
    
    &:disabled {
      background: #CCC;
      cursor: not-allowed;
    }
  }
}
```

## 🎭 Animationen

### Basis-Animationen
```scss
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

@keyframes slideInRight {
  from { transform: translateX(20px); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}

// Anwendung
.message {
  animation: fadeIn 0.3s ease-out;
}

.suggestion-card {
  animation: fadeIn 0.4s ease-out backwards;
  
  @for $i from 1 through 6 {
    &:nth-child(#{$i}) {
      animation-delay: #{$i * 0.1}s;
    }
  }
}
```

### Ladeanimationen
```scss
.loading-dots {
  display: inline-flex;
  gap: 4px;
  
  span {
    width: 8px;
    height: 8px;
    background: var(--color-primary);
    border-radius: 50%;
    animation: bounce 1.4s infinite ease-in-out both;
    
    &:nth-child(1) { animation-delay: -0.32s; }
    &:nth-child(2) { animation-delay: -0.16s; }
  }
}

@keyframes bounce {
  0%, 80%, 100% { transform: scale(0); }
  40% { transform: scale(1); }
}
```

## 📱 Responsive Design

### Breakpoints
```scss
$breakpoints: (
  mobile: 480px,
  tablet: 768px,
  desktop: 1024px,
  wide: 1280px
);

@mixin respond-to($breakpoint) {
  @media (min-width: map-get($breakpoints, $breakpoint)) {
    @content;
  }
}
```

### Mobile-First Approach
```scss
.header {
  padding: 20px;
  font-size: 24px;
  
  @include respond-to(tablet) {
    padding: 30px;
    font-size: 32px;
  }
  
  @include respond-to(desktop) {
    font-size: 40px;
  }
}
```

## 🎯 Micro-Interactions

### Hover-Effekte
```scss
.interactive-element {
  transition: all 0.2s ease;
  
  &:hover {
    transform: translateY(-2px);
    box-shadow: var(--shadow-md);
  }
  
  &:active {
    transform: translateY(0);
    box-shadow: var(--shadow-sm);
  }
}
```

### Focus-States
```scss
.focusable {
  &:focus {
    outline: none;
    box-shadow: 0 0 0 3px rgba(255, 210, 0, 0.3);
  }
  
  &:focus-visible {
    outline: 2px solid var(--color-primary);
    outline-offset: 2px;
  }
}
```

## ♿ Barrierefreiheit

### ARIA-Labels
```html
<!-- Spracherkennung -->
<button 
  aria-label="Spracherkennung starten"
  aria-pressed={isListening}
  className="voice-button"
>
  {isListening ? '🎤' : '🎙️'}
</button>

<!-- Ladeindikator -->
<div 
  role="status" 
  aria-live="polite"
  aria-label="Nachricht wird geladen"
>
  <span className="loading-text">Assistant schreibt...</span>
</div>
```

### Keyboard Navigation
```scss
.keyboard-navigable {
  &:focus {
    outline: 2px solid var(--color-primary);
    outline-offset: 2px;
  }
  
  // Skip Links
  .skip-link {
    position: absolute;
    left: -9999px;
    
    &:focus {
      position: fixed;
      top: 10px;
      left: 10px;
      z-index: 999;
    }
  }
}
```

## 🌗 Dark Mode (Optional)

```scss
@media (prefers-color-scheme: dark) {
  :root {
    --color-bg-main: #1A1A1A;
    --color-bg-white: #2A2A2A;
    --color-text-primary: #F0F0F0;
    --color-text-secondary: #AAA;
    
    // Anpassungen für Kontrast
    --color-primary: #FFD200;
    --shadow-sm: 0 2px 8px rgba(0,0,0,0.3);
  }
  
  .message.assistant .message-content {
    background: var(--color-bg-white);
    border-color: #444;
  }
}
```

## 📏 Design-System Komponenten

### Buttons
```tsx
// Button-Varianten
<Button variant="primary">Senden</Button>
<Button variant="secondary">Abbrechen</Button>
<Button variant="text">Mehr anzeigen</Button>

// Größen
<Button size="small">Klein</Button>
<Button size="medium">Mittel</Button>
<Button size="large">Groß</Button>
```

### Badges
```tsx
<Badge type="success">Lokal</Badge>
<Badge type="info">Web</Badge>
<Badge type="warning">KI</Badge>
```

### Cards
```tsx
<Card hoverable onClick={handleClick}>
  <Card.Header>
    <h3>Wanderung</h3>
  </Card.Header>
  <Card.Body>
    <p>Schöne Bergtouren im Montafon</p>
  </Card.Body>
</Card>
```

## 🎯 Best Practices

1. **Performance**
   - CSS-in-JS vermeiden für kritische Styles
   - Animation mit `transform` und `opacity`
   - Lazy Loading für Bilder

2. **Konsistenz**
   - Design Tokens verwenden
   - Komponenten-Library pflegen
   - Style Guide dokumentieren

3. **Usability**
   - Touch-Targets min. 44x44px
   - Kontrast-Verhältnis min. 4.5:1
   - Fehler klar kommunizieren

4. **Mobile**
   - Viewport-Meta-Tag setzen
   - Tap-Delays vermeiden
   - Swipe-Gesten unterstützen

---
*UI/UX Design Version: 1.0 | Stand: Januar 2025*