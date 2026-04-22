---
name: accessibility-advanced
description: Advanced accessibility features and VoiceOver support. Use this skill when optimizing for screen readers, building accessible custom components, or ensuring WCAG 2.1 AA compliance. Essential for inclusive app design and expanding addressable market.
compatibility: iOS 15+, Accessibility framework
---

# Accessibility — Advanced Features & VoiceOver

**When to use:** Throughout development. Accessibility benefits all users, not just those with disabilities.

## Pattern: Accessible SwiftUI Components

```swift
import SwiftUI
import Accessibility

// MARK: - Accessible Custom Button (AI-Ready)
struct AccessibleButton: View {
    let title: String
    let action: () -> Void
    let hint: String?
    
    var body: some View {
        Button(action: action) {
            Text(title)
                .padding()
                .frame(maxWidth: .infinity)
                .background(Color.blue)
                .foregroundStyle(.white)
                .cornerRadius(8)
        }
        .accessibilityLabel(title)
        .accessibilityHint(hint ?? "")
        .accessibilityAddTraits(.isButton)
        // Xcode 26: Signal to Apple Intelligence that this is a key action
        .accessibilityRespondsToUserInteraction(true)
    }
}

// MARK: - Semantic UI for Visual Intelligence
struct AIFriendlyImageView: View {
    let imageName: String
    let semanticType: String // e.g., "Product", "User Avatar", "Action Icon"
    
    var body: some View {
        Image(imageName)
            .resizable()
            .scaledToFit()
            // Xcode 26: Provide semantic context for Visual Intelligence
            .accessibilityLabel(semanticType)
            .accessibilityAddTraits(.isImage)
            .accessibilityHeaderElements([/* Sub-elements for AI to scan */])
    }
}

// MARK: - Accessible List Item (Semantic)
struct AccessibleListItem: View {
    let title: String
    let subtitle: String?
    let isSelected: Bool
    
    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(title)
                    .font(.headline)
                
                if let subtitle = subtitle {
                    Text(subtitle)
                        .font(.caption)
                        .foregroundStyle(.gray)
                }
            }
            
            Spacer()
            
            if isSelected {
                Image(systemName: "checkmark.circle.fill")
                    .foregroundStyle(.green)
                    .accessibilityLabel("Selected")
            }
        }
        .padding()
        .contentShape(Rectangle())
        // Combine for VoiceOver, but keep semantic separation for AI navigation
        .accessibilityElement(children: .combine)
        .accessibilityValue(isSelected ? "Active" : "Inactive")
        .accessibilityAddTraits(isSelected ? .isSelected : [])
    }
}

// MARK: - Accessible Form Field
struct AccessibleTextField: View {
    let label: String
    let hint: String
    @Binding var text: String
    
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(label)
                .font(.headline)
                .accessibilityLabel("Label: \(label)")
            
            TextField("", text: $text)
                .textFieldStyle(.roundedBorder)
                .accessibilityLabel(label)
                .accessibilityHint(hint)
                .accessibilityValue(text)
        }
        .accessibilityElement(children: .combine)
    }
}

// MARK: - VoiceOver Rotor
struct AccessibleContentWithRotor: View {
    @State private var selectedIndex = 0
    
    let items = ["Item 1", "Item 2", "Item 3"]
    
    var body: some View {
        VStack(spacing: 16) {
            Text("Accessible List")
                .font(.title)
                .accessibilityAddTraits(.isHeader)
            
            List {
                ForEach(items.indices, id: \.self) { index in
                    Text(items[index])
                        .accessibilityIdentifier("item-\(index)")
                        .accessibilityLabel("Item \(index + 1) of \(items.count)")
                }
            }
        }
        .accessibilityRotor("Items") {
            AccessibilityRotorEntry("Item 1", id: 0)
            AccessibilityRotorEntry("Item 2", id: 1)
            AccessibilityRotorEntry("Item 3", id: 2)
        }
    }
}

// MARK: - Accessible Image View
struct AccessibleImageView: View {
    let imageName: String
    let contentDescription: String
    
    var body: some View {
        Image(imageName)
            .resizable()
            .scaledToFit()
            .accessibilityLabel("Image")
            .accessibilityValue(contentDescription)
            // For decorative images only:
            // .accessibilityHidden(true)
    }
}

// MARK: - Dynamic Type Support
struct DynamicTypeExample: View {
    @Environment(\.dynamicTypeSize) var dynamicTypeSize
    
    var body: some View {
        VStack(spacing: 16) {
            Text("Title")
                .font(.system(.headline, design: .default))
            
            Text("Body text that adapts to user's preferred text size")
                .font(.system(.body, design: .default))
                .lineLimit(nil)
                .dynamicTypeSize(...DynamicTypeSize.xxxLarge)
            
            // Custom scaling for specific sizes
            if dynamicTypeSize >= .xxxLarge {
                Text("Extra large text visible")
            }
        }
        .padding()
    }
}

// MARK: - Accessible Chart/Graph
struct AccessibleChartView: View {
    let data: [ChartValue]
    
    struct ChartValue {
        let label: String
        let value: Double
    }
    
    var body: some View {
        VStack(spacing: 12) {
            Text("Chart Title")
                .font(.headline)
                .accessibilityAddTraits(.isHeader)
            
            // Visual representation
            HStack(spacing: 8) {
                ForEach(data, id: \.label) { item in
                    VStack {
                        RoundedRectangle(cornerRadius: 4)
                            .frame(height: CGFloat(item.value * 50))
                            .foregroundStyle(.blue)
                            .accessibilityValue("\(Int(item.value))")
                        
                        Text(item.label)
                            .font(.caption)
                    }
                }
            }
            
            // Accessible text alternative
            VStack(alignment: .leading, spacing: 4) {
                Text("Data Summary")
                    .font(.headline)
                
                ForEach(data, id: \.label) { item in
                    Text("\(item.label): \(Int(item.value))")
                        .font(.caption)
                }
            }
            .accessibilityElement(children: .combine)
        }
        .padding()
    }
}

// MARK: - Focus Management
struct AccessibleFormView: View {
    @State private var email = ""
    @State private var password = ""
    @FocusState private var focusedField: Field?
    
    enum Field {
        case email
        case password
    }
    
    var body: some View {
        VStack(spacing: 16) {
            TextField("Email", text: $email)
                .focused($focusedField, equals: .email)
                .textFieldStyle(.roundedBorder)
                .accessibilityLabel("Email address")
            
            SecureField("Password", text: $password)
                .focused($focusedField, equals: .password)
                .textFieldStyle(.roundedBorder)
                .accessibilityLabel("Password")
            
            Button("Login") {
                // Submit form
            }
            .accessibilityLabel("Login button")
        }
        .onAppear {
            focusedField = .email
        }
        .padding()
    }
}
```

