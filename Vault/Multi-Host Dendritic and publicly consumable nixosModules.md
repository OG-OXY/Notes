Here is your Obsidian markdown note, cleaned up with proper syntax, accurate Nix code formatting, and structured headings.

---

# Structuring a Multi-Host Setup with `modulesWithSystem`

When managing multiple hosts (e.g., desktop, laptop, server) in a dendritic layout, you want to avoid duplicating `lib.nixosSystem` boilerplate, hardcoding system architectures (`x86_64-linux`, `aarch64-linux`), or re-evaluating `nixpkgs` for every machine.

By using **flake-parts** alongside `modulesWithSystem`, you centralize `pkgs` evaluation inside `perSystem` and write system-agnostic host profiles that cleanly pull packages from `self'` and `inputs'`.

---

## Directory Structure

```
├── flake.nix
├── parts/
│   ├── pkgs.nix             # Centralized nixpkgs & overlays
│   ├── modules.nix          # Internal modules using modulesWithSystem
│   └── hosts.nix            # Multi-host definitions
├── modules/
│   └── nixos/               # Dendritic NixOS modules
│       ├── core.nix
│       ├── hyprland.nix
│       └── server-base.nix
└── hosts/
    ├── desktop/             # Machine-specific configs
    │   └── default.nix
    ├── laptop/
    │   └── default.nix
    └── home-server/
        └── default.nix

```

---

## 1. Centralized Package Evaluation

Configure `pkgs` once inside `perSystem` so that all hosts and modules share a single, unified package set with `allowUnfree` and custom overlays applied.

### `parts/pkgs.nix`

```nix
{ inputs, ... }: {
  perSystem = { system, ... }: {
    module.args.pkgs = import inputs.nixpkgs {
      inherit system;
      config = {
        allowUnfree = true;
        allowUnfreePredicate = _: true;
      };
      overlays = [
        (final: prev: {
          myCustomTool = prev.callPackage ../pkgs/myCustomTool { };
        })
      ];
    };
  };
}

```

---

## 2. Multi-Host Orchestration

Define helper functions to instantiate hosts without duplicating code. Each host consumes `self'.pkgs` and accepts `self'` and `inputs'` via `specialArgs`.

### `parts/hosts.nix`

```nix
{ inputs, self, ... }:
let
  # Helper to construct a host cleanly
  mkHost = { hostname, system ? "x86_64-linux", extraModules ? [ ] }:
    let
      self' = self.perSystem.${system};
      inputs' = inputs.flake-parts.lib.inputsWithSystem system inputs;
    in
    inputs.nixpkgs.lib.nixosSystem {
      inherit system;
      specialArgs = { inherit self' inputs'; };
      modules = [
        # Force NixOS to use perSystem's configured pkgs
        { nixpkgs.pkgs = self'.pkgs; }
        { networking.hostName = hostname; }
        
        # Import core modules
        ../modules/nixos/core.nix
        ../hosts/${hostname}
      ] ++ extraModules;
    };
in
{
  flake.nixosConfigurations = {
    # High-performance desktop
    desktop = mkHost {
      hostname = "desktop";
      system = "x86_64-linux";
      extraModules = [ ../modules/nixos/hyprland.nix ];
    };

    # Portable laptop
    laptop = mkHost {
      hostname = "laptop";
      system = "x86_64-linux";
      extraModules = [ ../modules/nixos/hyprland.nix ];
    };

    # ARM-based Home Server
    home-server = mkHost {
      hostname = "home-server";
      system = "aarch64-linux"; # ARM arch handled seamlessly!
      extraModules = [ ../modules/nixos/server-base.nix ];
    };
  };
}

```

---

## 3. Exporting `flake.nixosModules` for Public Consumption

When exporting NixOS modules for downstream flakes to import via `inputs.your-flake.nixosModules.default`, the consumer's architecture (`x86_64-linux`, `aarch64-linux`) is not known upfront.

If a public module references packages directly from flake inputs (e.g. `inputs.ghostty.packages.x86_64-linux.default`), it will fail on ARM or non-standard targets.

Using `modulesWithSystem` inside `flake.nixosModules` solves this by automatically resolving `self'` and `inputs'` to whatever system architecture the consumer is evaluating.

### `parts/modules.nix`

```nix
{ inputs, options, ... }: {
  flake.nixosModules = {
    # 1. Export individual features
    hyprland-desktop = { config, lib, pkgs, ... }: {
      imports = [
        (inputs.flake-parts.lib.modulesWithSystem { inherit inputs options; }
          ({ self', inputs', ... }: {
            # Enable hardware/system configs
            programs.hyprland.enable = true;

            environment.systemPackages = [
              # Pulls input packages bound automatically to target system
              inputs'.hyprland.packages.hyprland
              inputs'.ghostty.packages.default
              
              # Pulls custom packages built in this flake's perSystem
              self'.packages.myCustomTool
            ];
          })
        )
      ];
    };

    # 2. Export a default bundled module entrypoint
    default = { ... }: {
      imports = [
        inputs.self.nixosModules.hyprland-desktop
      ];
    };
  };
}

```

---

## Downstream Usage

Because the module is wrapped in `modulesWithSystem`, an external user can import it without configuring special arguments or specifying architecture strings:

### External `flake.nix`

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    ty-config.url = "github:your-username/nix-config";
  };

  outputs = { nixpkgs, ty-config, ... }: {
    nixosConfigurations.my-machine = nixpkgs.lib.nixosSystem {
      system = "aarch64-linux"; # Works on ARM automatically!
      modules = [
        ty-config.nixosModules.hyprland-desktop
        ./hardware-configuration.nix
      ];
    };
  };
}

```

---

## Core Benefits

* **Architecture Agnostic:** Whether building locally for `x86_64-linux` or cross-compiling for an `aarch64-linux` server, `modulesWithSystem` binds `self'` and `inputs'` to the target machine without string hacks.
* **True Public Reusability:** Downstream consumers get your custom packages and binary inputs without needing extra parameter passing or manual `specialArgs` wiring.
* **Single Evaluation Tree:** Multi-host environments share the exact same `perSystem` package instantiation logic, keeping evaluation fast and memory overhead low.
  
  [[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]