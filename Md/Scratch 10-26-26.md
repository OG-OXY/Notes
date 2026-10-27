# Making Org Mode Look Good

## Why Vanilla Org Looks Rough in Terminal

1. *Raw Syntax Noise:* Headers show full `*`, `**`, `***` leading markers.
2. *Standard Bullets:* Lists display as plain `-` or `+` dashes instead of styled glyphs.
3. *No Visual Separation:* Block quotes (`#+BEGIN_QUOTE`), source code blocks,
and horizontal rules don't have distinct background subtle tints or margins.
4. *Metadata Overload:* Properties, tags, and drawer lines take up equal visual
weight as your actual prose.

---

## How to Make Org Mode Look Obsidian-Level Clean in Terminal

Drop these settings into your Emacs `init.el` or Org package configuration:

### 1. Hide Header Stars & Enable Clean Outline View

Replaces the ugly wall of stars (`***`) with clean indentation and modern bullet symbols.

```elisp
(use-package org
  :ensure nil
  :custom
  ;; Hide leading stars for clean indentation
  (org-hide-leading-stars t)
  (org-startup-indented t)
  ;; Hide formatting markup (*bold*, /italic/, etc.) like Obsidian does
  (org-hide-emphasis-markers t)
  ;; Pretty symbols for collapsed/expanded headers
  (org-ellipsis " ▾")
  ;; Prettier bullet lists
  (org-pretty-entities t))

```

### 2. Install `org-superstar` (Modern Bullet Icons)

Replaces default Markdown/Org heading asterisks with sleek UTF-8 glyphs that match
Obsidian’s visual hierarchy.

```elisp
(use-package org-superstar
  :ensure t
  :ghook 'org-mode-hook
  :custom
  ;; Custom bullet glyphs for levels 1 through 5
  (org-superstar-headline-bullets-list '("◉" "○" "✸" "✿" "•"))
  (org-superstar-special-item-bullet-alist '((?- . "•") (?+ . "➤"))))

```

### 3. Modern Callouts & Syntax Highlighting

To get Obsidian-style blockquotes and styled callouts (`> [!NOTE]`),
use `org-modern`.
It restyles headers, tags, tables, and code blocks directly inside terminal
buffers using clean Unicode blocks:

```elisp
(use-package org-modern
  :ensure t
  :ghook 'org-mode-hook
  :custom
  (org-modern-star '("◉" "○" "✸" "✿"))
  (org-modern-table t)
  (org-modern-block-name t)
  (org-modern-keyword t))

```

### 4. Auto-Display Inline Images in Terminal

Ensure graphics/images render inside Ghostty/terminal Emacs if you use modern
terminal graphics protocols:

```elisp
(setq org-startup-with-inline-images t)
(setq org-image-actual-width '(300))

```

---

## Quick Comparison

| Feature | Vanilla Terminal Org | Configured Modern Org | Obsidian |
| --- | --- | --- | --- |
| *Headers* | `*** Header Text` | ` ✸ Header Text` | Large Proportional Text |
| *Bullets* | `- Item` | ` • Item` | ` • Item` |
| *Emphasis* | `*bold text*` | **bold text** | **bold text** |
| *Tables* | ` | Raw | ASCII |

---

## Extra Optional Tweaks

### 1. Org Mode Code Blocks Are Missing Syntax Highlighting

By default, Emacs Org mode does *NOT* apply syntax highlighting to source
code blocks unless you explicitly turn it on and tell it to use font-lock!

Add this to your Org config in `init.el`:

```elisp
(setq org-src-fontify-natively t)       ;; Highlights Nix/Rust/C code inside #+begin_src blocks!
(setq org-src-tab-acts-natively t)      ;; Makes Tab key indent code correctly inside blocks
(setq org-src-preserve-indentation t)   ;; Preserves exact indentation

```

As soon as you add `org-src-fontify-natively t`, your `#+begin_src nix` blocks
in Org will get the exact same colorful, ultra-crisp syntax highlighting that
your Markdown files have in image 4.

---

### 2. Collapse/Hide Ugly Org Properties Drawers

In image 2, every heading is cluttered with `:PROPERTIES:`. To hide metadata noise:

