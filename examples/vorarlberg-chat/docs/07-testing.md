# 🧪 Testing Guide: Ländle Chatbot

## Testing-Strategie

### Testpyramide
```
         E2E Tests
        /         \
       /           \
    Integration Tests
   /                 \
  /                   \
Unit Tests (Foundation)
```

## Unit Tests

### Intent Detection Tests
```typescript
// src/services/__tests__/intentDetection.test.ts
describe('Intent Detection', () => {
  test.each([
    ['Wo kann ich gut essen?', IntentType.RESTAURANT],
    ['Schöne Wanderung gesucht', IntentType.HIKING],
    ['Museum in Bregenz', IntentType.LOCATION],
    ['Hotel für heute Nacht', IntentType.ACCOMMODATION]
  ])('detects intent for "%s"', async (query, expectedIntent) => {
    const result = await detectIntent(query)
    expect(result.type).toBe(expectedIntent)
  })
})
```

### Component Tests
```typescript
// src/components/__tests__/VoiceInput.test.tsx
import { render, fireEvent } from '@testing-library/react'
import { VoiceInput } from '../VoiceInput'

test('toggles listening state on click', () => {
  const onTranscript = jest.fn()
  const { getByRole } = render(
    <VoiceInput onTranscript={onTranscript} />
  )
  
  const button = getByRole('button')
  fireEvent.click(button)
  
  expect(button).toHaveAttribute('aria-pressed', 'true')
})
```

## Integration Tests

### API Integration
```typescript
// src/services/__tests__/answerPipeline.integration.test.ts
describe('Answer Pipeline Integration', () => {
  test('processes location query with database', async () => {
    const result = await processQuery(
      'Sehenswürdigkeiten in Feldkirch',
      mockEngine
    )
    
    expect(result.sources).toContainEqual(
      expect.objectContaining({ type: 'database' })
    )
    expect(result.answer).toContain('Schattenburg')
  })
})
```

## E2E Tests

### Cypress Setup
```typescript
// cypress/e2e/chat-flow.cy.ts
describe('Chat Flow', () => {
  it('completes a full conversation', () => {
    cy.visit('/')
    
    // Warte auf Modell-Laden
    cy.contains('Bereit', { timeout: 30000 })
    
    // Suggestion Card klicken
    cy.contains('Restaurants').click()
    
    // Antwort prüfen
    cy.get('.message.assistant')
      .should('contain', 'Restaurant')
    
    // Manuelle Eingabe
    cy.get('input[type="text"]')
      .type('Wandern im Montafon{enter}')
    
    cy.get('.message.assistant')
      .last()
      .should('contain', 'Montafon')
  })
})
```

## Performance Tests

### Lighthouse CI
```json
// lighthouserc.json
{
  "ci": {
    "assert": {
      "assertions": {
        "categories:performance": ["error", {"minScore": 0.9}],
        "categories:accessibility": ["error", {"minScore": 0.95}],
        "first-contentful-paint": ["error", {"maxNumericValue": 2000}],
        "interactive": ["error", {"maxNumericValue": 5000}]
      }
    }
  }
}
```

## Accessibility Tests

### axe-core Integration
```typescript
// src/utils/a11y.test.ts
import { axe } from 'jest-axe'

test('has no accessibility violations', async () => {
  const { container } = render(<App />)
  const results = await axe(container)
  expect(results).toHaveNoViolations()
})
```

## Load Testing

### K6 Script
```javascript
// k6/load-test.js
import http from 'k6/http'
import { check } from 'k6'

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 0 }
  ]
}

export default function() {
  const res = http.post('https://api.example.com/chat', {
    query: 'Restaurant in Bregenz'
  })
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 2000ms': (r) => r.timings.duration < 2000
  })
}
```

## Testing Checklist

### Pre-Deployment
- [ ] Unit Tests: >80% Coverage
- [ ] Integration Tests bestanden
- [ ] E2E Tests in Chrome/Firefox/Safari
- [ ] Mobile Tests (iOS/Android)
- [ ] Accessibility Audit
- [ ] Performance Budget eingehalten
- [ ] Security Scan durchgeführt

---
*Testing Version: 1.0 | Stand: Januar 2025*