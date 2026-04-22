---
name: metadata-localization
description: Multi-language support and App Store metadata localization. Use this skill when expanding to international markets, translating app content, or localizing App Store listings. Essential for reaching global audiences and improving discoverability in different regions.
compatibility: iOS 17+, String Catalogs
---

# Metadata Localization — String Catalogs & Dynamic AI

**When to use:** Before expanding to new markets. Use String Catalogs for static UI and AI Translation APIs for dynamic content.

## Pattern: String Catalogs (Xcode 26 Native)

String Catalogs (`.xcstrings`) replace legacy `.strings` files. They provide a visual editor, automatic pluralization, and string variation management.

```swift
// Usage in SwiftUI
struct LocalizedContentView: View {
    var body: some View {
        VStack(spacing: 16) {
            // Static localization from String Catalog
            Text("Welcome to our App", comment: "Main greeting")
                .font(.title)
            
            // Pluralization handled automatically by String Catalog
            // Setup "You have %lld tasks" in the .xcstrings editor
            Text("You have \(taskCount) tasks")
        }
    }
}
```

## Pattern: Dynamic AI Localization
*Translating user-generated or AI-generated content on-device.*

```swift
import Foundation
import Translation // iOS 18+ framework

@Observable
final class TranslationManager {
    
    @MainActor
    func translateContent(_ text: String, to targetLanguage: Locale.Language) async throws -> String {
        // This uses on-device Apple Intelligence for translation
        // No network required, high privacy.
        // Requires: import Translation
        
        // Placeholder for the iOS 18 Translation API implementation
        return "Translated: \(text)" 
    }
}
```

## App Store Metadata Template

```markdown
### English (en-US)
**App Name:** My App
**Subtitle:** Productivity for everyone
**Keywords:** productivity, tasks, management
**Description:** My App helps you manage tasks and stay organized...

### Italian (it)
**App Name:** La Mia App
**Subtitle:** Produttività per tutti
**Keywords:** produttività, attività, gestione
**Description:** La Mia App ti aiuta a gestire le attività...

### German (de)
**App Name:** Meine App
**Subtitle:** Produktivität für alle
**Keywords:** produktivität, aufgaben, verwaltung
**Description:** Meine App hilft dir, Aufgaben zu verwalten...

### French (fr)
**App Name:** Mon App
**Subtitle:** Productivité pour tous
**Keywords:** productivité, tâches, gestion
**Description:** Mon App vous aide à gérer vos tâches...

### Spanish (es)
**App Name:** Mi Aplicación
**Subtitle:** Productividad para todos
**Keywords:** productividad, tareas, gestión
**Description:** Mi Aplicación te ayuda a gestionar tareas...
```

## Implementation Checklist

- [ ] Set up String Catalog in Xcode (File → New → String Catalog)
- [ ] Add base English strings
- [ ] Enable localization for each target language
- [ ] Translate all user-facing strings
- [ ] Test with system language changes
- [ ] Verify pluralization rules per language
- [ ] Create App Store metadata for each region
- [ ] Test App Store search in each language
- [ ] Configure pricing per region
- [ ] Submit localized app metadata to App Store Connect
- [ ] Monitor download trends per region
- [ ] Gather user feedback for translation improvements