```elisp
(setq org-cycle-hide-drawer-startup t)

```

---

### 3. Replace Boring Headlines with Variable Heights and Colors

Obsidian looks slick because headings are **larger and distinct** from body text.
Emacs GUI can do this natively, but out of the box Org uses the exact same plain
font size for everything.

If you are in GUI Emacs (or terminal with proper faces), add custom font sizes
and colors to your Org headings:

```elisp
(custom-set-faces
  '(org-level-1 ((t (:inherit outline-1 :height 1.35 :weight bold))))
  '(org-level-2 ((t (:inherit outline-2 :height 1.2 :weight bold))))
  '(org-level-3 ((t (:inherit outline-3 :height 1.1 :weight bold))))
  '(org-level-4 ((t (:inherit outline-4 :height 1.05 :weight bold)))))

```

---

### 4. Enable `org-modern` or `org-bullets`

In image 2, headings start with raw asterisks (`* 1. The Concrete File Layout`).

To turn those raw asterisks into sleek, modern bullet icons:

```elisp
(use-package org-modern
  :ensure t
  :hook (org-mode . org-modern-mode)
  :config
  (setq org-modern-star '("◉" "○" "✸" "✿")))

```

---

### Why Markdown Looked Better Out-of-the-Box

In Emacs, Markdown mode (`markdown-mode`) inherits font-locking from the
buffer's main color scheme by default, so code blocks immediately pick up your
editor's theme syntax colors.

Org mode, was built as an execution engine first and document viewer
second—so it keeps source blocks un-highlighted until you turn on
`(setq org-src-fontify-natively t)`.

Add `org-src-fontify-natively t` to your `init.el`, reload Emacs, and watch
those Nix code blocks in your Org document instantly light up with full syntax
highlighting!

# Batch Markdown to Org Conversion

Pandoc itself processes files individually or concatenates them,
but you can use a simple one-liner loop in your shell to recursively
or flatly convert a whole folder of `.md` files into `.org` files.

---

## Option 1: Flat Conversion (Current Directory)

If all your Markdown files are sitting directly inside a single folder,
navigate to that directory in your terminal and run this loop:

```bash
for f in *.md; do
    pandoc -f markdown -t org -o "${f%.md}.org" "$f"
done

```

- *How it works:* It loops through every `.md` file, strips the `.md`
extension using parameter expansion (`${f%.md}`), and outputs a matching `.org` file
(e.g., `notes.md` becomes `notes.org`).

---

## Option 2: Recursive Conversion (Including Subdirectories)

If your markdown files are nested deep inside subdirectories and you want to
convert them while *mirroring the directory structure*,
you can combine `find` with bash:

```bash
find . -name "*.md" -type f -exec bash -c '
  for file; do
    dest="${file%.md}.org"
    pandoc -f markdown -t org -o "$dest" "$file"
  done
' _ {} +

```

- *How it works:* It recursively searches for all `.md` files from the current
directory downwards and generates a corresponding `.org` file right alongside
each original markdown file.

---

## Alternative: Using Emacs (`org-pandoc-import`)

Since you mentioned running Emacs in the terminal, if you want a native
Emacs workflow to handle imports/exports cleanly without writing shell loops,
check out the community package `org-pandoc-import`.
It provides interactive commands inside Emacs to seamlessly pull entire
Markdown directory structures straight into Org mode.

# Super useful pgrep/emacsclient CMD's

- Two useful commands for querying Systemd Sockets and socket names with
`pgrep` and `ls` on the `$XDG_RUNTIME_DIR` variable.


````shell
pgrep -af emacs

ls -la $XDG_RUNTIME_DIR/emacs

````

# Emacs Lazy Loading

Hell no, lazy loading isn't exclusive to Doom at all! Doom just wraps standard
Emacs primitives—primarily **`use-package`**-behind its own heavy
macro-abstractions.

The best part? Since you're running Emacs 31 (`emacs-pgtk-31.1`), `use-package`
is built right into GNU Emacs core. You don't even need to pull in an external
library for it; it's native.

Instead of writing raw `require` statements that load everything upfront,
you use `use-package` keywords to defer loading until a command is actually run,
a hook fires, a key is pressed, or a file type is opened.

