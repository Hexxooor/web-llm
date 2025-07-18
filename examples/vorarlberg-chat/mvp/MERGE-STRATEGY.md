# 🔄 Merge-Strategie - Vorarlberg Chat MVP

## 📅 Merge-Schedule

### Daily Merges (Täglich 18:00)
Jeden Tag werden die Branches in dieser Reihenfolge gemerged:

1. **agent/data-cache** → main
2. **agent/ai-integration** → main  
3. **agent/frontend** → main
4. **agent/styling** → main
5. **main** → alle agent/* branches (Rück-Sync)

### Integration Branch
Zusätzlich gibt es einen `integration` Branch für kontinuierliche Tests:
- Alle Agent-Branches mergen automatisch nach `integration`
- Tägliche Tests laufen auf `integration`
- Wenn stabil → `integration` wird nach `main` gemerged

## 🔀 Merge-Workflow

### Für Agents (Automatisch)
```bash
# Agent pusht seine Arbeit
git add .
git commit -m "[AGENT-X] Feature implementiert"
git push origin agent/[branch]

# GitHub Action merged automatisch nach integration
```

### Für Maintainer (Manuell)
```bash
# 1. Integration Branch testen
git checkout integration
npm test
npm run build

# 2. Wenn alles OK, merge nach main
git checkout main
git merge integration --no-ff

# 3. Tag für stabilen Release
git tag v0.1.0-daily-$(date +%Y%m%d)
git push origin main --tags
```

## 🚦 Merge-Regeln

### Automatische Merges erlaubt wenn:
- ✅ Alle Tests grün
- ✅ Build erfolgreich
- ✅ Keine Konflikte
- ✅ Code Review approved (optional)

### Merge blockiert wenn:
- ❌ Tests fehlschlagen
- ❌ Build Errors
- ❌ Merge-Konflikte
- ❌ Außerhalb der erlaubten Bereiche

## 📊 Status-Board

| Branch | Letzter Merge | Status | Konflikte |
|--------|---------------|---------|-----------|
| agent/frontend | - | 🟡 Initial | Keine |
| agent/ai-integration | - | 🟡 Initial | Keine |
| agent/data-cache | - | 🟡 Initial | Keine |
| agent/styling | - | 🟡 Initial | Keine |
| integration | - | 🟡 Initial | - |

## 🔧 Konflikt-Resolution

### Bei Merge-Konflikten:
1. **Automatische Resolution** für simple Konflikte
2. **Maintainer-Review** für komplexe Konflikte
3. **Agent-Kommunikation** über Issues

### Konflikt-Prioritäten:
```
1. Shared Types (src/types/shared.ts) - Maintainer entscheidet
2. Package.json - Alle Dependencies behalten
3. Component Interfaces - Agent-1 hat Vorrang
4. Styles - Agent-4 hat Vorrang
```

## 📈 Merge-Frequenz

### Phase 1: Setup (Tag 1-2)
- Stündliche Merges nach `integration`
- Abends Review und Merge nach `main`

### Phase 2: Development (Tag 3-7)
- Alle 2 Stunden nach `integration`
- 2x täglich nach `main` (12:00, 18:00)

### Phase 3: Finalization (Tag 8)
- Kontinuierliche Integration
- Feature-Freeze um 15:00
- Final Release um 18:00

## 🤖 GitHub Actions Setup

```yaml
name: Auto-Merge to Integration
on:
  push:
    branches: ['agent/*']

jobs:
  merge:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
          
      - name: Configure Git
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          
      - name: Merge to Integration
        run: |
          git checkout integration
          git merge ${{ github.ref_name }} --no-ff -m "Auto-merge ${{ github.ref_name }}"
          git push origin integration
```

## 📝 Merge-Protokoll

### Template für Merge-Commits:
```
[MERGE] agent/[branch] → [target]

Included changes:
- Feature X implemented
- Bug Y fixed
- Performance improvement Z

Tests: ✅ Passing
Build: ✅ Success
Conflicts: None
```

---

**Nächster geplanter Merge:** Heute 18:00 Uhr
