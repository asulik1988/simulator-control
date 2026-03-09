---
name: iphone-sim
description: "Interact with the iPhone Simulator — navigate via accessibility tree, tap UI elements, type text, swipe, and control device settings. Use when asked to test, interact with, or verify anything in the iOS Simulator."
argument-hint: "[screenshot | tap <description> | type <text> | swipe <direction> | home | launch <bundle-id> | terminate <bundle-id> | dark | light | status-bar]"
allowed-tools: Bash, Read
---

# iPhone Simulator Control

You are controlling an iOS Simulator running on macOS. Use `xcrun simctl`, `idb`, `cliclick`, and `osascript` to interact with it.

**The user's request:** $ARGUMENTS

## Core Workflow

**Prefer the accessibility tree over screenshots for navigation.** The accessibility tree gives you element types, labels, and frames — structured data that's cheaper and more reliable than image analysis.

1. **Check tools**: Run `which idb` to determine if idb is available
2. **Focus** the simulator: `osascript -e 'tell application "Simulator" to activate'`
3. **Describe** the screen: `idb ui describe-all` (if idb available)
4. **Act** — tap by coordinates from the accessibility tree, type text, swipe, etc.
5. **Verify** — describe again or take a screenshot for visual confirmation
6. **Report** what happened

**Fallback (no idb):** Screenshot → Read → identify positions visually → act using cliclick (see "Screenshot Navigation" section below).

## Prerequisites

- `idb` recommended (`brew tap facebook/fb && brew install idb-companion && pip3 install fb-idb`)
- `cliclick` as fallback for coordinate-based interaction (`brew install cliclick`)
- Simulator must be running with a booted device
- Check with: `xcrun simctl list devices booted`

## Accessibility Tree Navigation (Preferred)

When `idb` is available, use the accessibility tree to find and interact with elements semantically — no screenshots needed.

**Note:** `--udid UDID` is a global flag on `idb`, not on subcommands. Use `idb --udid booted <command>` or omit it if only one simulator is booted.

### Describe the Screen

```bash
# Full accessibility tree — JSON list of all elements with types, labels, bounds
idb ui describe-all
idb --udid UDID ui describe-all          # specific device

# Describe element at a specific point (simulator coordinates)
idb ui describe-point X Y
```

The output includes element **type** (Button, TextField, StaticText, etc.), **label**, **frame** (x, y, width, height in simulator points), and **enabled/focused** state.

### Find Elements

Parse the `describe-all` output to locate elements:
- **By label text**: Look for the element whose label matches your target (e.g., "Login", "Submit")
- **By type**: Filter for `Button`, `TextField`, `SecureTextField`, `Switch`, etc.
- **By frame**: The frame gives you the tap coordinates in simulator points

### Tap

```bash
# Tap at simulator-point coordinates (from the element's frame center)
# Calculate center: center_x = frame_x + frame_width/2, center_y = frame_y + frame_height/2
idb ui tap $CENTER_X $CENTER_Y
idb ui tap X Y --duration 1.5            # long press
```

**Key advantage:** idb uses simulator-point coordinates directly — no macOS screen coordinate mapping needed.

### Type Text

```bash
idb ui text "Hello World"
```

### Key Presses

```bash
idb ui key KEYCODE                        # single key press
idb ui key KEYCODE --duration 1.0         # hold key
idb ui key-sequence 4 5 6                 # sequential key presses
```

### Hardware Buttons

```bash
idb ui button HOME
idb ui button LOCK
idb ui button SIRI
idb ui button SIDE_BUTTON
idb ui button APPLE_PAY
idb ui button HOME --duration 2.0         # hold button
```

### Swipe

```bash
# Swipe from (x1,y1) to (x2,y2) in simulator points
idb ui swipe $X1 $Y1 $X2 $Y2
idb ui swipe $X1 $Y1 $X2 $Y2 --delta 20  # custom step size in points
```

### Other idb Commands

```bash
idb focus                                 # bring simulator to foreground
idb open "myapp://deep-link"              # open URL scheme
idb set_location 42.3601 -71.0589         # override location (note: space, not comma)
idb approve com.example.app photos camera location contacts  # grant permissions
idb add-media cat.jpg dog.mov             # add to camera roll
idb clear_keychain                        # clear keychain
idb log                                   # tail device logs
idb record video output.mp4               # record screen (Ctrl+C to stop)
```