## How Lazy Loading Works in Vanilla Emacs

You get the exact same granular deferred loading that Doom uses,
but with full transparency:

```elisp
;;; init.el -*- lexical-binding: t; -*-

;; 1. Built-in use-package configuration
(require 'use-package)
(setq use-package-always-ensure nil) ; Since Nix manages packages, don't let MELPA auto-install

;; 2. Lazy load Evil only when entering modal states (or globally if you want it instantly)
(use-package evil
  :init
  (setq evil-want-keybinding nil)
  :config
  (evil-mode 1))

;; 3. Lazy load Magit ONLY when you run `M-x magit-status`
(use-package magit
  :commands (magit-status magit-dispatch))

;; 4. Lazy load Majutsu only when you invoke its workflow commands
(use-package majutsu
  :after magit
  :commands (majutsu-status))

;; 5. Lazy load your Helix Emulation Layer (hel) on global hook or keybinding
(use-package hel
  :after evil
  :hook (after-init . global-hel-mode))

```

## The Key Deferral Keywords

- *`:commands (cmd1 cmd2)`* — Doesn't load the package until you explicitly
run one of those commands.
- *`:bind ("C-c x" . some-func)`* — Doesn't load the package until you hit
that specific keybinding.
- *`:hook (some-mode . some-func)`* — Waits to load until that specific
major/minor mode hooks in.
- *`:defer t`* — Defers loading until idle time or explicit reference.

You aren't locked into Doom's ecosystem to get a snappy startup time; Vanilla
Emacs with proper `use-package` triggers will snap open instantly without
bloating your memory.

No sources used.

Do you want to map out clean lazy-loading blocks for your major editing and
version-control packages so your vanilla startup stays lightning fast?

# Vanilla Emacs Config Example

To translate this over to a clean, single-file or lightweight *Vanilla Emacs*
setup while using `hel` instead of `evil`, you drop Doom's macro-heavy
abstractions (`after!`, `add-hook!`, `package!`) in favor of native Elisp
(`with-eval-after-load`, `add-hook`, `use-package`).

Here is how you handle it:

1. *Remove Evil from your Nix packages list* so it doesn't even exist in the
environment to fight `hel`.
2. *Translate your Doom `config.el` into standard `use-package` blocks* inside
a single structured `init.el`.

---

## 1. Updated Nix Package List (`emacs.nix`)

Notice `epkgs.evil` and `epkgs.evil-mc` are completely removed, and we make
sure all dependencies your other packages need (like `plz`, `transient`, `dash`,
`avy`, `ultra-scroll` for `hel` and `majutsu`) are explicitly declared in the
list so Emacs can find them:

```nix
{
  inputs,
  ...
}:
{
  flake.nixosModules.emacs = { pkgs, ... }:
  let
    emacsDir = ./emacs.d;
    emacs = (pkgs.emacsPackagesFor pkgs.emacs-pgtk).emacsWithPackages (epkgs:
    let
      fpkgs = pname: src: packageRequires: epkgs.melpaBuild {
        inherit pname src packageRequires;
        version = "1.0.0";
      };
    in
    [
      epkgs.use-package
      epkgs.vertico
      epkgs.marginalia
      epkgs.consult
      epkgs.orderless
      epkgs.corfu
      epkgs.corfu-terminal
      epkgs.org-roam
      epkgs.org-appear
      epkgs.org-modern
      epkgs.org-superstar
      epkgs.pandoc-mode
      epkgs.undo-fu-session
      epkgs.magit
      epkgs.doom-themes
      epkgs.doom-modeline
      epkgs.treesit-grammars.with-all-grammars
      # Dependencies required by hel / majutsu
      epkgs.dash
      epkgs.avy
      epkgs.pcre2el
      epkgs.ultra-scroll
      epkgs.plz
      epkgs.transient
      (fpkgs "image-roll" inputs.image-roll [ ])
      (fpkgs "majutsu" inputs.majutsu [ epkgs.magit epkgs.consult epkgs.plz epkgs.transient ])
      (fpkgs "hel" inputs.hel [ epkgs.dash epkgs.avy epkgs.pcre2el epkgs.ultra-scroll ])
    ]);
  in
  {
    environment.systemPackages = [ emacs ];
    systemd.user.services.emacs-pgtk = {
      description = "Emacs PGTK Daemon";
      wantedBy = [ "default.target" ];
      after = [ "graphical-session.target" ];
      serviceConfig = {
        Type = "simple";
        ExecStart = "${emacs}/bin/emacs --init-directory=${emacsDir} --fg-daemon=emacs";
        ExecStop = "${emacs}/bin/emacsclient --socket-name=emacs --eval '(kill-emacs)'";
        Restart = "on-failure";
      };
    };    
  };
}

```

