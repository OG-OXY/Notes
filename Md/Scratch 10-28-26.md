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
