You hit the exact nail on the head. Pre-baked wrapper libraries (like nix-wrapper-modules) look clean in YouTube videos when the author is showing off Kitty, Zsh, or Neovim—programs that already have pre-made wrappers built for them. But the moment you want to configure modern tools like Ghostty, Hyprland 0.55+, or custom Wayland utilities, the abstractions fall apart and leave you writing verbose boilerplate.
If you don't want to rely on pkgs.symlinkJoin or bloated external wrapper libraries, you can build a clean, native flake module that handles homeless wrapped packages alongside your system configuration.
The Pure Flake-Parts Architecture (No Fluff, No HM)
Instead of relying on fragile third-party modules or outdated symlinkJoin scripts, you can build a lightweight helper function right inside your perSystem block using standard pkgs.writeShellApplication or pkgs.makeWrapper.
Here is how you write a native Ghostty package inside a dendritic flake-parts setup.
1. The Portable Ghostty Wrapper Module
You can construct a clean Ghostty package directly in perSystem using standard pkgs.makeWrapper. This bakes your config straight into the store without needing external wrapper modules or Home Manager.
# modules/features/ghostty/default.nix
{ inputs, ... }:

{
  # System-level module (for your NixOS machine)
  flake.nixosModules.ghostty = { config, lib, pkgs, ... }: {
    options.features.ghostty.enable = lib.mkEnableOption "Ghostty Terminal";

    config = lib.mkIf config.features.ghostty.enable {
      environment.systemPackages = [
        # Pulls the wrapped package directly from the perSystem flake outputs
        inputs.self.packages.${pkgs.stdenv.hostPlatform.system}.ghostty
      ];
    };
  };

  # Package export (allows `nix run github:you/dotfiles#ghostty` anywhere)
  perSystem = { pkgs, ... }: let
    # Write the raw Ghostty config file directly to the Nix Store
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
      version = pkgs.ghostty.version;

      # Don't recompile; just wrap the binary
      dontUnpack = true;
      nativeBuildInputs = [ pkgs.makeWrapper ];

      installPhase = ''
        mkdir -p $out/bin
        makeWrapper ${pkgs.ghostty}/bin/ghostty $out/bin/ghostty \
          --add-flags "--config-file=${ghosttyConfig}" \
          --prefix PATH : ${lib.makeBinPath [ pkgs.fish pkgs.git ]}
      '';
    };
  };
}

Why This Clean Pattern Works
 * Zero External Dependencies: You don't need nix-wrapper-modules or Home Manager. You are using pkgs.makeWrapper, which is part of Nixpkgs standard library and will never go out of date.
 * Total Control Over Flags: Since Ghostty expects --config-file=, you explicitly pass --config-file=${ghosttyConfig}. You aren't guessing what an external wrapper library calls its options.
 * Runtime Path Environment (--prefix PATH): Notice the --prefix PATH line. This fixes the issue where Ghostty launches but can't find fish, git, or your terminal tools. It bakes those dependencies directly into Ghostty's execution scope.
 * Dual Capability:
   * On your NixOS machine: Turn on features.ghostty.enable = true;.
   * On any random Linux box: Run nix run .#ghostty or nix run github:yourusername/nixos-config#ghostty.
Scaling This Up: Creating a Generic Package Wrapper Helper
If you have 10 apps that don't have built-in wrapper modules, you don't want to repeat mkDerivation over and over. You can write a 5-line helper function at the top of your flake to wrap any package with a config file and flag.
# A reusable helper function for your perSystem scope
let
  wrapApp = { pkg, binName ? pkg.pname, flags ? [], runtimeDeps ? [] }:
    pkgs.stdenv.mkDerivation {
      pname = "${binName}-wrapped";
      version = pkg.version;
      dontUnpack = true;
      nativeBuildInputs = [ pkgs.makeWrapper ];
      installPhase = ''
        mkdir -p $out/bin
        makeWrapper ${pkg}/bin/${binName} $out/bin/${binName} \
          ${pkgs.lib.optionalString (flags != []) "--add-flags \"${pkgs.lib.concatStringsSep " " flags}\""} \
          ${pkgs.lib.optionalString (runtimeDeps != []) "--prefix PATH : \"${pkgs.lib.makeBinPath runtimeDeps}\""}
      '';
    };