---

## 2. Complete Vanilla `init.el`

Place this file directly inside your `emacs.d/init.el` path. It completely
replaces Doom's split structure (`init.el`, `packages.el`, `config.el`) by
combining everything into standard vanilla `use-package` declarations.

```elisp
;;; init.el -*- lexical-binding: t; -*-

;; --- 1. Package & Environment Setup ---
(require 'use-package)
(setq use-package-always-ensure nil)

;; Basic UI cleanup
(menu-bar-mode -1)
(tool-bar-mode -1)
(scroll-bar-mode -1)
(global-display-line-numbers-mode 1)

;; Load Theme & Modeline
(load-theme 'doom-one t)
(doom-modeline-mode 1)

;; --- 2. Completion (Vertico / Corfu / Orderless) ---
(use-package vertico
  :init (vertico-mode 1))

(use-package marginalia
  :init (marginalia-mode 1))

(use-package consult
  :bind (("C-s" . consult-line)
         ("C-x b" . consult-buffer)))

(use-package orderless
  :custom
  (completion-styles '(orderless basic))
  (completion-category-defaults nil)
  (completion-category-overrides '((file (styles partial-completion)))))

(use-package corfu
  :init
  (global-corfu-mode 1)
  :custom
  (corfu-auto t)
  (corfu-quit-no-match t))

;; Terminal fallback for corfu if running emacs -nw inside Ghostty
(unless (display-graphic-p)
  (use-package corfu-terminal
    :init (corfu-terminal-mode +1)))

;; --- 3. Helix Emulation Layer (hel) ---
;; Completely isolated from Evil since Evil is absent from the system package list
(use-package hel
  :config
  (global-hel-mode +1))

;; --- 4. Persistent Undo (Undo-Fu-Session) ---
(use-package undo-fu-session
  :init
  (global-undo-fu-session-mode +1)
  :custom
  (undo-fu-session-file-limit 1000)
  (undo-fu-session-compression 'zlib))

;; --- 5. Org-mode & Ecosystem ---
(use-package org
  :config
  (setq org-directory "~/Notes/Org/"
        org-agenda-files (list org-directory)
        org-src-fontify-natively t
        org-src-tab-acts-natively t
        org-src-preserve-indentation t
        org-cycle-hide-drawer-startup t
        org-hide-leading-stars t
        org-hide-emphasis-markers t
        org-pretty-entities t
        org-startup-indented t
        org-ellipsis " ▾"
        org-startup-with-inline-images t
        image-use-external-converter t
        image-converter 'chafa
        browse-url-browser-function 'browse-url-xdg-open
        org-link-frame-setup
        '((vm . vm-visit-folder)
          (vm-imap . vm-visit-imap-folder)
          (gnus . gnus)
          (file . find-file)
          (wl . wl-other-frame))))

(use-package org-modern
  :hook (org-mode . org-modern-mode))

(use-package org-superstar
  :hook (org-mode . org-superstar-mode)
  :custom
  (org-superstar-headline-bullets-list '("◉" "○" "✸" "✿" "•"))
  (org-superstar-special-item-bullet-alist '((?- . "•") (?+ . "➤"))))

(use-package org-appear
  :hook (org-mode . org-appear-mode)
  :custom
  (org-appear-auto-emphasis t)
  (org-appear-autolinks t)
  (org-appear-autosubmarkers t))

(use-package org-roam
  :custom
  (org-roam-directory (expand-file-name "Roam" org-directory))
  :bind (("C-c n l" . org-roam-buffer-toggle)
         ("C-c n f" . org-roam-node-find)
         ("C-c n i" . org-roam-node-insert))
  :config
  (org-roam-setup))

;; --- 6. Pandoc & Markdown Integration ---
(use-package pandoc-mode
  :hook ((markdown-mode org-mode) . turn-on-pandoc))

;; --- 7. Additional Tools (Magit, Majutsu, Image-Roll) ---
(use-package magit
  :bind ("C-x g" . magit-status))

(use-package majutsu
  :after magit)

(use-package image-roll
  :hook (image-mode . image-roll-mode))

;; --- 8. GUI-Only Specific Configurations ---
(when (display-graphic-p)
  (custom-set-faces
   '(org-level-1 ((t (:inherit outline-1 :height 1.35 :weight bold))))
   '(org-level-2 ((t (:inherit outline-2 :height 1.2 :weight bold))))
   '(org-level-3 ((t (:inherit outline-3 :height 1.1 :weight bold))))
   '(org-level-4 ((t (:inherit outline-4 :height 1.05 :weight bold))))))

;; Terminal echo area safeguard for tooltips
(unless (display-graphic-p)
  (setq tooltip-use-echo-area t))

```

