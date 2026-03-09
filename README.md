# iPhone Simulator Control for Claude Code

A lightweight [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that gives Claude full control over the iOS Simulator. No bundled scripts — just a single prompt file that teaches Claude to use native macOS tools (`xcrun simctl`, `idb`, `cliclick`, `osascript`) directly.

## How It Works

This skill is a single `SKILL.md` file. When invoked, it instructs Claude how to:

- **Navigate via accessibility tree** — find buttons, text fields, and other elements by label/type using `idb`, no screenshots needed
- **Fall back to screenshots** — when `idb` isn't available, uses `cliclick` with coordinate mapping
- **Tap, type, swipe** — interact with any UI element
- **Control the simulator** — dark mode, status bar, location, permissions, app lifecycle
- **Automate menus** — Home button, rotate, shake, keyboard toggles via AppleScript

## Installation

Copy the skill into your Claude Code skills directory:

```bash
# Global (available in all projects)
cp -r .claude/skills/iphone-sim ~/.claude/skills/

# Or project-level (available only in this project)
# Just clone/copy this repo — the skill is already in .claude/skills/
```

### Dependencies

**Recommended** (accessibility-tree navigation):
```bash
brew tap facebook/fb && brew install idb-companion && pip3 install fb-idb
```

**Fallback** (screenshot + coordinate-based navigation):
```bash
brew install cliclick
```

You also need Xcode with a simulator device. Verify with:
```bash
xcrun simctl list devices booted
```

## Usage

Once installed, Claude Code automatically uses the skill when you ask it to interact with the simulator:

```
> /iphone-sim screenshot
> /iphone-sim tap the Login button
> /iphone-sim type "hello@example.com" in the email field
> /iphone-sim swipe up
> /iphone-sim launch com.apple.mobilesafari
> /iphone-sim dark mode
```

Or just ask naturally — Claude will invoke the skill when it's relevant:

```
> Take a screenshot of the simulator
> Tap on Settings and enable Dark Mode
> Fill in the login form and submit it
```

## Two Navigation Modes

### Accessibility Tree (preferred, requires `idb`)

Claude reads the UI hierarchy directly — element types, labels, and coordinates — without needing a screenshot. Faster, cheaper (fewer tokens), and more reliable across UI changes.

```
describe-all → find element by label → tap by frame center → verify
```

### Screenshot-Based (fallback, requires `cliclick`)

Claude takes a screenshot, reads it visually, maps simulator coordinates to macOS screen coordinates, and clicks with `cliclick`. Works without `idb` but costs more tokens per interaction.

```
screenshot → read image → calculate coordinates → cliclick tap → verify
```

## What's Included

| File | Purpose |
|------|---------|
| `.claude/skills/iphone-sim/SKILL.md` | The skill — Claude's instructions for simulator control |

## Capabilities

- Screenshots (with auto-resize to stay within token limits)
- Tap, double-tap, long press
- Text input (direct typing, pasteboard, special keys)
- Scrolling and swiping
- App launch, terminate, install, uninstall, deep links
- Dark/light mode, content size, status bar overrides
- Location simulation
- Privacy/permission grants
- Push notification simulation
- Menu automation (Home, Shake, Rotate, Keyboard toggles)

## License

MIT License — see [LICENSE](LICENSE).

Copyright (c) 2026 Warped Technologies LLC. Created by Adam Sulik.
