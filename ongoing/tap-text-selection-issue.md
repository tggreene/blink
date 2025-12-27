# Tap Text Selection Issue

## Symptom
When selecting text with double-tap (word) or triple-tap (line):
1. Selection appears correctly
2. Small delay (~200-500ms)
3. Terminal appears to "refresh" and selection is lost

## Investigation

### Previous Attempt (6817c354)
Added a dummy `_doubleTapRecognizer` that single-tap must wait for via `require(toFail:)`.
This prevents single-tap from firing on the first tap of a double-tap, but **the issue persists**.

### Root Cause Analysis

The selection is being cleared by `cleanSelection()` called from `reportStateReset()`:

```swift
// SmarterTermInput.swift:240-243
func reportStateReset() {
  reportStateReset(false)
  device?.view?.cleanSelection()  // <-- This kills the selection!
}
```

This gets triggered by the `device` property's `didSet`:

```swift
// SmarterTermInput.swift:97-99
weak var device: TermDevice? = nil {
  didSet { reportStateReset() }
}
```

### Call Path

1. Single-tap fires `_on1fTap` → `focusOnShellAction()` (WKWebView.swift:249-251)
2. `focusOnShellAction()` → `_focusOnShell()` → `_attachInputToCurrentTerm()` (SpaceController.swift:768-770)
3. `_attachInputToCurrentTerm()` → `device.attachInput(deviceView.webView)` (SpaceController.swift:487)
4. `attachInput:` sets `_input.device = self` (TermDevice.m:405)
5. This triggers `didSet` on `device` property
6. `reportStateReset()` calls `cleanSelection()` → **Selection gone!**

### Why Previous Fix Didn't Work

Even though we added `_1fTapRecognizer.require(toFail: _doubleTapRecognizer)`:
- The double-tap recognizer has no action (dummy)
- Single-tap still fires after double-tap recognizer times out (no more taps)
- BUT: `hasSelection` should disable `_1fTapRecognizer` before this happens

Possible issues:
1. Race condition: selection message arrives after single-tap already started recognizing
2. Swift's `didSet` fires even when setting same value (no oldValue check)
3. `shouldRecognizeSimultaneouslyWith` returns `true` for all recognizers

### Key Files

- `Blink/WebKit/WKWebView.swift` - Gesture recognizers
- `Blink/SmarterKeys/SmarterTermInput.swift` - reportStateReset(), device property
- `Blink/TermDevice.m` - attachInput:
- `Blink/SpaceController.swift` - focusOnShellAction(), _attachInputToCurrentTerm()
- `Resources/term.js` - selectionchange event handler

## Proposed Fixes

### Fix 1: Guard device didSet
Only call `reportStateReset()` when device actually changes:
```swift
weak var device: TermDevice? = nil {
  didSet {
    guard device !== oldValue else { return }
    reportStateReset()
  }
}
```

### Fix 2: Don't fire single-tap when selection active
Check `hasSelection` in the 1f tap handler before calling focusOnShellAction:
```swift
@objc func _on1fTap(_ recognizer: UITapGestureRecognizer) {
  // ... existing code ...
  case .recognized:
    if hasSelection { return }  // <-- Add this guard
    // ... rest of handler
}
```

### Fix 3: Separate state reset from selection clearing
Maybe `reportStateReset()` shouldn't always clean selection:
```swift
func reportStateReset() {
  reportStateReset(false)
  // Don't call cleanSelection() here
}
```

## Status
- [x] Fix implemented (two-pronged approach)
- [ ] Fix tested
- [ ] Committed

## Changes Made

### WKWebView.swift (line 245-246)
Added guard to prevent `_on1fTap` from calling `focusOnShellAction` when selection is active:
```swift
// Don't interfere when there's an active selection
if hasSelection { return }
```

### SmarterTermInput.swift (lines 97-102)
Added guard to prevent `reportStateReset()` from firing when device is re-set to same value:
```swift
weak var device: TermDevice? = nil {
  didSet {
    guard device !== oldValue else { return }
    reportStateReset()
  }
}
```