# Overrides for Emacs's `H-E-L`

In Helix (and Kakoune), *a 1-character selection IS the normal cursor state*.
There is no such thing as a cursor floating in the void "between" characters
without highlighting what it's on.
When you jump around, that single highlighted box gives you instant, unmistakable
spatial feedback: *if I press `d` or `c` right now, THIS exact character dies.*

When `hel` strips that away, forces a thin bar cursor, and defaults to
deleting *backward* without a visual region, it throws out the core promise of
object-action editing. You're left guessing where the point boundary actually sits.

---

## The Clean Overrides to Reclaim Your Helix Experience

Since we know *why* `hel` was behaving so weirdly, we can completely override
the maintainer's opinions in your `init.el`.

Here is the targeted snippet to drop into your Vanilla `init.el` under the
`(use-package hel)` block. It reinstates the high-visibility *block cursor*,
forces `d` to delete forward (or delete the active selection), and fixes
`gl`/`gh` so they act like simple, non-selecting goto motions:

```elisp
;; --- 3. Helix Emulation Layer (hel) ---
(use-package hel
  :config
  ;; 1. OVERRIDE AUTHOR CURSOR DEFAULTS: Bring back high-visibility block cursor in Normal state!
  (setq hel-normal-cursor 'box       ; Solid block cursor so you always see the 1-character selection
        hel-insert-cursor 'hbar)     ; Underline (or 'bar) for Insert state so your eyes don't lose it

  ;; 2. Activate hel safely
  (if (fboundp 'hel-global-mode)
      (hel-global-mode 1)
    (global-hel-mode 1)))

;; Fix keybindings & motion math after hel loads
(with-eval-after-load 'hel
  ;; FIX 'd': Delete active selection, OR delete character forward under the block cursor
  (defun my/hel-delete-inclusive ()
    "Delete active region if present, otherwise delete character directly under the block cursor."
    (interactive)
    (if (use-region-p)
        (call-interactively #'hel-delete)
      (delete-char 1)))

  (define-key hel-normal-mode-map (kbd "d") #'my/hel-delete-inclusive)

  ;; FIX 'gl' and 'gh': Jump to line start/end WITHOUT forcing persistent regional selections
  (defun my/hel-goto-line-end ()
    (interactive)
    (deactivate-mark)
    (move-end-of-line 1))

  (defun my/hel-goto-line-start ()
    (interactive)
    (deactivate-mark)
    (move-beginning-of-line 1))

  (when (boundp 'hel-goto-map)
    (define-key hel-goto-map (kbd "l") #'my/hel-goto-line-end)
    (define-key hel-goto-map (kbd "h") #'my/hel-goto-line-start)))

```

---

## Why this fixes the friction immediately:

1. *Visual Clarity:* You get your solid high-contrast block cursor back in
Normal mode—no more losing a bar cursor or guessing where the boundary is.
2. *Predictable Deletions:* Tapping `d` on a character deletes the character
under the block. Tapping `d` on an active region deletes the region.
3. *Clean Goto Motions:* `gh` and `gl` just jump your cursor to line start/end
without forcing unwanted highlights across the whole line.
