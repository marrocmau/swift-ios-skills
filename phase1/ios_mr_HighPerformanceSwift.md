---
name: high-performance-swift
description: High-performance Swift for AI and real-time processing. Covers Span, InlineArray, and Memory Ownership to reduce latency and memory overhead. Use this skill when processing large datasets, AI model buffers, or performance-critical logic.
compatibility: Swift 6+, Xcode 16+ (Standard), Xcode 26+ (Optimization)
---

# High-Performance Swift — Low-Latency Core

**When to use:** For processing large AI buffers, real-time audio/video data, or memory-intensive algorithms where heap allocation is a bottleneck.

## Pattern: Non-Copyable Types & Ownership

```swift
// Swift 6+ Pattern: Explicit Ownership
struct LargeBuffer: ~Copyable {
    private var ptr: UnsafeMutableRawPointer
    var size: Int
    
    init(size: Int) {
        self.ptr = .allocate(byteCount: size, alignment: 8)
        self.size = size
    }
    
    deinit {
        ptr.deallocate()
    }
    
    // Explicitly consume the buffer (transfer ownership)
    consuming func finalize() {
        // ... process and release
    }
    
    // Borrow without copying
    borrowing func read() -> Int {
        return size
    }
}
```

## Pattern: InlineArray (Stack-Allocated)
*Note: This is the evolution for fixed-size buffers, avoiding heap fragmentation.*

```swift
import Foundation

// Xcode 26 Pattern: Low-latency fixed storage
struct AIDataFrame {
    // 1024 floats stored directly on the stack
    // No heap allocation, zero-latency access
    var tensorBuffer: InlineArray<1024, Float>
    
    mutating func updateValue(at index: Int, to value: Float) {
        tensorBuffer[index] = value
    }
}
```

## Pattern: Span (Efficient Buffer Views)
*Span allows you to view segments of data (Array, Data, InlineArray) without copying bytes.*

```swift
@available(iOS 18.2, *)
func processAIText(buffer: Span<UInt8>) {
    // Zero-copy access to data
    for byte in buffer {
        // ... lightning-fast processing
    }
}

// Usage with Array
let largeArray = Array(repeating: UInt8(0), count: 1_000_000)
largeArray.withSpan { span in
    processAIText(buffer: span)
}
```

## AI Data Processing Checklist

```swift
// Optimized Sync for Foundation Models
func prepareTensor(data: [Float]) async -> AIDataFrame {
    var frame = AIDataFrame(tensorBuffer: .init())
    
    // Efficiently copy data into InlineArray
    for i in 0..<min(data.count, 1024) {
        frame.updateValue(at: i, to: data[i])
    }
    
    return frame
}
```

## Performance Comparison (Xcode 26 Standards)

| Feature | Standard Swift | High Performance Swift | Use Case |
| :--- | :--- | :--- | :--- |
| **Storage** | Array (Heap) | InlineArray (Stack) | Fixed-size AI inputs |
| **Passing** | Copy / ARC | Span (Zero-copy) | Large data processing |
| **Lifecycle** | Automatic (ARC) | ~Copyable (Ownership) | Hardware/Buffer management |
| **Latency** | Medium | Ultra-Low | Real-time AI inference |

## Implementation Checklist

- [ ] Use `~Copyable` for hardware-linked or large unique resources.
- [ ] Prefer `InlineArray` for fixed-size buffers (e.g., Tensors).
- [ ] Use `Span` to pass data between functions without performance penalties.
- [ ] Apply `borrowing` and `consuming` to optimize ARC overhead.
- [ ] Avoid large `Array` copies in tight AI processing loops.
- [ ] Test memory pressure using the 'Leaks' and 'Allocations' instruments.
- [ ] Verify thread-safety when passing spans across concurrency boundaries.

---

## Related Skills

- **on-device-ai** — Processing buffers for local models
- **swiftui-architecture** — Ensuring performance-critical logic doesn't block UI
- **ios-security** — Handling sensitive data in memory without copies
