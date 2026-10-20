That exact bug where focus freezes on a window until you kill it is an absolute rage-inducer, but there is a very specific Wayland/Niri reason why it happens—and a couple of exact fixes so you stop losing your state.

---

### What's Actually Causing the Focus Freeze

In Wayland compositors (like Niri), when an application issues an input grab—such as an active pop-up, right-click context menu, or drop-down preview that loses track of its pointer state—it creates a **Wayland seat lock**.

The compositor thinks the window is still holding an active modal/grab event. Because Wayland strictly isolates input focus between client windows for security, Niri refuses to hand keyboard/mouse focus to *any* other window until that pointer grab is released.

Common culprits in terminal setups:

1. **XWayland Handshake Hangs:** If Ghostty or another app falls back to XWayland (or interacts with an XWayland app like default Obsidian), the bridge between X11 input grabs and Wayland seat focus gets stuck.
2. **Terminal CSD / Border Grabs:** Client-Side Decorations (CSDs) trying to draw custom titlebars or mouse-drag areas inside Niri.
3. **Overriding Keybinds:** Hotkeys firing inside the terminal while Niri is trying to interject a global compositor bind.

---

### How to Fix It permanently in NixOS / Niri

#### 1. Force Native Wayland Everywhere

Make sure your terminal and all apps are running pure Wayland (no XWayland proxy bugs). In your NixOS or home-manager configuration, set:

```nix
environment.sessionVariables = {
  NIXOS_OZONE_WL = "1";
  ELECTRON_OZONE_PLATFORM_HINT = "auto";
  GDK_BACKEND = "wayland";
  QT_QPA_PLATFORM = "wayland";
};

```

#### 2. Disable Client-Side Decorations in `config.kdl`

Niri provides native focus-rings and window borders. If apps try to draw their own CSD, they frequently lock mouse/keyboard pointer grabs.

In your `~/.config/niri/config.kdl`:

```kdl
prefer-no-csd

```

*(This forces applications to let Niri manage window decorations and focus handoffs natively).*

#### 3. The Emergency Focus Un-sticker (Without Killing Ghostty!)

If focus ever gets locked again before you tweak your config, **do not kill your terminal**. Try these keyboard bypasses first:

* **Trigger a Window Resize:** Press your Niri resize hotkey (usually `Mod + R` or `Mod + Mouse Right Click`). Forcing Niri to recalculate surface dimensions breaks active Wayland pointer grabs instantly.
* **Switch Workspace and Back:** Use `Mod + Page_Down` / `Mod + Page_Up` (or your workspace bind) to move to an empty workspace and back. This forces the Wayland compositor seat to reset keyboard focus.
* **Restart Niri in-place:** You can restart the Niri compositor without killing your open processes by running `niri msg action quit` or using the compositor reload hotkey.

---

### The Tab Saver (Automate tab recovery in Ghostty)

If you ever *do* have to close Ghostty, you shouldn't be manually re-creating tabs. You can pass a startup command or script to Ghostty in your Nix config to spin up your standard 6-tab workspace layout in a single keystroke.

You can add a small custom Fish function to launch your workspace preset automatically:

```fish
function dev-session
    # Spawns Ghostty tabs with pre-defined names/commands
    ghostty -e helix . &
    # or leverage Ghostty configuration actions / multiplexer session hooks
end

```

Combining native Wayland flags with `prefer-no-csd` in Niri usually kills that focus lock for good so you can stay in Ghostty at full speed without your setup locking up.