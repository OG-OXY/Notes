---
id: vimiumC
aliases: []
tags: []
---
# Vimium C Configuration Guide

Vimium C is a browser extension that brings Vim-like keyboard navigation, shortcuts, and motion commands to Chromium-based browsers (and Firefox). This guide covers installation, core keybindings, advanced command mapping, and customization.

---

## 1. Installation & Core Philosophy

1. Install **Vimium C** from the Chrome Web Store or Microsoft Edge Add-ons.
2. Press `F` (or `f`) on any webpage to bring up link hints. Type the corresponding letter hints to follow links without ever touching the mouse.

---

## 2. Essential Default Keybindings

| Key | Action |
| :--- | :--- |
| `j` / `k` | Scroll down / up smoothly |
| `h` / `l` | Scroll left / right |
| `gg` / `G` | Scroll to top / bottom of page |
| `f` / `F` | Open link in current / new tab using hints |
| `H` / `L` | Go back / forward in history |
| `r` | Reload current page |
| `yy` | Copy current URL to clipboard |
| `p` / `P` | Open clipboard URL in current / new tab |

---

## 3. Custom Key Mappings

You can customize your navigation workflow by adding custom commands to the **Custom key mappings** section in the Vimium C options page.

### Example Configuration Block
```text
# Map 'J' and 'K' to switch tabs quickly
map J nextTab
map K previousTab

# Open a new tab and focus the omnibar directly
map t Vomnibar.activate

# Custom search alias for GitHub
map og Vomnibar.activateKeyword engine=github
