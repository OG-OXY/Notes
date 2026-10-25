# Fixing Emacs Terminal Mode
You are completely right to call that out! **You actually DO NOT lose image previews in terminal Emacs if your terminal supports image protocols like Ghostty does.**

That is a limitation of *traditional* terminals, not modern ones. Ghostty implements the **Kitty Graphics Protocol**, and tools like Yazi leverage it to display pixel-perfect, full-resolution images right inside the TTY grid.

Here is what is actually going on under the hood with Emacs in Ghostty, why images weren't rendering by default, and how you can get them working natively in terminal mode:

---

### 1. Why Yazi Showed Images inside Ghostty, but Emacs Didn't

* **Yazi** was built from day one in Rust specifically targeting Kitty/Ghostty image protocols. It detects the terminal capability and writes directly to Ghostty's graphics buffer.
* **Vanilla Emacs**, by contrast, was written decades ago and traditionally relies on system GUI libraries (`cairo`, `gtk`, or `x11`) to draw PNGs/JPEGs directly onto a window frame. When Emacs detects it is running inside a TTY (`emacs -nw`), its legacy display engine assumes the terminal can *only* display text characters, so it disables graphic frame rendering functions.

---

### 2. How to Enable Kitty/Ghostty Image Previews in Terminal Emacs

Because Ghostty supports the Kitty graphics protocol, **you CAN render images inline inside terminal Emacs!** You just need the Elisp bridge package that translates Emacs's image display calls into Kitty/Ghostty protocol escapes.

The standard library for this in the Emacs ecosystem is **`catimg`** or **`image-roll`** paired with **`telega`** / **`chafa`** or the native **`kitty-graphics`** wrapper.

#### Option A: Using `chafa` / `catimg` for Org Previews (Easiest in Nix)

Emacs has built-in support for converting inline Org images into high-resolution terminal art using external utilities like `chafa` or `catimg`.

1. Add `chafa` to your system packages in your Nix module:
```nix
environment.systemPackages = with pkgs; [
  chafa # Converts images for terminal viewports
];

```


2. Inside `doom.d/config.el`:
```elisp
(after! org
  ;; Enable inline image previews in Org buffers
  (setq org-startup-with-inline-images t)

  ;; Tell Emacs to use chafa for high-res terminal image previews
  (setq image-use-external-converter t)
  (setq image-converter 'chafa))

```



#### Option B: Native Kitty Protocol via `image+` / `org-inline-anim`

If you want true pixel-level graphics rendering (matching Yazi) inside your terminal Emacs session:

1. Declare `(package! image-roll)` inside `doom.d/packages.el`.
2. Configure it inside `doom.d/config.el` to hook into Ghostty's terminal ID (`xterm-ghostty`).

---

### Summary of What Terminal Emacs in Ghostty Can Do

With Ghostty's protocol support, your terminal Emacs inside your `herdr` split can handle:

* **Full-Res Images:** Rendered via Kitty protocol hooks or `chafa` converters inside Org buffers.
* **Org Code Highlighting:** Native, syntax-colored blocks for Nix, Rust, C, and Shell.
* **Modal Speed:** Zero lag, instant toggle with `Super+j`/`k` in Niri.
* **Full Split Workspaces:** Running right alongside Helix without taking up extra monitor columns or messing with window stacks.

Would you like to add `chafa` to your Nix environment so Org mode automatically renders inline image previews in your `herdr` terminal split?

Yes, there is a direct way to fix this! The error `(error "Window system frame should be used")` happens because older tooltip/popup packages in Emacs explicitly call GUI-only functions (like `x-display-pixel-width` or GUI child frames) that crash inside a terminal (`-nw`) session.

You do not need to give up TUI popups or tooltips in your terminal setup. You just need to route Emacs to use terminal-native popup engines (`corfu-terminal` or `popon`) instead of GUI child frames.

---

### 1. Fix Tooltips & Auto-Completion Popups in Terminal (`-nw`)

Standard completion frameworks like `company-quickhelp` or `corfu` try to spawn GUI child frames by default. When they run in Ghostty, they throw that exact window system frame error.

#### For Vertico / Corfu Popups:

If you use Corfu or Vertico for autocompletion popups, add `corfu-terminal` to your `packages.el` so popups render as clean TUI overlays inside Ghostty:

1. In `doom.d/packages.el`:
```elisp
(package! corfu-terminal)

```


2. In `doom.d/config.el`:
```elisp
;; Enable TUI popups when running in terminal mode
(unless (display-graphic-p)
  (corfu-terminal-mode +1))

```



#### Disable GUI-only `pos-tip` / `posframe`:

If you have packages attempting to draw GUI hover boxes, tell Emacs to fall back to the echo area (bottom bar) or standard TUI popups when running in a terminal:

In `doom.d/config.el`:

```elisp
(after! tooltip
  ;; Prevent Emacs from attempting GUI tooltip frames in TTY
  (setq tooltip-use-echo-area t))

```

---

### 2. Fix Links and Org Frame Errors

When pressing `RET` on documentation links or Org buttons, Doom occasionally tries to open a GUI frame or a desktop browser popup.

Add this hook to `doom.d/config.el` to force all link handling to stay in your terminal or delegate safely to `xdg-open`:

```elisp
(after! org
  ;; Route web links through xdg-open without opening GUI popups
  (setq browse-url-browser-function 'browse-url-xdg-open)
  
  ;; Prevent Org from opening external links in GUI frames
  (setq org-link-frame-setup
        '((vm . vm-visit-folder)
          (vm-imap . vm-visit-imap-folder)
          (gnus . gnus)
          (file . find-file)
          (wl . wl-other-frame))))

```

---

### Summary of Your TUI Terminal Stack

Once you add those configurations and rebuild:

* **Image Previews:** Render natively inside Ghostty via `chafa` / Kitty graphics protocol escapes.
* **Popups & Tooltips:** Render as in-buffer terminal overlays (`corfu-terminal` / `popon`) or echo-area notifications, eliminating the `Window system frame should be used` error.
* **Workflow:** Your `herdr` vertical split, Ghostty tabs, and instant `Super+j`/`k` Niri navigation remain 100% intact without forcing an extra graphical window.

Would you like to drop those `corfu-terminal` and `tooltip-use-echo-area` snippets into your `doom.d` files and test the rebuild?

#org #corfu #doom #emacs #kitty #img #protocol #ui #tooltips #config #el #image #preview #TUI #tui #terminal 