### Workflow Example (with idb)

```bash
# 1. See what's on screen
idb ui describe-all

# 2. Find "Login" button in output — e.g., frame: {x: 137, y: 680, w: 100, h: 44}
# Center = (187, 702)

# 3. Tap it
idb ui tap 187 702

# 4. Verify
idb ui describe-all
```

### When to Still Use Screenshots

- **Visual verification**: Confirming colors, layout, images, or visual state
- **Bug reports**: Showing the user what happened
- **No idb**: Fall back to the screenshot-based workflow below

## Screenshot Navigation (Fallback)

Use this approach when `idb` is not available, or when you need visual verification.

**IMPORTANT:** Always resize screenshots after capturing. Raw simulator screenshots (e.g. 1179x2556) exceed the 2000px image dimension limit and will break the conversation context.

```bash
xcrun simctl io booted screenshot /tmp/sim_screenshot.png && sips -Z 1800 /tmp/sim_screenshot.png --out /tmp/sim_screenshot.png
```
Use the Read tool to view the PNG. This is how you "see" the simulator.

## Coordinate Mapping

To tap on a UI element, you must convert simulator pixel coordinates to macOS screen coordinates.

```bash
# Get window position and size
osascript -e 'tell application "System Events" to tell process "Simulator" to get {position, size} of window 1'
# Returns: win_x, win_y, win_w, win_h
```

**Formula (no title bar offset needed — verified by testing):**
```
mac_x = win_x + (sim_x / device_width) * win_w
mac_y = win_y + (sim_y / device_height) * win_h
```

Get device resolution from: `sips -g pixelWidth -g pixelHeight /tmp/sim_screenshot.png`
Common: iPhone 16 = 1179x2556

**Example calculation in bash:**
```bash
BOUNDS=$(osascript -e 'tell application "System Events" to tell process "Simulator" to get {position, size} of window 1')
WIN_X=$(echo $BOUNDS | cut -d',' -f1 | tr -d ' ')
WIN_Y=$(echo $BOUNDS | cut -d',' -f2 | tr -d ' ')
WIN_W=$(echo $BOUNDS | cut -d',' -f3 | tr -d ' ')
WIN_H=$(echo $BOUNDS | cut -d',' -f4 | tr -d ' ')

# For sim coords (SIM_X, SIM_Y) on a 1179x2556 device:
MAC_X=$(python3 -c "print(int($WIN_X + ($SIM_X / 1179) * $WIN_W))")
MAC_Y=$(python3 -c "print(int($WIN_Y + ($SIM_Y / 2556) * $WIN_H))")
```

## Tapping

```bash
cliclick c:$MAC_X,$MAC_Y        # single tap
cliclick dc:$MAC_X,$MAC_Y       # double tap
```

Always focus the simulator first and re-read window bounds before each tap (window may have moved).

## Typing Text

**Method 1 — Direct typing** (requires Hardware Keyboard connected in I/O > Keyboard):
```bash
cliclick t:'Hello World'
```

**Method 2 — Pasteboard** (more reliable, works always):
```bash
echo -n "Hello World" | xcrun simctl pbcopy booted
cliclick kd:cmd t:v ku:cmd
```

**Special keys:**
```bash
cliclick kp:return       # Enter/Return
cliclick kp:delete       # Backspace
cliclick kp:tab          # Tab
cliclick kp:space        # Space
cliclick kp:esc          # Escape
cliclick kp:arrow-up     # Arrow keys
cliclick kp:arrow-down
```

**IMPORTANT:** `kp:` only works with special key names. For letter keys with modifiers, use `kd:` + `t:` + `ku:`:
```bash
cliclick kd:cmd t:a ku:cmd       # Cmd+A (select all)
cliclick kd:cmd t:v ku:cmd       # Cmd+V (paste)
cliclick kd:cmd,shift t:h ku:cmd,shift  # Cmd+Shift+H (Home)
```

## Scrolling / Swiping

