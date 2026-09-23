Here is your note cleaned up and formatted into pure, production-grade Obsidian Markdown.

I stripped out all conversational boilerplate, fixed syntax errors in code blocks (like bad string escaping in the shell commands, missing semicolon terminations, and invalid option paths), standardized the layout, and structured the concepts with callouts, wikilinks, and clean code sections.

---

# Native Package Wrapping & Custom Module Authoring in `flake-parts`

#nix #nixos #flakes #flake-parts #architecture #devops #ghostty

Pre-baked wrapper abstractions (like `nix-wrapper-modules` or Home Manager package overrides) work well for basic applications, but they crumble when configuring bleeding-edge tools (e.g., Ghostty, Hyprland 0.55+, custom Wayland utilities). When third-party abstractions fail, you are often left writing verbose, fragile boilerplate with `pkgs.symlinkJoin`.

Building native package wrappers directly inside a dendritic `flake-parts` architecture using standard `pkgs.makeWrapper` or `pkgs.runCommand` completely eliminates third-party dependencies while maintaining type safety, instant portability (`nix run`), and total control over flags and runtime environments.

---

## Paradigm 1: Lightweight Function-Based Wrapping (`perSystem`)

When you don't need a full NixOS options schema and simply want to export a configured tool for both your NixOS system and direct execution (`nix run .#app`).

### Portable Ghostty Wrapper Module

This pattern uses `pkgs.makeWrapper` to write the configuration directly to the Nix Store and bake runtime PATH dependencies into the executable itself.

```nix
# modules/features/ghostty/default.nix
{ inputs, ... }:

{
  # 1. System-level module (for NixOS machine target)
  flake.nixosModules.ghostty = { config, lib, pkgs, ... }: {
    options.features.ghostty.enable = lib.mkEnableOption "Ghostty Terminal";

    config = lib.mkIf config.features.ghostty.enable {
      environment.systemPackages = [
        # Pulls the wrapped package directly from perSystem flake outputs
        inputs.self.packages.${pkgs.stdenv.hostPlatform.system}.ghostty
      ];
    };
  };

  # 2. Package export (allows `nix run .#ghostty` anywhere)
  perSystem = { pkgs, lib, ... }: let
    ghosttyConfig = pkgs.writeText "ghostty-config" ''
      theme = catppuccin-mocha
      font-size = 12
      window-padding-x = 12
      window-padding-y = 12
      command = ${pkgs.fish}/bin/fish
    '';
  in {
    packages.ghostty = pkgs.stdenv.mkDerivation {
      pname = "ghostty-wrapped";
      version = pkgs.ghostty.version or "0.1.0";

      dontUnpack = true;
      nativeBuildInputs = [ pkgs.makeWrapper ];

      installPhase = ''
        mkdir -p $out/bin
        makeWrapper ${pkgs.ghostty}/bin/ghostty$out/bin/ghostty \
          --add-flags "--config-file=${ghosttyConfig}" \
          --prefix PATH : ${lib.makeBinPath [ pkgs.fish pkgs.git ]}
      '';
    };
  };
}

```

> [!success] Core Benefits
> * **Zero External Dependencies:** Relies strictly on `pkgs.makeWrapper` from upstream [[Nixpkgs]].
> * **Hardened Runtime Scope (`--prefix PATH`):** Fixes the common issue where launched GUIs cannot find terminal shells, Git, or compositor utilities by baking runtime dependencies directly into the package execution path.
> * **Dual Capabilities:** Can be toggled on NixOS via `features.ghostty.enable = true;` or executed on any non-NixOS Linux machine via `nix run github:user/dotfiles#ghostty`.
> 
> 

---

## Scaling Up: Generic Package Wrapper Helper

Rather than repeating `mkDerivation` boilerplate across ten different standalone tools, you can abstract wrapper creation into a high-order function inside your `perSystem` block.

```nix
# Example perSystem helper abstraction
perSystem = { pkgs, lib, ... }: let
  wrapApp = { 
    pkg, 
    binName ? pkg.pname, 
    flags ? [ ], 
    runtimeDeps ? [ ] 
  }:
    pkgs.stdenv.mkDerivation {
      pname = "${binName}-wrapped";
      version = pkg.version or "0.0.0";
      dontUnpack = true;
      nativeBuildInputs = [ pkgs.makeWrapper ];
      installPhase = ''
        mkdir -p $out/bin
        makeWrapper ${pkg}/bin/${binName} $out/bin/${binName} \
          ${lib.optionalString (flags != [ ]) "--add-flags \"${lib.concatStringsSep " " flags}\""} \
          ${lib.optionalString (runtimeDeps != [ ]) "--prefix PATH : \"${lib.makeBinPath runtimeDeps}\""}
      '';
    };

  ghosttyConf = pkgs.writeText "ghostty.conf" "theme = catppuccin-mocha";
  hyprlandConf = pkgs.writeText "hyprland.conf" "# Hyprland Core Config";
in {
  packages = {
    # Ghostty Wrapper
    ghostty = wrapApp {
      pkg = pkgs.ghostty;
      flags = [ "--config-file=${ghosttyConf}" ];
      runtimeDeps = [ pkgs.fish pkgs.git ];
    };

    # Hyprland Compositor Wrapper
    hyprland = wrapApp {
      pkg = pkgs.hyprland;
      flags = [ "--config ${hyprlandConf}" ];
      runtimeDeps = [ pkgs.kitty pkgs.wofi pkgs.mako ];
    };
  };
};

```

