---
name: quality-code-swiftui
description: "Pro" SwiftUI quality standards. Covers modern APIs, accessibility, performance, and Swift 6.2+ code hygiene.
trigger: "Implementing SwiftUI interfaces or reviewing code"
---

# SwiftUI Pro Quality Standards — Marino Framework

**Purpose:** To ensure that SwiftUI code is modern, accessible, and high-performance. This skill helps avoid common AI "blind spots" such as using deprecated APIs or neglecting accessibility.

## 1. Modern API Usage (Swift 6.2+)
*   **Colors & Shapes**: Always use `foregroundStyle()` and `background()` instead of the legacy `foregroundColor()` or `background(Color...)`.
*   **Navigation**: Use `NavigationStack` with `navigationDestination(for:destination:)`. Never use `NavigationView` (deprecated).
*   **Observation**: Use `@Bindable` when creating bindings to properties of `@Observable` objects.
*   **SF Symbols**: Use `symbolVariant()` and `symbolRenderingMode()` for advanced icon styling.

## 2. Accessibility-First (Mandatory)
Every interactive element MUST be "visible" to VoiceOver.
*   **Labels**: Every `Button` using only an icon MUST have an `accessibilityLabel`.
*   **Traits**: Add `.accessibilityAddTraits(.isHeader)` to logical section headers.
*   **Hints**: Provide an `.accessibilityHint()` for complex actions.
*   **Dynamic Type**: Always verify that layouts do not break with extra-large font sizes. Use `ViewThatFits` for responsive text strategies.

## 3. Performance & State Management
*   **Identity**: Use stable identifiers in `ForEach`. Avoid using `\.self` on strings or types that may change value but represent the same identity.
*   **Encapsulation**: Always mark `@State` properties as `private` to ensure they are local to the view.
*   **Logic Separation**: The view `body` should only contain UI declarations. All decision-making logic resides in the `@Observable` ViewModel.
*   **Task Management**: Use `.task` instead of `.onAppear` for asynchronous data loading.

## 4. Code Hygiene
*   **Omissions**: Use `_` if a closure parameter is not utilized.
*   **Layout**: Prefer semantic containers (`VStack`, `HStack`, `ZStack`) and strictly use the framework's `DesignTokens` for spacing and padding.
*   **Previews**: Every View MUST have a working `#Preview` using mock data from the `docs/mocks/` folder.
*   **Explicit Returns**: Avoid explicit `return` in single-expression view builders.

---
**Status:** ✅ Aligned with Apple HIG & Modern SwiftUI Standards (2026).
