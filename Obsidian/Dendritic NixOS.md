To build a pure **Dendritic Nix** architecture using `**flake-parts**`, `**import-tree**`, and `**nix-wrapper-modules**`, you need to shift away from standard top-down host configurations.

Instead of configuring applications _inside_ a NixOS module or Home Manager, every tool becomes a **self-contained, standalone wrapped package** under `perSystem.packages`. NixOS host modules then simply pull in these pre-wrapped packages.

Here is the exact structural blueprint to tie all four pieces together cleanly.

### Key Architectural Pattern

1. `**flake.nix**`: The entrypoint that invokes `flake-parts.lib.mkFlake` and passes `./modules` into `import-tree`.
2. `**modules/parts.nix**`: Declares target `systems` and shared `flake-parts` top-level options.
3. `**modules/features/<app>.nix**`: Self-contained module files defining wrapped application derivations via `nix-wrapper-modules`.
4. `**modules/hosts/<host>.nix**`: NixOS system configs that import features or bundle wrapped packages.

### Step-by-Step Implementation

#### 1. Flake Entrypoint (`flake.nix`)

Keep `flake.nix` down to bare inputs and a single call to `mkFlake` using `import-tree`.

{
  description = "Dendritic Nix Flake Architecture";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

    flake-parts = {
      url = "github:hercules-ci/flake-parts";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    import-tree.url = "github:robertodauria/import-tree";

    wrapper-modules = {
      url = "github:viperML/nh-wrapper-modules"; # or nix-wrapper-modules
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = inputs@{ flake-parts, import-tree, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      # Recursively imports all .nix files under ./modules as flake-parts modules
      imports = [ (import-tree ./modules) ];
    };
}

#### 2. Declarative System Targets (`modules/parts.nix`)

This file defines system architectures and standard module parameters.

{ inputs, ... }:

{
  imports = [
    inputs.flake-parts.flakeModules.modulesWithSystem
  ];

  systems = [ "x86_64-linux" "aarch64-linux" ];

  perSystem = { pkgs, system, ... }: {
    # Custom unfree package overrides or overlays if needed
    _module.args.pkgs = import inputs.nixpkgs {
      inherit system;
      config.allowUnfree = true;
    };
  };
}

#### 3. Feature Wrappers (`modules/features/kitty.nix`)

Each application file defines its own wrapped package output inside `perSystem.packages`.

By wrapping the config directly into the executable, `nix run .#kitty` will run Kitty with your complete dotfiles on **any** Linux machine without Home Manager or local `~/.config/kitty` files.

{ inputs, ... }:

{
  perSystem = { pkgs, self', ... }: {
    packages.kitty = inputs.wrapper-modules.lib.makeWrapper {
      inherit pkgs;
      package = pkgs.kitty;

      # Bundle runtime dependencies directly into PATH for this binary
      prefixPATH = [
        pkgs.nerd-fonts.jetbrains-mono
      ];

      # Flag pointing directly to a store-bound config file
      flags = [
        "--config"
        "${pkgs.writeText "kitty.conf" ''
          font_family      JetBrainsMono Nerd Font
          font_size        11.0
          background_opacity 0.95
          confirm_os_window_close 0
        ''}"
      ];
    };
  };
}

#### 4. Complex Feature Wrapper with External Inputs (`modules/features/hyprland.nix`)

For desktop environments or shell environments that depend on other wrapped packages from `self'`:

{ inputs, ... }:

{
  perSystem = { pkgs, self', inputs', ... }: {
    packages.hyprland-wrapped = inputs.wrapper-modules.lib.makeWrapper {
      inherit pkgs;
      # Use upstream hyprland binary from flake inputs
      package = inputs'.hyprland.packages.default;

      # Inject your custom wrapped packages into Hyprland's runtime PATH
      prefixPATH = [
        self'.packages.kitty          # Custom wrapped Kitty
        pkgs.wofi
        pkgs.waybar
      ];

      flags = [
        "--config"
        "${./hyprland.conf}"         # Direct path to local config in repo
      ];
    };
  };
}

#### 5. Exposing Systems via `modulesWithSystem` (`modules/hosts/desktop.nix`)

To pull these wrapped packages into actual NixOS machine configurations, use `modulesWithSystem` to access `self'` outputs per architecture.

{ modulesWithSystem, ... }:

{
  flake.nixosConfigurations.desktop = modulesWithSystem (
    { self', pkgs, ... }:
    {
      system = "x86_64-linux";
      modules = [
        # System base configurations
        {
          boot.loader.systemd-boot.enable = true;
          networking.hostName = "desktop";

          # Inject pre-wrapped applications into system environment
          environment.systemPackages = [
            self'.packages.kitty
            self'.packages.hyprland-wrapped
          ];
        }
      ];
    }
  );
}

### Core Mechanics of This Setup

- **No `specialArgs` Spaghetti:** `import-tree` automatically sweeps all modules into `flake-parts`. `flake-parts` handles feeding `inputs`, `self'`, and `pkgs` everywhere seamlessly.
- **Hermetic & Instant Runnable:** Every app inside `./modules/features/` can be invoked standalone from remote machines via `nix run github:yourname/dotfiles#<app>`.
- **Zero Dotfile Symlinking:** App settings exist solely in the Nix store as wrapped flags/arguments attached to individual package wrappers, completely bypassing `~/.config` state conflicts.

[[Flake-parts options parts.nix]]
[[Flake parts fixes -system-]]