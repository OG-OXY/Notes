The reason Nix is complaining about `allowUnfree` even after you put it in `config.nix`, `default.nix`, and `hardware.nix` is that **NixOS system builds evaluate a completely separate instance of `nixpkgs`** from the `perSystem.pkgs` defined in `parts.nix`.

If you set `_module.args.pkgs = import inputs.nixpkgs { inherit system; config.allowUnfree = true; };` inside `parts.nix`, that **only** fixes `allowUnfree` for `perSystem.packages` (like wrapped CLI/GUI tools). It **does not** touch the `pkgs` instance evaluated inside `nixosConfigurations`.

In NixOS, when you load the kernel module or packages for NVIDIA (`hardware.nvidia.package`, `services.xserver.videoDrivers = [ "nvidia" ]`), NixOS evaluates its internal `pkgs` parameter. If that module tree isn't explicitly told to allow unfree software, it throws the error.

### The 2 Fixes (Choose One)

#### Option A: Set `nixpkgs.config.allowUnfree = true;` inside `modulesWithSystem` (Recommended)

When using `modulesWithSystem` to generate your `nixosConfigurations`, you need to explicitly pass the unfree option inside the module block for that host.

In your host setup (or inside `modules/hosts/desktop.nix`):

{ modulesWithSystem, inputs, ... }:

{
  flake.nixosConfigurations.desktop = modulesWithSystem (
    { self', pkgs, system, ... }:
    {
      system = system;
      modules = [
        # 1. THIS IS THE CRITICAL LINE FOR NVIDIA / SYSTEM UNFREE:
        {
          nixpkgs.config.allowUnfree = true;
        }

        # 2. Your driver and hardware setup
        {
          services.xserver.videoDrivers = [ "nvidia" ];
          hardware.nvidia = {
            modesetting.enable = true;
            open = false; # Proprietary NVIDIA driver is unfree
          };

          environment.systemPackages = [
            self'.packages.kitty
          ];
        }
      ];
    }
  );
}

#### Option B: Force NixOS to use `perSystem`'s `pkgs` directly

If you want NixOS to reuse the exact `pkgs` instance configured in `parts.nix` (which already has `allowUnfree = true` set), pass `pkgs` directly into `nixosSystem` or `modulesWithSystem`:

{ modulesWithSystem, ... }:

{
  flake.nixosConfigurations.desktop = modulesWithSystem (
    { self', pkgs, ... }:
    {
      # Pass perSystem's pkgs directly so NixOS doesn't re-evaluate nixpkgs
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

### Why putting it in `config.nix` / `default.nix` failed previously

If your `config.nix` or `default.nix` contained:

{ pkgs, ... }: {
  nixpkgs.config.allowUnfree = true;
}

...but was imported _outside_ or _before_ the `nixosConfigurations` module list evaluated, or if `pkgs` was hard-overridden at the flake level via `specialArgs`, the module system dropped or failed to merge `nixpkgs.config`.

Putting `{ nixpkgs.config.allowUnfree = true; }` directly inside the `modules = [ ... ]` list passed into `modulesWithSystem` guarantees it evaluates during host compilation.

It really is a slick architecture once all those moving pieces click together! Having every app wrapped with its own config into a portable binary—while `modulesWithSystem` cleanly bridges `pkgs` and `allowUnfree` to your host—gives you a incredibly clean, zero-state setup.

Have fun wiring it up! Let me know if any weird evaluation traces or missing binary PATH issues pop up when you run your first system rebuild.
[[Flake-parts options parts.nix]]
[[580.xx.xx cachyos lto thin]]