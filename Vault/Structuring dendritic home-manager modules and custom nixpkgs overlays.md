### 1. Structuring Dendritic Home Manager Modules

In a dendritic structure, modules avoid giant monolith files by branching into atomic, self-contained units organized by domain, feature, or tool. Each module exposes a option toggle (or imports cleanly without side effects) and consumes `self'` and `inputs'` via `extraSpecialArgs`.

#### Directory Tree

├── flake.nix
├── parts/
│   ├── nixos.nix            # System-level configurations
│   └── home.nix             # Home Manager instantiations
└── modules/
    └── home/
        ├── default.nix      # Dendritic aggregator importing submodules
        ├── core/            # System-agnostic essentials
        │   ├── fish.nix
        │   ├── ghostty.nix
        │   └── git.nix
        ├── desktop/         # Graphical environment & window management
        │   ├── hyprland.nix
        │   └── waybar.nix
        └── editors/         # Development environments
            └── neovim.nix

#### The Submodule Pattern

Keep individual modules pure and isolated. Avoid hardcoded package sets; pull directly from `pkgs`, `self'`, or `inputs'`:

# modules/home/core/ghostty.nix
{ config, lib, pkgs, inputs', ... }:

let
  cfg = config.features.ghostty;
in {
  options.features.ghostty = {
    enable = lib.mkEnableOption "Ghostty terminal emulator";
  };

  config = lib.mkIf cfg.enable {
    home.packages = [
      # Consumes inputs' bound to current host architecture automatically
      inputs'.ghostty.packages.default
    ];

    # Modular program configurations
    programs.ghostty = {
      enable = true;
      enableFishIntegration = true;
    };
  };
}

#### The Aggregator (`modules/home/default.nix`)

The root dendritic entrypoint collects every branch and exposes default enablement or feature flags:

# modules/home/default.nix
{ config, lib, ... }:

{
  imports = [
    ./core/fish.nix
    ./core/ghostty.nix
    ./core/git.nix
    ./desktop/hyprland.nix
    ./editors/neovim.nix
  ];

  # Toggle default dendritic feature trees
  features = {
    ghostty.enable = lib.mkDefault true;
    fish.enable = lib.mkDefault true;
  };
}

### 2. Sharing Custom Nixpkgs Overlays Across Modules & Home Manager

To guarantee that custom patches, overrides, or custom package additions are visible across **both** NixOS system modules and Home Manager modules without evaluating `nixpkgs` multiple times, configure your overlays inside `perSystem` and pass the resulting `pkgs` instance down.

#### Defining Overlays in `perSystem` (`parts/overlays.nix` or `flake.nix`)

Construct `pkgs` once inside `perSystem`, applying custom overlays alongside `allowUnfree`:

# parts/nixpkgs.nix
{ inputs, ... }: {
  perSystem = { system, ... }: {
    _module.args.pkgs = import inputs.nixpkgs {
      inherit system;
      config = {
        allowUnfree = true;
        allowUnfreePredicate = _: true;
      };
      overlays = [
        # Example 1: Add custom packages or inputs as overlays
        (final: prev: {
          myCustomTool = prev.callPackage ../pkgs/myCustomTool { };
        })

        # Example 2: Apply patches or overrides to existing packages
        (final: prev: {
          mpv = prev.mpv.override {
            youtubeSupport = true;
          };
        })
      ];
    };
  };
}

#### Binding the Overlaid `pkgs` Set into NixOS and Home Manager

When instantiating the machine inside `flake.nixosConfigurations` (or `flake.homeConfigurations`), force both NixOS and Home Manager to consume `self'.pkgs`.

By setting `home-manager.useGlobalPkgs = true;`, Home Manager inherits the exact system `pkgs` instance (including all overlays and `allowUnfree` options) without needing to re-evaluate its own package set.

# parts/nixos.nix
{ inputs, ... }: {
  flake.nixosConfigurations.desktop =
    let
      system = "x86_64-linux";
      # Pull the pre-configured pkgs instance directly from perSystem
      self' = inputs.self.perSystem.${system};
      inputs' = inputs.flake-parts.lib.inputsWithSystem system inputs;
    in
    inputs.nixpkgs.lib.nixosSystem {
      inherit system;
      specialArgs = { inherit self' inputs'; };
      modules = [
        # Force entire NixOS module evaluation to use perSystem's pkgs
        { nixpkgs.pkgs = self'.pkgs; }

        # Integrate Home Manager
        inputs.home-manager.nixosModules.home-manager
        {
          home-manager = {
            # IMPLICITLY SHARES OVERLAYS:
            # Tells HM to use NixOS's nixpkgs instance (which has our overlays applied)
            useGlobalPkgs = true;
            useUserPackages = true;

            extraSpecialArgs = { inherit self' inputs'; };

            users.ty = {
              imports = [ ../modules/home ];
            };
          };
        }

        ../hosts/desktop
      ];
    };
}

### Why This Architectural Pattern Works Cleanly

1. **Zero Double-Evaluation:** Nixpkgs is instantiated exactly once per architecture inside `perSystem`. Rebuilding your system or home environment won't re-download or re-evaluate package trees twice.
2. **Global Overlay Visibility:** Packages overridden in your overlays (e.g., `pkgs.mpv`) are updated everywhere—whether referenced inside a NixOS system package list or deeply nested inside a Home Manager module like `modules/home/desktop/waybar.nix`.
3. **No Unfree Flakiness:** Because `allowUnfree = true` and `overlays` are attached to the root `pkgs` object via `nixpkgs.pkgs = self'.pkgs;`, you will never encounter scope errors or unfree license evaluation rejections inside submodules.