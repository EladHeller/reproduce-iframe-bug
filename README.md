# Chrome IFrame Self-Removal Bug Reproduction

## 🔴 BUG CONFIRMED - Chrome 142.0.7444.60

This is a **confirmed, reproducible bug** in Chrome where:
- Event handler is created in the **iframe** using `onkeypress` property (not `addEventListener`)
- When pressing **any key** on a button inside the iframe, the parent removes the iframe
- **Chrome tab crashes with "Aw, Snap!"** (but mouse click works fine!)
- **Only happens on MacBook**

**Status:** Reproducible in Chrome 142 (latest version as of Nov 2025)  
**Severity:** High - Causes complete tab crash  
**Real-world impact:** Affects production systems with modal popups using legacy event handling

## Files

### Main Test Files (Start Here!)
- **`src/index.html`** - ⭐ PRIMARY TEST - Main page with iframe (100% crash rate)
- **`src/iframe.html`** - IFrame content with the bug pattern

**Live Demo:** https://eladheller.github.io/reproduce-iframe-bug/


## 🚀 Quick Start - Reproduce the Bug

### Option 1: Local Test

1. **Start local web server:**
   ```bash
   npx http-server ./src -p 8000
   ```

2. **Open the main test:**
   ```
   http://localhost:8000/
   ```

### Option 2: GitHub Pages (Hosted Demo)

Once deployed:
```
https://eladheller.github.io/reproduce-iframe-bug/
```

3. **Steps to crash Chrome:**
   - The page loads with an iframe containing a button (auto-focused)
   - Press **any key** (example limited to Enter/Space for demonstration)
   - 💥 **Chrome tab crashes with "Aw, Snap!"**

4. **Compare with mouse click:**
   - Reload the page
   - **Click** the OK button with mouse inside the iframe
   - ✓ Works fine!

**The bug only happens with keyboard (any key), not mouse clicks!**
**Note:** This bug is only reproducible on MacBook.

## Confirmed Bug Behavior

### ✓ Works Fine (No Crash)
- Clicking button with **mouse**
- Using `addEventListener` instead of `onclick` property
- Removing iframe with `setTimeout` delay
- Removing the e.preventDefault()

### 💥 Crashes Chrome Tab
- Pressing **any key** on button (Enter/Space used in example for demonstration)
- Using **`onkeypress` property** assignment (not addEventListener)
- Event handler created in the **iframe** (not using addEventListener)
- Handler calls parent window function which removes the iframe
- **Only on MacBook**

**Result:** "Aw, Snap!" - Chrome tab crashes completely

## Key Details (Critical for Reproduction)

### The Critical Pattern!

The bug requires **ALL** of these conditions:

1. **IFrame contains an html element** (button, for example)
2. **Button's `onkeypress` handler** (created in iframe, not using addEventListener) calls parent window function directly
3. **Parent function calls `iframe.remove()`** to remove the iframe
4. **Event is triggered by keyboard** (any key works, example uses Enter/Space), not mouse click
5. **Running on MacBook**

### The Critical Chain

**In iframe (src/iframe.html):**
```javascript
const btn = document.getElementById('btn');

// CRITICAL: onkeypress directly calls parent function
btn.onkeypress = function(e) {
    if (e.keyCode == 13 || e.charCode == 32) {
        window.parent.hide();  // ← Calls parent while keypress is active!
        return false; // or e.preventDefault()
    }
};

btn.focus();
```

**In parent (src/index.html):**
```javascript
const iframe = document.getElementById('iframe');
function hide() {
    iframe.remove();  // ← 💥 CRASH when called from keypress!
}
```

This specific combination causes Chrome to crash!

## Why This Happens

The crash occurs due to this sequence:

1. User presses **Enter** or **Space** on button in iframe
2. `btn.onkeypress` handler (in iframe) fires
3. Handler calls `window.parent.hide()` directly
4. Parent's `hide()` function calls `iframe.remove()` - **destroying the iframe document**
5. Control tries to return to the `onkeypress` handler
6. **The iframe document no longer exists** - Chrome crashes! 💥

The key is **cross-document event handling**: a keypress event handler in the iframe calls a parent function that destroys the iframe document while the event is still being processed.

**Why mouse click works:** Click events have different timing/propagation, allowing the iframe to be removed after the event completes.

