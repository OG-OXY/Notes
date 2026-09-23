Here is the formatted, clean Obsidian Markdown note using proper syntax formatting, highlighted code blocks, callouts, and clean section breaks matching the style of your prior notes.

```markdown
# Handling `allowUnfree` with `flake-parts` & NixOS

When building NixOS configurations using `flake-parts`, you may encounter unfree package evaluation errors (e.g., NVIDIA drivers, proprietary software) even after setting `allowUnfree = true` in standard system modules.

> [!warning] The Root Cause
> **NixOS system builds evaluate a completely separate instance of `nixpkgs`** from the `perSystem.pkgs` defined in `parts.nix`.
> 
> Setting `_module.args.pkgs = import inputs.nixpkgs { inherit system; config.allowUnfree = true; };` inside `parts.nix` **only** configures `allowUnfree` for `perSystem.packages` (e.g., standalone CLI/GUI tools). It **does not** propagate to the `pkgs` instance evaluated internally by `nixosConfigurations`.

---

## Solutions

### Option A: Declare `nixpkgs.config.allowUnfree` in Host Modules (Recommended)

When using `modulesWithSystem` to declare host configurations, explicitly pass the unfree option inside the module list evaluated by the system:

```nix
{ modulesWithSystem, inputs, ... }:

{
  flake.nixosConfigurations.desktop = modulesWithSystem (
    { self', pkgs, system, ... }:
    {
      inherit system;
      modules = [
        # 1. Enable unfree packages for this host evaluation:
        {
          nixpkgs.config.allowUnfree = true;
        }

        # 2. Hardware / Driver configuration:
        {
          services.xserver.videoDrivers = [ "nvidia" ];
          hardware.nvidia = {
            modesetting.enable = true;
            open = false; # Proprietary driver requires allowUnfree
          };

          environment.systemPackages = [
            self'.packages.kitty
          ];
        }
      ];
    }
  );
}

```

---

### Option B: Pass `perSystem.pkgs` Directly to Host

To force NixOS to reuse the exact `pkgs` instance configured inside `parts.nix` (which already has `allowUnfree = true`), inject `pkgs` into the system definition:

```nix
{ modulesWithSystem, ... }:

{
  flake.nixosConfigurations.desktop = modulesWithSystem (
    { self', pkgs, ... }:
    {
      # Pass perSystem's pkgs directly so NixOS does not re-evaluate nixpkgs
      inherit pkgs;

      modules = [
        ./hardware.nix
        ./config.nix
        {
          services.xserver.videoDrivers = [ "nvidia" ];
        }
      ];
    }
  );
}

```

---

## Structural Breakdown

> [!info] Why standard module imports failed
> If setting `nixpkgs.config.allowUnfree = true;` inside external files (like `config.nix` or `default.nix`) threw errors, it typically occurred because:
> 1. The file was imported outside or before the `nixosConfigurations` module list evaluated.
> 2. `pkgs` was hard-overridden at the flake level using `specialArgs`, causing the NixOS module system to drop or ignore merged `nixpkgs.config` settings.
> 
> 

Inlining `{ nixpkgs.config.allowUnfree = true; }` directly within the `modules = [ ... ]` block evaluated by `modulesWithSystem` guarantees the option is present during host evaluation.

[[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]
[[Flake-parts options parts.nix]]
[[580.xx.xx cachyos lto thin]]