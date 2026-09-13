Zen Browser handles its UI (vertical tab bar, side panels, split views) through its internal shortcuts page, but its default keybindings only cover top-level browser chrome actions—not web page link clicking.

Fixing this requires configuring Zen's native keymap alongside page-level hint navigation.

### 1. Configure Native Zen Shortcuts (Chrome Level)

Go to **Settings (`Ctrl + ,`) -> Keyboard Shortcuts** inside Zen. Set up custom bindings for Zen-specific features:

|Feature|Recommended Zen Binding|Action|
|---|---|---|
|**Tab Movement**|`Ctrl + J` / `Ctrl + K`|Switch to Next / Previous vertical tab|
|**Split View**|`Alt + V` / `Alt + S`|Vertical / Horizontal split active tab|
|**Split Focus**|`Alt + H/J/K/L`|Change focus between split panels|
|**Zen Sidebar / Web Panels**|`Ctrl + B`|Toggle side panel|
|**URL Bar Focus**|`Ctrl + L` or `o`|Open URL bar instantly|

### 2. Plug the Gap: Page-Level Navigation (Vimium C)

Zen is Firefox-based, meaning native extensions run at full performance. Installing **Vimium C** from the Firefox Add-ons store handles full link navigation:

1. **Install Vimium C** in Zen.
2. Hit `**f**`: Overlays 2-letter hints over every link, button, and image on the web page. Type the letters to click without touching the mouse.
3. `**gi**`: Focuses directly on text boxes/search inputs.
4. **`j` / `k` / `d` / `u`**: Smooth scrolling down, up, half-page down, half-page up.
5. **`H` / `L`**: Go back / forward in browser history.

### 3. Handle Extension Dead-Zones (Wayland / Hyprland Level)

Web extensions like Vimium C cannot run on internal browser pages (e.g., `about:preferences`, `about:blank`, or Zen's settings page) due to Firefox security restrictions.

To navigate these non-web pages without grabbing the mouse:

- Use `**Tab**` and `**Shift + Tab**` to cycle elements, and `**Space**` to select.
- Use `**warpd**` via Hyprland:
    
    # Hyprland binding to jump mouse cursor anywhere via keyboard
    bind = $mainMod, G, exec, warpd --hint
    
    Pressing `SUPER + G` overlays letters across the screen. Typing the letters teleports the cursor and left-clicks, handling internal browser menus, upload dialogs, or rogue web interfaces seamlessly.