```bash
# Swipe up (scroll down)
cliclick dd:$X,$START_Y du:$X,$END_Y

# Swipe down (scroll up)
cliclick dd:$X,$START_Y du:$X,$END_Y   # reverse Y values

# Swipe with intermediate points for longer scrolls
cliclick dd:$X,700 dm:$X,500 du:$X,300
```

**WARNING:** Swiping works for in-app scrolling but is too slow for system gestures (unlock, app switcher). Use menu automation for those.

## App Lifecycle

```bash
xcrun simctl launch booted <bundle-id>
xcrun simctl launch --console booted <bundle-id>    # with stdout
xcrun simctl terminate booted <bundle-id>
xcrun simctl uninstall booted <bundle-id>
xcrun simctl install booted /path/to/App.app
xcrun simctl openurl booted "myapp://deep-link"      # opens in default handler
```

## Device Controls

```bash
# Appearance
xcrun simctl ui booted appearance dark
xcrun simctl ui booted appearance light

# Content size
xcrun simctl ui booted content_size extra-large
xcrun simctl ui booted content_size large              # default

# Status bar (clean for screenshots)
xcrun simctl status_bar booted override --time "9:41" --batteryState charged --batteryLevel 100 --wifiBars 3 --cellularBars 4
xcrun simctl status_bar booted clear

# Location
xcrun simctl location booted set 42.3601,-71.0589

# Permissions (grant without prompts)
xcrun simctl privacy booted grant photos <bundle-id>
xcrun simctl privacy booted grant camera <bundle-id>
xcrun simctl privacy booted grant location <bundle-id>

# Pasteboard
echo -n "text" | xcrun simctl pbcopy booted
xcrun simctl pbpaste booted

# Add media
xcrun simctl addmedia booted /path/to/photo.jpg
```

## Menu Automation (AppleScript)

```bash
# Home button (most reliable way)
osascript -e 'tell application "System Events" to tell process "Simulator" to click menu item "Home" of menu "Device" of menu bar 1'

# Also unlocks from lock screen ^

# Shake gesture
osascript -e 'tell application "System Events" to tell process "Simulator" to click menu item "Shake" of menu "Device" of menu bar 1'

# Rotate
osascript -e 'tell application "System Events" to tell process "Simulator" to click menu item "Rotate Left" of menu "Device" of menu bar 1'

# Toggle hardware keyboard
osascript -e 'tell application "System Events" to tell process "Simulator" to click menu item "Connect Hardware Keyboard" of menu "Keyboard" of menu item "Keyboard" of menu "I/O" of menu bar 1'

# Toggle software keyboard
osascript -e 'tell application "System Events" to tell process "Simulator" to click menu item "Toggle Software Keyboard" of menu "Keyboard" of menu item "Keyboard" of menu "I/O" of menu bar 1'

# List any menu items
osascript -e 'tell application "System Events" to tell process "Simulator" to get name of every menu item of menu "Device" of menu bar 1'
```

## Common Pitfalls

- **Use idb first**: If `idb` is available, prefer `idb ui describe-all` + `idb ui tap` over screenshots + cliclick. It's faster, cheaper (tokens), and more reliable.
- **idb `--udid` is global**: Use `idb --udid UDID ui tap X Y`, not `idb ui tap --udid UDID X Y`.
- **idb coordinates are simulator points**: Unlike cliclick (macOS screen pixels), idb uses the simulator's own coordinate system. No conversion needed.
- **idb `set_location` uses spaces**: `idb set_location 42.36 -71.06` (not comma-separated like simctl).
- **idb not finding elements**: Some elements may not have accessibility labels. Fall back to screenshots for unlabeled UI.
- **Typing fails silently**: Hardware Keyboard must be connected (I/O > Keyboard). The text field IS focused even without a visible software keyboard.
- **Clicks miss (cliclick)**: Always re-read window bounds before clicking — they change when the window moves.
- **Lock screen stuck on black**: Use Device > Home menu to wake. Do NOT try to swipe-unlock.
- **`kp:h` error**: `kp:` is for special keys only. Use `t:h` with `kd:/ku:` for letter keys.
- **`openurl` opens Safari**: That's expected for http URLs. Use `xcrun simctl launch` to switch back.
- **Multiple simulators**: Use UDID instead of "booted" if more than one sim is running.
