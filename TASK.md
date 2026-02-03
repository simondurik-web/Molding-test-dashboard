# Task: Fix Sidebar Toggle Button Positioning

## File: index.html (single-file HTML dashboard, ~8000 lines)

## Problem
The sidebar toggle button (Hide/Menu) overlaps dashboard content. It's currently a floating/absolute element that covers things.

## What to Do

1. Move the toggle button to the LEFT side, near the top of the sidebar
2. Place it ABOVE or WITHIN the sidebar header area (near "Molding Operations Dashboard" title)
3. When sidebar is HIDDEN: button stays visible as a small tab on the left edge, does NOT overlap main content
4. When sidebar is VISIBLE: button is naturally part of the sidebar header
5. Adjust main content margin/padding so nothing overlaps in either state
6. Must work in both light and dark themes

## Rules
- SURGICAL edits only — do NOT regenerate or rewrite large sections
- Preserve ALL existing functionality
- The language variable is `currentLang` (NOT `currentLanguage`)
- Look for the `toggleSidebar` function and `.sidebar-toggle` button

## After Done
Commit with message: "v7.19 - Fix sidebar toggle button positioning (no overlap)"
Then push to origin.
