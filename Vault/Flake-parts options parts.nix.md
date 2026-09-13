In `flake-parts`, the top-level configuration options live within the context of `mkFlake` (or any flake-parts module). They broadly break down into three categories:

1. **Standard Flake Outputs** (`flake` options like `nixosConfigurations`, `overlays`, `templates`, etc.)
2. **System Matrix Controls** (`systems`, `perSystem`, `modulesWithSystem`)
3. **Flake-Parts Metaprogramming & Extensions** (`imports`, `debug`, `partition`, option declarations, etc.)

### Every Major Top-Level Flake-Parts Option

The following `parts.nix` contains every native top-level option exposed by `flake-parts` (along with core ecosystem options), populated simultaneously to show syntax and available attributes.

{ inputs, config, lib, options, flake-parts-lib, specialArgs, ... }:

{
  # =========================================================================
  # 1. MODULE SYSTEM IMPORTS & META
  # =========================================================================
  
  # External flake-parts modules or sub-files to bring into the evaluation context
  imports = [
    # inputs.flake-parts.flakeModules.modulesWithSystem
    # ./other-part.nix
  ];

  # Enables internal debug traces during flake evaluation
  debug = true;

  # Define custom top-level options for your own flake-parts schema
  options = {
    myCustomGlobalOption = lib.mkOption {
      type = lib.types.str;
      default = "hello";
      description = "Custom top-level variable for internal flake logic";
    };
  };

  # Set values for custom or imported options
  config = {
    myCustomGlobalOption = "world";
  };

  # =========================================================================
  # 2. SYSTEM MATRIX CONFIGURATION
  # =========================================================================

  # Defines target architectures for perSystem (packages, devShells, apps, etc.)
  systems = [
    "x86_64-linux"
    "aarch64-linux"
    "x86_64-darwin"
    "aarch64-darwin"
  ];

  # Module evaluated for EVERY system listed in `systems`
  perSystem = { config, self', inputs', pkgs, system, lib, ... }: {
    # Custom options declared specifically inside the perSystem submodule
    options = {
      myPerSystemOption = lib.mkOption {
        type = lib.types.bool;
        default = true;
      };
    };

    config = {
      # Custom nixpkgs evaluation settings for this system
      _module.args.pkgs = import inputs.nixpkgs {
        inherit system;
        config.allowUnfree = true;
      };

      # System-dependent Flake Outputs
      packages = {
        default = pkgs.hello;
        custom-app = pkgs.stdenv.mkDerivation {
          name = "custom";
          src = ./.;
        };
      };

      apps = {
        default = {
          type = "app";
          program = "${config.packages.default}/bin/hello";
        };
      };

      devShells = {
        default = pkgs.mkShell {
          buildInputs = [ pkgs.git pkgs.nixpkgs-fmt ];
        };
      };

      checks = {
        package-builds = config.packages.default;
      };

      formatter = pkgs.nixpkgs-fmt;

      legacyPackages = pkgs;
    };
  };

  # Helper function to inject system-specific attributes (self', pkgs) into global modules
  modulesWithSystem = { self', inputs', pkgs, system, ... }: {
    # Commonly used when declaring nixosConfigurations that need self'.packages.${system}
  };

  # Advanced multi-evaluation logic (used for isolating Darwin/Linux or hydra targets)
  partition = { ... }: {
    # Partitioning rules go here
  };

  # =========================================================================
  # 3. GLOBAL FLAKE OUTPUTS (`flake.*`)
  # =========================================================================
  # Attributes declared here bypass system matrices and map directly to `flake.outputs`
  
  flake = {
    # --- System Configurations ---
    nixosConfigurations = {
      my-host = inputs.nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [ ./hosts/desktop.nix ];
      };
    };

    darwinConfigurations = {
      my-mac = inputs.darwin.lib.darwinSystem {
        system = "aarch64-darwin";
        modules = [ ./hosts/macbook.nix ];
      };
    };

    homeConfigurations = {
      "user@host" = inputs.home-manager.lib.homeManagerConfiguration {
        pkgs = inputs.nixpkgs.legacyPackages.x86_64-linux;
        modules = [ ./home.nix ];
      };
    };

    # --- Reusable Modules ---
    nixosModules = {
      default = ./modules/nixos/default.nix;
      feature = { config, lib, pkgs, ... }: { };
    };

    darwinModules = {
      default = ./modules/darwin/default.nix;
    };

    homeManagerModules = {
      default = ./modules/home/default.nix;
    };

    flakeModules = {
      default = ./modules/parts.nix;
    };

    # --- Overlays & Extension Points ---
    overlays = {
      default = final: prev: {
        my-package = prev.callPackage ./package.nix { };
      };
    };

    # --- Templates & Development ---
    templates = {
      default = {
        path = ./template;
        description = "Standard dendritic flake template";
      };
    };

    # Raw arbitrary flake attributes
    lib = {
      myCustomHelper = x: x + 1;
    };
  };
}

### Core Structural Breakdown

- `**systems**`: Sets the string list of platforms (`x86_64-linux`, etc.) that `perSystem` will iterate through.
- `**perSystem**`: Generates all system-bound flake outputs (`packages.${system}`, `devShells.${system}`, `apps.${system}`, `checks.${system}`, `formatter.${system}`).
- `**flake**`: Generates all top-level, non-system-bound outputs (`nixosConfigurations`, `overlays`, `nixosModules`, `lib`, `templates`).
- `**modulesWithSystem**`: Provided by `flake-parts.flakeModules.modulesWithSystem`. Bridges the gap so `flake.nixosConfigurations` can reference `self'.packages` without hardcoding system strings.
- **`imports` / `options` / `config`**: Leverages the standard Nix module system inside the flake level, allowing you to split your `flake.nix` into multiple modular files (like dendritic parts).