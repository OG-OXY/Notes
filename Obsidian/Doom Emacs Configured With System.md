`nix-doom-emacs-unstraightened` exposes a **Home Manager module (`homeModule`)**, not a top-level NixOS system module (`nixosModules.default`).

Nix threw `attribute 'nixosModules' missing` because `inputs.nix-doom-emacs-unstraightened` does not export an `nixosModules` attribute set.

You have two clean options depending on how your dendritic flake handles modules:

---

### Option 1: Wire it through Home Manager (Recommended)

In your Home Manager module import list, import `homeModule`:

#### In your Home Manager user module (`home.nix` or your user module):

```nix
{ inputs, pkgs, ... }:

{
  imports = [
    inputs.nix-doom-emacs-unstraightened.homeModule
  ];

  programs.doom-emacs = {
    enable = true;
    doomDir = ./doom.d; # Directory holding init.el, config.el, packages.el
    provideEmacs = true; # Generates emacs, emacsclient, and doom-emacs binaries
  };

  # Optional: Enable user-level Emacs daemon service so emacsclient connects instantly
  services.emacs = {
    enable = true;
    client.enable = true;
  };
}

```

---

### Option 2: System-Level Package via Overlay (No Home Manager)

If you aren't using Home Manager and want to install Doom Emacs directly in `environment.systemPackages` at the system level, use `nix-doom-emacs-unstraightened`'s overlay.

#### 1. Apply the Overlay in your system config:

```nix
{ inputs, pkgs, ... }:

{
  nixpkgs.overlays = [
    inputs.nix-doom-emacs-unstraightened.overlays.default
  ];

  environment.systemPackages = [
    # Installs emacsWithDoom (which includes emacs, emacsclient, and doom binaries)
    (pkgs.emacsWithDoom {
      doomDir = ./doom.d;
      doomLocalDir = "~/.local/share/nix-doom";
    })
  ];
}

```

---
[[Oct 20 TODO]]