The main reason Org mode looks plain out of the box in terminal Emacs—especially compared to Obsidian—is that **vanilla terminal Emacs defaults to raw text with ASCII symbols**.

Obsidian applies full CSS layout, variable-pitch fonts, rich padding, callout boxes, and smooth UI elements natively in an Electron renderer. Terminal Emacs *can* look shockingly gorgeous, but because it runs inside a TTY/terminal grid, it requires a few specific packages and settings to give you that clean, polished look with proper bullet rendering, clean headers, and visual hierarchy.

---

### Why Vanilla Org Looks Rough in Terminal

1. **Raw Syntax Noise:** Headers show full `*`, `**`, `***` leading markers.
2. **Standard Bullets:** Lists display as plain `-` or `+` dashes instead of styled glyphs.
3. **No Visual Separation:** Block quotes (`#+BEGIN_QUOTE`), source code blocks, and horizontal rules don't have distinct background subtle tints or margins.
4. **Metadata Overload:** Properties, tags, and drawer lines take up equal visual weight as your actual prose.

---

### How to Make Org Mode Look Obsidian-Level Clean in Terminal

Drop these settings into your Emacs `init.el` or Org package configuration:

#### 1. Hide Header Stars & Enable Clean Outline View

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

#### 2. Install `org-superstar` (Modern Bullet Icons)

Replaces default Markdown/Org heading asterisks with sleek UTF-8 glyphs that match Obsidian’s visual hierarchy.

```elisp
(use-package org-superstar
  :ensure t
  :ghook 'org-mode-hook
  :custom
  ;; Custom bullet glyphs for levels 1 through 5
  (org-superstar-headline-bullets-list '("◉" "○" "✸" "✿" "•"))
  (org-superstar-special-item-bullet-alist '((?- . "•") (?+ . "➤"))))

```

#### 3. Modern Callouts & Syntax Highlighting

To get Obsidian-style blockquotes and styled callouts (`> [!NOTE]`), use **`org-modern`**. It restyles headers, tags, tables, and code blocks directly inside terminal buffers using clean Unicode blocks:

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

#### 4. Auto-Display Inline Images in Terminal

Ensure graphics/images render inside Ghostty/terminal Emacs if you use modern terminal graphics protocols:

```elisp
(setq org-startup-with-inline-images t)
(setq org-image-actual-width '(300))

```

---

### Quick Comparison

| Feature | Vanilla Terminal Org | Configured Modern Org | Obsidian |
| --- | --- | --- | --- |
| **Headers** | `*** Header Text` | ` ✸ Header Text` | Large Proportional Text |
| **Bullets** | `- Item` | ` • Item` | ` • Item` |
| **Emphasis** | `*bold text*` | **bold text** | **bold text** |
| **Tables** | ` | Raw | ASCII |

---

Would you like to share your current `init.el` or Org config block so we can see what packages you're running and tweak them directly?

#org #obsidian #looks #visuals #doom #emacs #org-mode #mode #org-superstar #visuals #aesthetic #theme #settings 
