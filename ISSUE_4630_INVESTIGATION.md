# Investigation Report: Issue #4630 - macOS System Tray Menu Real-Time Updates

## Issue Summary

**Issue:** [#4630](https://github.com/wailsapp/wails/issues/4630) - "[v3] How to update the menu label in real time when the tray is open?"

**Problem:** When calling `menu.Update()` on a system tray menu in Wails v3 on macOS, changes to menu item labels do not appear in real-time while the menu is open. The updates only become visible after closing and reopening the menu.

**Platform:** macOS (v3-alpha branch)

## Root Cause Analysis

### 1. Current Implementation

#### macOS System Tray Menu Flow

1. **Menu Setup** (v3/pkg/application/systemtray_darwin.go:166-171)
   - When the system tray is initialized via `Run()`, it calls `s.menu.Update()`
   - This creates the NSMenu object and populates it with menu items

2. **Menu Display** (v3/pkg/application/systemtray_darwin.go:268, 280)
   - When the user clicks the tray icon, `processClick()` is called
   - This calls `C.showMenu(s.nsStatusItem, s.nsMenu)`
   - The C function `showMenu()` uses `popUpStatusItemMenu:` to display the menu (v3/pkg/application/systemtray_darwin.m:143-161)

3. **Menu Update** (v3/pkg/application/menu_darwin.go:80-89)
   - When `Menu.Update()` is called, it:
     - Clears the existing NSMenu using `C.clearMenu(m.nsMenu)`
     - Recreates all menu items by calling `m.processMenu()`
   - However, this only updates the NSMenu object itself

#### The Problem

The macOS implementation uses `popUpStatusItemMenu:` which displays a **snapshot** of the menu at the time it's shown. After the menu is displayed:

- Changes to the underlying NSMenu object do **not** automatically reflect in the already-displayed menu
- The displayed menu is essentially a static view until it's closed and reopened
- No mechanism exists to notify macOS that the menu needs to be refreshed while open

### 2. Linux Implementation (Working Correctly)

In contrast, Linux's system tray implementation in v3 has a working refresh mechanism:

**Key File:** v3/pkg/application/systemtray_linux.go

1. **refresh() method** (lines 178-193)
   ```go
   func (s *linuxSystemTray) refresh() {
       s.menuVersion++
       if err := s.menuProps.Set("com.canonical.dbusmenu", "Version",
           dbus.MakeVariant(s.menuVersion)); err != nil {
           // error handling
       }
       if err := menu.Emit(s.conn, &menu.Dbusmenu_LayoutUpdatedSignal{
           Path: menuPath,
           Body: &menu.Dbusmenu_LayoutUpdatedSignalBody{
               Revision: s.menuVersion,
           },
       }); err != nil {
           // error handling
       }
   }
   ```

2. **Called in setMenu()** (line 225)
   - After processing menu items, `s.refresh()` is called
   - This notifies DBus about the menu update

3. **Called in update()** (lines 587-590)
   - When individual menu items change, `refresh()` is called
   - The desktop environment receives the signal and updates the displayed menu

**Why it works:** Linux uses DBus to communicate with the desktop environment. The `LayoutUpdatedSignal` tells the system "the menu has changed, please refresh the display."

### 3. Missing macOS Solution

#### What's Missing

The macOS implementation lacks:

1. **NSMenuDelegate implementation** - No delegate is set for the NSMenu objects
2. **menuNeedsUpdate: callback** - This NSMenuDelegate method is not implemented
3. **Menu refresh mechanism** - No way to tell the displayed menu to refresh

#### How Native macOS Menus Work

Native macOS system menus (WiFi, Bluetooth, Battery) that update in real-time use:

1. **NSMenuDelegate protocol**
   - Implement the `menuNeedsUpdate:` method
   - This method is called automatically by macOS when the menu is about to be displayed
   - It's also called periodically while the menu remains open (if needed)

2. **Menu item updates**
   - Menu items can be updated programmatically
   - If the menu has a delegate, changes can trigger a refresh

## Technical Details

### Files Analyzed

**Core System Tray Files:**
- v3/pkg/application/systemtray.go - Cross-platform interface
- v3/pkg/application/systemtray_darwin.go - macOS Go implementation
- v3/pkg/application/systemtray_darwin.h - macOS C header
- v3/pkg/application/systemtray_darwin.m - macOS Objective-C implementation
- v3/pkg/application/systemtray_linux.go - Linux implementation (reference)

**Menu Files:**
- v3/pkg/application/menu.go - Cross-platform menu interface
- v3/pkg/application/menu_darwin.go - macOS menu implementation

### Key Functions

**v3/pkg/application/systemtray_darwin.m:143-161** - `showMenu()`
```objective-c
void showMenu(void* nsStatusItem, void *nsMenu) {
    dispatch_async(dispatch_get_main_queue(), ^{
        NSStatusItem *statusItem = (NSStatusItem *)nsStatusItem;
        [statusItem popUpStatusItemMenu:(NSMenu *)nsMenu];
        // ... cleanup code
    });
}
```

**v3/pkg/application/menu_darwin.go:80-89** - `macosMenu.update()`
```go
func (m *macosMenu) update() {
    InvokeSync(func() {
        if m.nsMenu == nil {
            m.nsMenu = C.createNSMenu(C.CString(m.menu.label))
        } else {
            C.clearMenu(m.nsMenu)
        }
        m.processMenu(m.nsMenu, m.menu)
    })
}
```

## Proposed Solutions

### Option 1: Implement NSMenuDelegate (Recommended)

**Approach:** Add NSMenuDelegate support to enable menu updates while the menu is open.

**Implementation Steps:**

1. Create a custom NSMenu delegate class in Objective-C
2. Implement the `menuNeedsUpdate:` method
3. Store a reference to the Go menu object in the delegate
4. When `menuNeedsUpdate:` is called, trigger a menu refresh
5. Update menu items based on the current state

**Pros:**
- Follows macOS best practices
- Enables true real-time updates
- Efficient - only updates when needed

**Cons:**
- Requires more extensive code changes
- Need to manage callbacks between Objective-C and Go
- More complex implementation

### Option 2: Force Menu Rebuild on Update

**Approach:** When `Update()` is called, close the currently open menu and reopen it with the updated content.

**Implementation Steps:**

1. Track whether the menu is currently open
2. When `Update()` is called while menu is open:
   - Close the menu programmatically
   - Update the NSMenu object
   - Reopen the menu immediately

**Pros:**
- Simpler to implement
- Uses existing code paths
- Minimal changes required

**Cons:**
- Visible flicker when menu closes/reopens
- Poor user experience
- Not a true "real-time" update

### Option 3: Periodic Menu Refresh

**Approach:** Implement a timer-based refresh mechanism that periodically updates menu items.

**Implementation Steps:**

1. When menu is opened, start a timer
2. Timer periodically calls menu item update functions
3. Stop timer when menu is closed

**Pros:**
- Automatic updates without explicit calls
- Simple to implement

**Cons:**
- Wasteful CPU usage
- Fixed refresh interval may not match update frequency
- Still requires delegate or other mechanism to trigger visual updates

## Recommendation

**Option 1 (NSMenuDelegate implementation)** is the recommended approach because:

1. It's the proper macOS-native solution
2. Provides the best user experience
3. Matches how native macOS menus work
4. Most efficient in terms of performance
5. Enables future enhancements

## Implementation Notes for Option 1

### Required Changes

1. **v3/pkg/application/systemtray_darwin.h**
   - Add delegate class declaration
   - Add functions to set delegate and trigger updates

2. **v3/pkg/application/systemtray_darwin.m**
   - Implement NSMenuDelegate protocol
   - Implement `menuNeedsUpdate:` method
   - Add callback mechanism to Go code

3. **v3/pkg/application/menu_darwin.go**
   - Add delegate setup in menu creation
   - Implement refresh mechanism that works with delegate

4. **v3/pkg/application/systemtray_darwin.go**
   - Update `setMenu()` to setup delegate properly
   - Add tracking for menu open/close state

### Example Delegate Implementation Pattern

```objective-c
@interface MenuDelegate : NSObject <NSMenuDelegate>
@property (nonatomic) void* menuPtr;  // Pointer to Go menu object
@end

@implementation MenuDelegate

- (void)menuNeedsUpdate:(NSMenu *)menu {
    // Call back to Go code to refresh menu items
    if (self.menuPtr != NULL) {
        refreshMenuFromGo(self.menuPtr);
    }
}

@end
```

## Related Work

- **PR #4615** - Fixed Linux system tray updates by adding the `refresh()` method
  - Only affects Linux platform (v3/pkg/application/systemtray_linux.go)
  - Introduced proper DBus signaling for menu updates
  - macOS requires a different approach due to different OS APIs

## Testing Considerations

1. Test menu updates while menu is open
2. Test with rapid updates (e.g., counter incrementing every 100ms)
3. Test with various menu item types (checkboxes, radio buttons, submenus)
4. Test memory management (ensure no leaks with frequent updates)
5. Test on different macOS versions (10.13+)

## References

- Issue: https://github.com/wailsapp/wails/issues/4630
- PR #4615: https://github.com/wailsapp/wails/pull/4615
- Apple NSMenu Documentation: https://developer.apple.com/documentation/appkit/nsmenu
- Apple NSMenuDelegate Documentation: https://developer.apple.com/documentation/appkit/nsmenudelegate

---

**Investigation Date:** 2025-11-19
**Wails Version:** v3-alpha (branch: v3-alpha)
**Investigator:** Claude (AI Assistant)
