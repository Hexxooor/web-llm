# 🚀 Agent Quick Reference - Vorarlberg Chat MVP

## 📍 Git Workflow für Agents

### 1. Branch wechseln
```bash
# Zum eigenen Branch wechseln
git checkout agent/frontend     # für Agent-1
git checkout agent/ai-integration # für Agent-2
git checkout agent/data-cache    # für Agent-3
git checkout agent/styling       # für Agent-4

# Aktuellen Branch anzeigen
git branch --show-current
```

### 2. Änderungen committen
```bash
# Status prüfen
git status

# Dateien hinzufügen
git add .
# ODER spezifische Dateien
git add src/components/Chat.tsx

# Commit mit Agent-Tag
git commit -m "[AGENT-1] Chat-Komponente implementiert"

# Push zum Remote
git push origin agent/frontend
```

### 3. Updates vom main holen
```bash
# Auf main wechseln
git checkout main

# Updates holen
git pull origin main

# Zurück zum eigenen Branch
git checkout agent/frontend

# Main-Updates mergen
git merge main
```

## 📂 Erlaubte Arbeitsbereiche

### Agent-1 (Frontend)
✅ **Erlaubt:**
- `src/components/**`
- `src/pages/**`
- `src/hooks/**`
- `src/types/components.ts`

❌ **Nicht erlaubt:**
- `src/services/**`
- `src/styles/**`
- `src/data/**`

### Agent-2 (AI Integration)
✅ **Erlaubt:**
- `src/services/webllm/**`
- `src/services/intent/**`
- `src/config/ai/**`
- `src/types/ai.ts`

❌ **Nicht erlaubt:**
- `src/components/**`
- `src/styles/**`
- `src/data/**`

### Agent-3 (Data & Cache)
✅ **Erlaubt:**
- `src/services/cache/**`
- `src/data/**`
- `src/utils/storage/**`
- `src/types/data.ts`

❌ **Nicht erlaubt:**
- `src/components/**`
- `src/services/webllm/**`
- `src/styles/**`

### Agent-4 (Styling)
✅ **Erlaubt:**
- `src/styles/**`
- `public/assets/**`
- `src/components/**/*.css`
- `src/components/**/*.module.css`

❌ **Nicht erlaubt:**
- `src/services/**`
- `src/data/**`
- `src/hooks/**`

## 🔄 Status Updates

### In AGENT-COORDINATION.md updaten:
```markdown
### [2025-07-18] - Agent-1
- **Erledigt:** Vite Setup, ChatContainer erstellt
- **In Arbeit:** MessageList Komponente
- **Blocker:** Warte auf Data Types von Agent-3
- **Nächste Schritte:** Input Area implementieren
```

### In GitHub Issue kommentieren:
```
Status Update:
- [x] Vite Setup abgeschlossen
- [x] ChatContainer implementiert
- [ ] MessageList in Arbeit
- [ ] Blockiert: Data Types needed
```

## 🚨 Bei Problemen

### Merge-Konflikt
```bash
# Nicht panisch werden!
# 1. Konflikt-Dateien anschauen
git status

# 2. Dateien öffnen und <<<< >>>> Markierungen finden
# 3. Entscheiden welche Version behalten
# 4. Markierungen entfernen

# 5. Als gelöst markieren
git add [konflikt-datei]
git commit -m "[AGENT-X] Merge-Konflikt gelöst"
```

### Versehentlich falschen Branch geändert
```bash
# Änderungen stashen
git stash

# Zum richtigen Branch wechseln
git checkout agent/[dein-branch]

# Änderungen anwenden
git stash pop
```

## 📞 Kommunikation

### Blocker melden
1. Issue kommentieren mit `[BLOCKED]` Tag
2. In AGENT-COORDINATION.md dokumentieren
3. Alternative Aufgabe suchen

### Dependencies anfordern
```typescript
// In src/types/shared.ts dokumentieren
export interface SharedMessageType {
  // Agent-1 needs this
  id: string
  content: string
  role: 'user' | 'assistant'
  
  // Agent-2 adds this
  intent?: string
  confidence?: number
}
```

## 🎯 Täglicher Workflow

1. **Morgens:**
   - `git pull origin main`
   - GitHub Issues checken
   - AGENT-COORDINATION.md lesen

2. **Während der Arbeit:**
   - Kleine, häufige Commits
   - Bei Blockern sofort melden
   - Tests schreiben

3. **Abends:**
   - Alle Änderungen pushen
   - Status Update in Issue
   - AGENT-COORDINATION.md updaten

## 🛠️ Nützliche Commands

```bash
# Branch-Übersicht
git branch -a

# Letzte Commits anzeigen
git log --oneline -5

# Änderungen anzeigen
git diff

# Uncommitted Changes verwerfen
git checkout -- .

# Zum letzten Commit zurück
git reset --hard HEAD

# Wer hat was geändert
git blame [datei]
```

## 📋 Checkliste vor Push

- [ ] Code läuft ohne Fehler
- [ ] TypeScript Errors behoben
- [ ] Nur erlaubte Dateien geändert
- [ ] Sinnvolle Commit-Message
- [ ] Tests geschrieben/aktualisiert

---

**Bei Fragen:** Issue kommentieren oder in AGENT-COORDINATION.md nachschauen! 🤝
