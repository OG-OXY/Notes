The absolute cleanest, most idiomatic way to write a `flake-parts` setup that exports **reusable NixOS modules** (`flake.nixosModules`) populated with **custom packages built in `perSystem`** uses `modulesWithSystem` and `inputsWithSystem`.

Here is the entire pattern, followed by a breakdown of what it solves, why it works, and how it compares to traditional approaches.

### The Complete Implementation Pattern

In this architecture, `flake.nix` orchestrates everything using `flake-parts`. It exposes a custom package in `perSystem`, wraps a NixOS module using `modulesWithSystem`, and wires host configurations cleanly.

#### 1. `flake.nix` (Root)

{
  description = "Modular dendritic system with flake-parts";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    flake-parts.url = "github:hercules-ci/flake-parts";

    # Example external inputs
    hyprland.url = "github:hyprwm/Hyprland";
    ghostty.url = "github:ghostty-org/ghostty";
  };

  outputs = inputs@{ flake-parts, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      systems = [ "x86_64-linux" "aarch64-linux" ];

      imports = [
        ./parts/packages.nix
        ./parts/modules.nix
        ./parts/hosts.nix
      ];
    };
}

#### 2. `parts/packages.nix` (Defining Custom Packages & Overlays)

# parts/packages.nix
{ inputs, ... }: {
  perSystem = { pkgs, system, ... }: {
    # Instantiate custom pkgs set with allowUnfree enabled for perSystem
    _module.args.pkgs = import inputs.nixpkgs {
      inherit system;
      config.allowUnfree = true;
    };

    # Export custom packages built natively for each architecture
    packages = {
      my-custom-script = pkgs.writeShellScriptBin "my-script" ''
        echo "Running custom tool compiled for ${system}"
      '';
    };
  };
}

#### 3. `parts/modules.nix` (Creating System-Agnostic Modules with `modulesWithSystem`)

This is where the magic happens. `modulesWithSystem` creates a NixOS module that has access to `self'` and `inputs'` **before** the system architecture is evaluated by a host.

# parts/modules.nix
{ inputs, options, ... }: {
  flake.nixosModules = {
    # Use modulesWithSystem to bind perSystem args (self', inputs') directly into the module
    desktop-features = { config, lib, pkgs, ... }: {
      imports = [
        # Binds inputs' and self' context directly to the NixOS module definition
        (inputs.flake-parts.lib.modulesWithSystem { inherit inputs options; }
          ({ self', inputs', ... }: {
            
            # Sub-module definition consuming system-bound packages safely
            environment.systemPackages = [
              # Custom package built in perSystem (self'.packages)
              self'.packages.my-custom-script

              # Packages from inputs without manually specifying system strings
              inputs'.ghostty.packages.default
              inputs'.hyprland.packages.hyprland
            ];

            # Set unfree permissions across host evaluation
            nixpkgs.config.allowUnfree = true;
          })
        )
      ];
    };
  };
}

#### 4. `parts/hosts.nix` (Declaring Hosts with `lib.inputsWithSystem`)

# parts/hosts.nix
{ inputs, self, ... }: {
  flake.nixosConfigurations = {
    desktop = 
      let
        system = "x86_64-linux";
        # Manually bind inputs to a specific system string when instantiating raw lib function
        inputs' = inputs.flake-parts.lib.inputsWithSystem system inputs;
        self' = self.perSystem.${system};
      in
      inputs.nixpkgs.lib.nixosSystem {
        inherit system;
        
        # Inject self' and inputs' into all sub-modules via specialArgs
        specialArgs = { inherit self' inputs'; };

        modules = [
          # Import the self-contained module created via modulesWithSystem
          self.nixosModules.desktop-features

          # Force entire system to use perSystem's configured pkgs
          { nixpkgs.pkgs = self'.pkgs; }

          ../hosts/desktop
        ];
      };
  };
}

### What Are These Tools & What Do They Do?

#### `lib.inputsWithSystem`

- **What it is:** A helper function (`inputsWithSystem system inputs`) that takes your global `inputs` set and a `system` string (e.g., `"x86_64-linux"`), and transforms every input into its system-bound version (`inputs'`).
- **What it solves:** Instead of writing `inputs.ghostty.packages.x86_64-linux.default` inside host instantiations or special arguments, it creates `inputs'.ghostty.packages.default`. It strips the need to manually pass architecture strings into sub-evaluations.

#### `modulesWithSystem`

- **What it is:** A higher-order function provided by `flake-parts` that constructs a standard NixOS (or Home Manager) module, automatically injecting `self'`, `inputs'`, `pkgs`, and `system` into the module's scope.
- **What it solves:** Normally, a reusable module in `flake.nixosModules.myModule` **has no idea what system it is running on** when it is defined. Without `modulesWithSystem`, you cannot reference `self'.packages.myPackage` inside a standalone module because `self'` requires a system context that hasn't been evaluated yet. `modulesWithSystem` bridges this gap seamlessly.

### Comparison: Traditional Config vs. Advanced `flake-parts`

|Dimension|Traditional Flake Config|Advanced `flake-parts` (`modulesWithSystem`)|
|---|---|---|
|**System Strings**|Hardcoded `.x86_64-linux` scattered across deep sub-modules.|**Zero architecture strings** inside sub-modules (`self'.packages.x86_64-linux` becomes `self'.packages.app`).|
|**Exporting Modules**|Exposing `flake.nixosModules` requires consumers to pass `pkgs` or manually configure inputs.|`flake.nixosModules` exports **self-contained modules** that bundle custom packages automatically.|
|**Package Instantiation**|`nixpkgs` is evaluated multiple times (once for system, once for home-manager, once for devShells).|`nixpkgs` is evaluated **once per architecture** in `perSystem` and shared globally.|
|**Unfree / Overlays**|Frustrating scope bugs where `allowUnfree` fails inside sub-modules or dendritic imports.|Overlays and unfree configs attached to `perSystem.pkgs` inherit everywhere automatically.|
|**Multi-Arch Support**|Adding ARM (`aarch64-linux`) requires refactoring paths and `specialArgs`.|Cross-architecture building works out of the box without changing module code.|

### Why Is This Leveling Up Your Config?

1. **True Module Encapsulation:** If you publish or export `flake.nixosModules.desktop-features` for another machine or user to consume, it brings its dependencies along with it. The module knows how to fetch its custom `packages` from `self'` and flake inputs from `inputs'` without relying on the host system defining those arguments in `specialArgs`.
2. **Eliminates Double Evaluation:** Nix doesn't waste time or memory evaluating `import inputs.nixpkgs` multiple times. The `pkgs` built inside `perSystem` becomes the single source of truth for your system packages, home-manager packages, and dev shells.
3. **Clean Dendritic Refactoring:** You can create 50 small, hyper-specific module files in a dendritic tree. Every single module can cleanly pull `self'.packages.<name>` or `inputs'.<name>.packages.<pkg>` without adding boilerplates or importing `inputs` in every header.