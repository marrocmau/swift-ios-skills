---
name: metadata-localization
description: Multi-language support and App Store metadata localization. Use this skill when expanding to international markets, translating app content, or localizing App Store listings. Essential for reaching global audiences and improving discoverability in different regions.
compatibility: iOS 17+, String Catalogs
---

# Metadata Localization — Multi-Language Support

**When to use:** Before expanding to new markets. Increases downloads 2-3x in localized regions.

## Pattern: String Catalogs & Localization

```swift
// LocalizationService.swift
import Foundation

struct LocalizationService {
    static let supportedLanguages = ["en", "it", "de", "fr", "es", "ja"]
    
    static func localizedString(_ key: String, in language: String = Locale.current.language.languageCode?.identifier ?? "en") -> String {
        let bundle = Bundle(path: Bundle.main.path(forResource: language, ofType: "lproj") ?? "") ?? .main
        return NSLocalizedString(key, bundle: bundle, comment: "")
    }
    
    static func setLanguage(_ languageCode: String) {
        UserDefaults.standard.set([languageCode], forKey: "AppleLanguages")
        UserDefaults.standard.synchronize()
    }
}

// Usage in View
struct LocalizedContentView: View {
    @State private var selectedLanguage = Locale.current.language.languageCode?.identifier ?? "en"
    
    var body: some View {
        VStack(spacing: 16) {
            // Automatic translation based on system language
            Text("app.title")
            Text("app.subtitle")
            
            // Manual language selection
            Picker("Language", selection: $selectedLanguage) {
                Text("English").tag("en")
                Text("Italiano").tag("it")
                Text("Deutsch").tag("de")
                Text("Français").tag("fr")
                Text("Español").tag("es")
                Text("日本語").tag("ja")
            }
            .onChange(of: selectedLanguage) { _, newLanguage in
                LocalizationService.setLanguage(newLanguage)
            }
        }
        .padding()
    }
}

// Localizable.strings structure
/*
 en.lproj/Localizable.strings:
 "app.title" = "My App";
 "app.subtitle" = "Productivity for everyone";
 
 it.lproj/Localizable.strings:
 "app.title" = "La Mia App";
 "app.subtitle" = "Produttività per tutti";
 
 de.lproj/Localizable.strings:
 "app.title" = "Meine App";
 "app.subtitle" = "Produktivität für alle";
*/
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
