# Custom Emacs Packages NixOS Module

You are 100% correct, and the reason it feels backwards is because Doom Emacs
isn't actually a normal Emacs package.

Unlike evil or doom-themes, which are single, self-contained libraries
published to global repositories like MELPA, the Doom dashboard is hardcoded
inside Doom's massive monolithic source code repository. It relies on internal
Doom function hooks, custom font systems, and macros that vanilla Emacs cannot
understand out of the box.

Because of this, there isn't a simple epkgs.doom-dashboard available in Nixpkgs.

However, if you want a clean, declarative solution that completely avoids
writing long configuration scripts, you can fetch the community-made
standalone port of the dashboard using a Nix Flake input and inject it
straight into your package list.

## Here is how to do it natively and cleanly

1. Add the Dashboard to your Flake Inputs

Add the repository directly to your system flake.nix:

````nix
inputs = {
  nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
  
  # Grab the standalone port of the Doom Dashboard directly
  doom-dashboard-src = {
    url = "github:emacs-dashboard/doom-dashboard";
    flake = false;
  };
};

````

2. Build the Package and Add it to emacsWithPackages

In your NixOS module configuration where you declare your system packages,
use trivialBuild to compile it cleanly on system switch without relying on
mutable Emacs download configurations:

````nix
{ config, pkgs, inputs, ... }:

let
  # Declaratively compile the package from the git repository source
  doom-dashboard = pkgs.emacsPackages.trivialBuild {
    pname = "doom-dashboard";
    version = "latest";
    src = inputs.doom-dashboard-src;
    packageRequires = [ pkgs.emacsPackages.dashboard ];
  };
in
{
  environment.systemPackages = [
    (pkgs.emacsWithPackages (epkgs: [
      epkgs.dashboard
      epkgs.nerd-icons
      doom-dashboard  # Injected cleanly into your environment
    ]))
  ];
}
````

3. All you need in your init.el

Now that Nix handles the build cleanly behind the scenes, your actual Emacs
configuration drops down to just a few lines:

````elisp
(use-package dashboard
  :after nerd-icons
  :config
  (setq dashboard-display-icons-p t)
  (setq dashboard-icon-type 'nerd-icons)
  (dashboard-setup-startup-hook))

(use-package doom-dashboard
  :after dashboard
  :demand t)

````

This gets rid of the script bloat and treats it exactly like a native system package.
Would you like help tracking down any missing dependency errors during the system
rebuild, or should we verify your nh os switch mapping setup to ensure the new
input hooks register correctly?

# Eager vs. Lazy Loading

## What "Eager Loading at Startup" Means

*Eager loading* (or loading unconditionally at startup) means that the moment
Emacs boots up and reads your `init.el`, it immediately compiles, evaluates,
and loads the entire package's code into active memory.

- *The Cost:* Every package you eagerly load adds to your Emacs cold-start
boot time.
If you eagerly load 50 heavy packages, your startup time crawls.
- *The Benefit:* It is guaranteed to be ready. There is zero delay or
"autoload stub" overhead the first time you try to use it.
*(Note: If you run Emacs as a background daemon for your Ghostty/herdr splits,
cold-start boot time matters less day-to-day, but keeping it lean is still best practice).*

---

## How to Decide Eager vs. Lazy Loading

You make the decision based on **when** the package needs to be active in
order for your workflow not to break.

### 1. Eager Load (`:demand t`) — Use Sparingly

You eager-load a package **only** if it meets one of these strict criteria:

- *Global Minor Modes:* It fundamentally alters how every single buffer
behaves globally from the second you open a window (e.g., `helix-mode`, global
line numbers, global completion backends).
- *Themes & UI Layouts:* Visual frameworks that need to paint the screen
immediately so you don't get a flash of unstyled white/default text on startup
(e.g., `doom-one`, your dashboard).

### 2. Lazy Load (`:bind`, `:hook`, `:commands`, `:mode`) — Default to This

You lazy-load everything else. `use-package` makes this automatic by creating
lightweight placeholders. The package code stays unloaded until a specific
trigger occurs:

- *Keybindings (`:bind`):* Magit doesn't load into memory until you hit `C-x g`.
- *Hooks (`:hook`):* `org-modern` or `org-appear` don't load until you
actually open an `.org` file and enter `org-mode`.
- *File Extensions (`:mode`):* Markdown or Python linters don't load until you
open a file ending in `.md` or `.py`.

---

## How This Applies to Your `init.el`

Looking at your clean configuration architecture, here is how your packages
naturally divide:

- *Eager Loaded (`:demand t` or global init):*
- `helix-mode` (Needs to hook into every buffer instantly).
- `vertico` / `corfu` (Global completion engine).
- Theme & UI toggles.


- *Lazy Loaded (Deferred via hooks/binds):*
- `magit` (Loads only on `C-x g`).
- `org-roam` / `org-modern` (Loads only when entering `org-mode`).
- `pandoc-mode` (Loads only when markdown/org files trigger the hook).
