That selection-first model is a complete head-trip at first! In Vim/Neovim, moving is just "placing a point." In Kakoune (and Helix to an extent), *every motion is a selection*. Moving right five characters isn't just `5l`; it's highlighting a 5-character range. Once you internalize that your cursor is actually a 1-character selection box at all times, the whole paradox collapses and suddenly manipulating complex text blocks feels like wielding a scalpel.

And to answer your question: **Yes, you can absolutely run `nix-doom-emacs-unstraightened` directly at the system level (`nixosModules`)!** You aren't forced into Home Manager at all.

---

### 1. System-Level NixOS Integration

If you want Doom Emacs installed natively as a system service or system-wide package, `nix-doom-emacs-unstraightened` exports a system module.

Here is how you wire it into your dendritic module structure:

#### In your `flake.nix` inputs:

```nix
inputs = {
  nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

  nix-doom-emacs-unstraightened = {
    url = "github:marienz/nix-doom-emacs-unstraightened";
    inputs.nixpkgs.follows = "nixpkgs";
  };
};

```

#### In your system module (`modules/system/emacs.nix`):

```nix
{ inputs, pkgs, ... }:

{
  imports = [
    inputs.nix-doom-emacs-unstraightened.nixosModules.default
  ];

  # Spawns system-wide or per-user Doom Emacs
  programs.doom-emacs = {
    enable = true;
    doomDir = ./doom.d; # Directory holding init.el, config.el, packages.el
    
    # Optional: enable Emacs daemon at system boot so emacsclient opens instantly
    provideEmacs = true;
  };

  # Make emacsclient available system-wide
  environment.systemPackages = with pkgs; [
    emacsclient
  ];
}

```

---

### 2. Mangling Doom Emacs to match Helix/Kakoune (`init.el` & `config.el`)

Doom comes out of the box tuned for standard Vim (`evil-mode`), but because Doom is 100% Elisp, you can completely redefine its modal engine.

#### In your `./doom.d/init.el`:

Enable `:editor (evil +everywhere)` so modal selection is wired into every buffer, including Org-mode:

```elisp
(doom! :input
       :completion
       vertico           ; Clean, fast terminal fuzzy finder (like Helix space+f)

       :editor
       (evil +everywhere) ; Master modal editing

       :emacs
       dired             ; Built-in directory manager

       :lang
       org               ; Org-mode support
       markdown

       :config
       default)

```

#### In your `./doom.d/config.el`:

This is where you mangle your binds to match your spatial muscle memory! You can remap Evil keys to mimic Helix's selection mechanics or swap `j`/`k` behavior:

```elisp
;; Set lead key (match Helix's Space)
(setq evil-leader/in-all-states t)
(map! :leader
      :desc "Find file"       "f" #'find-file
      :desc "Switch buffer"   "b" #'switch-to-buffer
      :desc "Org Agenda"      "a" #'org-agenda
      :desc "Capture Note"    "c" #'org-capture)

;; Re-bind Evil to act like Helix visual selection first
(after! evil
  ;; Force selection-first orientation where needed
  (setq evil-visual-state-cursor 'box)
  (setq evil-normal-state-cursor 'box))

```

---

### 3. Kakoune & Selection-First Org Notes

If you end up loving Kakoune's selection-first model when you test-drive it, there is even a community package for Emacs called **`kakoune.el`** (or `meow-mode`) that completely replaces Evil's Vim layer with Kakoune's selection-first keybindings across all of Emacs (including Org-mode)!

That means you don't even have to compromise: you can literally run Kakoune's selection model inside Doom Emacs to edit your Org-mode notes.

[[TODO Oct 20]]