---

## Paradigm 2: First-Class Custom Module with `lib.mkOption`

For full configuration control, type-checking, and evaluation parity with native `nixpkgs` and Home Manager modules, define custom options directly with `lib.mkOption`.

### 1. The Custom Feature Module

```nix
# modules/features/ghostty.nix
{ config, lib, pkgs, ... }:

let
  cfg = config.features.ghostty;

  # Build raw configuration file from typed module options
  ghosttyConfigFile = pkgs.writeText "ghostty.conf" ''
    theme = ${cfg.settings.theme}
    font-size = ${toString cfg.settings.fontSize}
    window-padding-x = ${toString cfg.settings.padding}
    window-padding-y = ${toString cfg.settings.padding}
    ${cfg.extraConfig}
  '';

  # Construct the wrapped package using native Nix build tools
  wrappedGhostty = pkgs.runCommand "ghostty-wrapped" {
    nativeBuildInputs = [ pkgs.makeWrapper ];
  } ''
    mkdir -p $out/bin
    makeWrapper ${cfg.package}/bin/ghostty $out/bin/ghostty \
      --add-flags "--config-file=${ghosttyConfigFile}" \
      --prefix PATH : ${lib.makeBinPath cfg.runtimePackages}
  '';
in
{
  # Option Schema Definition
  options.features.ghostty = {
    enable = lib.mkEnableOption "Ghostty terminal module";

    package = lib.mkOption {
      type = lib.types.package;
      default = pkgs.ghostty;
      description = "The underlying base Ghostty package to wrap.";
    };

    runtimePackages = lib.mkOption {
      type = lib.types.listOf lib.types.package;
      default = [ pkgs.fish ];
      description = "Extra binaries to expose in Ghostty's PATH at runtime.";
    };

    settings = {
      theme = lib.mkOption {
        type = lib.types.str;
        default = "catppuccin-mocha";
        description = "Color scheme theme name.";
      };

      fontSize = lib.mkOption {
        type = lib.types.int;
        default = 12;
        description = "Font size in points.";
      };

      padding = lib.mkOption {
        type = lib.types.int;
        default = 10;
        description = "Window internal padding in pixels.";
      };
    };

    extraConfig = lib.mkOption {
      type = lib.types.lines;
      default = "";
      description = "Raw configuration text appended to ghostty.conf.";
    };
  };

  # System Execution Implementation
  config = lib.mkIf cfg.enable {
    environment.systemPackages = [ wrappedGhostty ];
  };
}

```

---

### 2. Standalone Flake Export via `lib.evalModules`

To make this custom module's package output executable via `nix run .#ghostty` *without* building a full NixOS system, evaluate the module standalone inside your `flake-parts` pipeline using `lib.evalModules`.

```nix
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    flake-parts.url = "github:hercules-ci/flake-parts";
  };

  outputs = inputs@{ flake-parts, nixpkgs, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      systems = [ "x86_64-linux" "aarch64-linux" ];

      # System-level export for NixOS host builds
      flake.nixosModules.ghostty = import ./modules/features/ghostty.nix;

      perSystem = { pkgs, system, ... }: {
        packages.ghostty = let
          # Evaluate our custom module independently of a NixOS system tree
          eval = nixpkgs.lib.evalModules {
            modules = [
              ./modules/features/ghostty.nix
              {
                features.ghostty.enable = true;
                _module.args.pkgs = pkgs;
              }
            ];
          };
        in
          # Extract the resulting wrapped package built by environment.systemPackages
          builtins.head eval.config.environment.systemPackages;
      };
    };
}

```

---

## Architectural Summary

| Pattern | Primary Advantage | Typical Scope |
| --- | --- | --- |
| **`perSystem` Wrapper Helper** | Zero overhead, fast, single file declaration | Custom scripts, simple CLI tools, fast prototyping |
| **Custom Option Module (`lib.evalModules`)** | Full type-checking, LSP options autocomplete, reusable across systems | Core desktop tools, Wayland compositors, complex apps |

> [!note] Evaluation Pipeline
> 1. **NixOS Host Target:** `nixosModules.ghostty` $\rightarrow$ Evaluated during `nixos-rebuild switch`.
> 2. **Standalone Runner Target:** `perSystem.packages.ghostty` $\rightarrow$ Isolated evaluation via `lib.evalModules` for instant execution via `nix run`.
>    

[[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]