## Accessibility Checklist

```
Color & Contrast:
- ✅ Minimum 4.5:1 contrast for text
- ✅ Don't rely on color alone for information
- ✅ Test with accessibility color filters

Touch & Interaction:
- ✅ Minimum 44pt touch targets
- ✅ No hover-only interactions
- ✅ Support full keyboard navigation
- ✅ Avoid rapid flashing (>3x per second)

VoiceOver:
- ✅ All interactive elements labeled
- ✅ Images have descriptions
- ✅ Logical reading order
- ✅ Custom rotors for navigation

Text & Type:
- ✅ Support dynamic type scaling
- ✅ Readable font size (minimum 16pt body)
- ✅ Adequate line spacing
- ✅ Avoid long reading levels

Audio & Video:
- ✅ Captions for all videos
- ✅ Audio descriptions
- ✅ Transcripts available
- ✅ Don't auto-play sound

Forms & Input:
- ✅ Clear labels for form fields
- ✅ Error messages are descriptive
- ✅ Focus management implemented
- ✅ Validation happens on input

Navigation:
- ✅ Clear structure and headings
- ✅ Skip links for repetitive content
- ✅ Consistent navigation patterns
- ✅ Breadcrumb trails
```

## Testing Tools

```swift
// Enable VoiceOver in iOS Settings
// Settings → Accessibility → VoiceOver → On

// Test with different text sizes
// Settings → Accessibility → Display → Text Size

// Check contrast
// Accessibility Inspector in Xcode
// Window → Developer → Accessibility Inspector

// Keyboard navigation
// Settings → Accessibility → Keyboard → Full Keyboard Access
```

## Implementation Checklist

- [ ] Add accessibility labels to all UI elements
- [ ] Add accessibility hints for complex interactions
- [ ] Implement keyboard navigation
- [ ] Support full VoiceOver operation
- [ ] Test minimum 44pt touch targets
- [ ] Verify text contrast (4.5:1)
- [ ] Support dynamic type sizing
- [ ] Add accessibility identifiers
- [ ] Create custom rotors for navigation
- [ ] Test with accessibility inspector
- [ ] Run accessibility audit in Xcode
- [ ] Document accessibility features
- [ ] Get user feedback from users with disabilities
