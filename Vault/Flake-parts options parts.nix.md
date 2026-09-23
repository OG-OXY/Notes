Here is the fully formatted Obsidian note. The Nix code is now properly indented inside a single block, ASCII header dividers have been converted into clean internal code comments, and the surrounding text uses standard Obsidian callouts and bold headers for scanning.

```markdown
# Flake-Parts Options (`parts.nix`)

In `flake-parts`, top-level configuration options live within the context of `mkFlake` (or any `flake-parts` module). They broadly break down into three core categories:

1. **Standard Flake Outputs:** Global `flake.*` options like `nixosConfigurations`, `overlays`, `templates`, etc.
2. **System Matrix Controls:** `systems`, `perSystem`, and `modulesWithSystem`.
3. **Metaprogramming & Extensions:** `imports`, `debug`, `partition`, and custom `option` declarations.

> [!info] Exhaustive Template
> The following `parts.nix` contains every native top-level option exposed by `flake-parts` alongside common ecosystem integrations, fully populated to illustrate syntax and available sub-attributes.

---

## Complete `parts.nix` Reference

```nix
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
  # 3. GLOBAL FLAKE OUTPUTS (flake.*)
  # =========================================================================

  # Attributes declared here bypass system matrices and map directly to flake.outputs
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

```

---

## Core Structural Breakdown

* **`systems`**: List of system triples (`x86_64-linux`, etc.) that `perSystem` will iterate through to evaluate system-bound outputs.
* **`perSystem`**: Generates all system-bound outputs (`packages.${system}`, `devShells.${system}`, `apps.${system}`, `checks.${system}`, `formatter.${system}`).
* **`flake`**: Declares top-level, non-system-bound outputs directly (`nixosConfigurations`, `overlays`, `nixosModules`, `lib`, `templates`).
* **`modulesWithSystem`**: Provided by `flake-parts.flakeModules.modulesWithSystem`. Bridges system-specific artifacts into global modules so `flake.nixosConfigurations` can reference `self'.packages` cleanly.
* **`imports` / `options` / `config**`: Leverages the standard Nix module system at the flake evaluation level, enabling multi-file modular layouts (dendritic patterns).
  
  [[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]