in {
  packages = {
    # Ghostty
    ghostty = wrapApp {
      pkg = pkgs.ghostty;
      flags = [ "--config-file=${ghosttyConfig}" ];
      runtimeDeps = [ pkgs.fish ];
    };

    # Hyprland
    hyprland = wrapApp {
      pkg = pkgs.hyprland;
      flags = [ "--config ${hyprlandConf}" ];
      runtimeDeps = [ pkgs.kitty pkgs.wofi pkgs.mako ];
    };
  };
}

This gives you a robust setup without relying on third-party abstractions, keeping your configuration fully portable and maintainable.

Building real custom NixOS modules with custom options—the exact same way `nixpkgs` or Home Manager does under the hood—is the cleanest and most scalable architectural approach.

By writing standard module options using `lib.mkOption`, `lib.mkEnableOption`, and `lib.types`, you get full type checking, clean modularity, and custom configuration interfaces without relying on third-party wrapper hacks.

## The Clean Custom Module Pattern

Here is how you write a native, first-class custom module with custom options that exposes both a **NixOS system configuration** and a **standalone wrapped package export** for `nix run`.

### 1. Defining the Custom Module with `lib.mkOption`

# modules/features/ghostty.nix
{ config, lib, pkgs, inputs, ... }:

let
  cfg = config.features.ghostty;

  # Build the raw configuration file from user options
  ghosttyConfigFile = pkgs.writeText "ghostty.conf" ''
    theme = ${cfg.settings.theme}
    font-size = ${toString cfg.settings.fontSize}
    window-padding-x = ${toString cfg.settings.padding}
    window-padding-y = ${toString cfg.settings.padding}
    ${cfg.extraConfig}
  '';

  # Construct the wrapped package using native Nix options
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
  # Define custom options (like nixpkgs/modules)
  options.features.ghostty = {
    enable = lib.mkEnableOption "Ghostty terminal module";

    package = lib.mkOption {
      type = lib.types.package;
      default = pkgs.ghostty;
      description = "The underlying Ghostty package to wrap.";
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
        description = "Color scheme theme for Ghostty.";
      };

      fontSize = lib.mkOption {
        type = lib.types.int;
        default = 12;
        description = "Font size in points.";
      };

      padding = lib.mkOption {
        type = lib.types.int;
        default = 10;
        description = "Window padding in pixels.";
      };
    };

    extraConfig = lib.mkOption {
      type = lib.types.lines;
      default = "";
      description = "Raw configuration lines appended to ghostty.conf.";
    };
  };

  # Module Implementation
  config = lib.mkIf cfg.enable {
    environment.systemPackages = [ wrappedGhostty ];
  };
}

## 2. Exporting It via `flake-parts` (For Standalone `nix run`)

To expose this wrapped package through `perSystem.packages` so you can use `nix run .#ghostty` on any system, evaluate the module standalone using `lib.evalModules`:

# flake.nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    flake-parts.url = "github:hercules-ci/flake-parts";
  };

  outputs = inputs@{ flake-parts, nixpkgs, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      systems = [ "x86_64-linux" "aarch64-linux" ];

      flake.nixosModules.ghostty = import ./modules/features/ghostty.nix;

      perSystem = { pkgs, system, ... }: {
        packages.ghostty = let
          # Evaluate our custom module independently of NixOS
          eval = nixpkgs.lib.evalModules {
            modules = [
              ./modules/features/ghostty.nix
              {
                # Evaluate with module enabled and default/custom options
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

## Key Advantages of This Architecture

- **Type Safety & Validation:** Using `lib.types.int`, `lib.types.str`, or `lib.types.listOf` ensures Nix catches invalid configurations (like passing a string to `fontSize`) at evaluation time with actionable error messages.
- **No Reliance on Third-Party Wrapper Modules:** You aren't constrained by whether an external wrapper library supports a specific application. If a program uses CLI flags, environment variables, or config paths, you control the wrapper logic directly.
- **Universal Usability:**
    - On a NixOS machine, import `nixosModules.ghostty` and configure `features.ghostty = { enable = true; theme = "tokyo-night"; };`.
    - On an external machine, `nix run github:your/repo#ghostty` executes the evaluated module and binary directly from the `/nix/store`.