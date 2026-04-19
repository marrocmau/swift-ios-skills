---
name: aso-keyword-research
description: App Store Optimization and keyword strategy. Use this skill when researching keywords for discoverability, optimizing app metadata, analyzing competitor keywords, or planning ASO strategy. Essential for organic downloads and app store visibility.
compatibility: iOS 17+
---

# ASO Keyword Research — App Store Optimization

**When to use:** Before launching version 1.0 and regularly after each update.

## Pattern: Keyword Strategy

```swift
struct ASOKeywordData {
    let keyword: String
    let searchVolume: Int
    let difficulty: Double  // 0-1, where 1 = hardest
    let monthlySearches: Int
    
    var opportunity: Double {
        Double(monthlySearches) * (1.0 - difficulty)
    }
    
    var recommendedPosition: String {
        switch opportunity {
        case 1000...:
            return "Include in app name"
        case 500..<1000:
            return "Include in subtitle"
        default:
            return "Include in keywords field"
        }
    }
}

// Example: Italian productivity app keywords
let italianKeywords = [
    ASOKeywordData(keyword: "gestione progetti", searchVolume: 12000, difficulty: 0.8, monthlySearches: 12000),
    ASOKeywordData(keyword: "task manager", searchVolume: 8000, difficulty: 0.7, monthlySearches: 8000),
    ASOKeywordData(keyword: "app produttività", searchVolume: 5000, difficulty: 0.6, monthlySearches: 5000),
    ASOKeywordData(keyword: "lista attività", searchVolume: 3000, difficulty: 0.5, monthlySearches: 3000),
    ASOKeywordData(keyword: "organizzazione", searchVolume: 2000, difficulty: 0.4, monthlySearches: 2000),
]

// Analyze keywords
let sortedByOpportunity = italianKeywords.sorted { $0.opportunity > $1.opportunity }

// Get top opportunities
let topKeywords = sortedByOpportunity.prefix(3)
    .map { $0.keyword }
    .joined(separator: ", ")
```

## ASO Strategy Template

```markdown
# ASO Strategy Document

## 1. Target Keywords (3-5 primary)
- Primary: "gestione progetti" (opportunity: 9,600)
- Secondary: "task manager" (opportunity: 5,600)
- Long-tail: "app produttività" (opportunity: 3,000)

## 2. App Name
Include primary keyword if opportunity > 1000:
Example: "Task Pro - Gestione Progetti"

## 3. Subtitle (30 characters max)
Include secondary keyword:
Example: "Il miglior task manager italiano"

## 4. Keywords Field
Comma-separated long-tail keywords (100 characters):
"lista attività, organizzazione, progetti, reminder, todo"

## 5. Description (4000 characters)
- Lead with primary keyword naturally
- Highlight unique value
- Include 2-3 secondary keywords
- Natural language (not keyword stuffing)
- Clear benefits and features

## 6. Screenshots
- Show key features
- Use text overlays with benefits
- Keep consistent with keywords theme
- Test different messaging with A/B testing

## 7. Preview Video (15-30 seconds)
- Demonstrate core functionality
- Include keyword in voiceover
- Show user benefits clearly

## 8. Category Selection
Choose most relevant:
- Productivity
- Business
- Utilities

## 9. Competitor Analysis
- Analyze top 10 ranking apps
- Identify keyword gaps
- Find positioning opportunities

## 10. Monitoring & Iteration
- Track keyword rankings weekly
- Monitor download trends
- Test new keywords quarterly
- Analyze search term reports
```

## Implementation Checklist

- [ ] Use App Annie or Sensor Tower for keyword research
- [ ] Identify primary (high volume) keywords
- [ ] Find secondary keywords (medium opportunity)
- [ ] Research long-tail keywords (low competition)
- [ ] Analyze competitor keywords
- [ ] Calculate opportunity score for each keyword
- [ ] Select top 3-5 keywords for positioning
- [ ] Optimize app name with primary keyword
- [ ] Write subtitle with secondary keyword
- [ ] Fill keywords field (100 character limit)
- [ ] Write natural description incorporating keywords
- [ ] Create 5-6 marketing screenshots
- [ ] Record preview video
- [ ] Set correct app category
- [ ] Monitor keyword rankings weekly
- [ ] Plan quarterly ASO updates
