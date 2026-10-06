# Combining Dendritic Layout (`import-tree`) with `flake-parts`
>Combining your **dendritic layout** `(import-tree)` with `flake-parts` eliminates manual `lib.evalModules` calls in `flake.nix`.
By letting a dedicated module file return a `flake-parts` module (with `perSystem` and `flake.nixosModules`), `import-tree` automatically discovers and evaluates your Ghostty configuration just like your UWSM, Niri, and Noctalia modules.
___
## File Structure
```text
.
├── flake.nix
└── modules/
    ├── parts.nix             # Flake-wide settings (systems, nixpkgs config)
    └── programs/
        ├── ghostty.nix       # Option schema & derivation engine
        └── my-ghostty.nix    # flake-parts module (Your settings + output binding)

```
___
### Step 1: `modules/programs/ghostty.nix` (The Engine / NixOS Option Schema)
>Keep this pure as a standard NixOS/Home Manager module schema.
```nix
{
 ...
}:
{
  flake.nixosModules.ghosttyModule = { pkgs, config, lib, ... }:

  with lib; # <== what is this? never used it, i try to avoid with statements to, probably best to remove it if going with this version and wrapping the module and exposing in flake.nixosModules so its auto imported by import-tree rexursively at top level so i can make a seperate module just ".ghostty" where i configure it in the same file with perSystem in it, two seperate blocks but in the same.nix.

  let
    cfg = config.mySystem.programs.ghostty;

    formatKeyValue = k: v:
      if isList v then
        concatStringsSep "\n" (map (item: "${k} = ${toString item}") v)
      else if isBool v then
        "${k} = ${if v then "true" else "false"}"
      else
        "${k} = ${toString v}";

    ghosttyConfigContent = concatStringsSep "\n" (
      mapAttrsToList formatKeyValue cfg.settings
    );

    ghosttyConfigFile = pkgs.writeText "ghostty-config" ghosttyConfigContent;

    wrappedGhostty = pkgs.symlinkJoin {
      name = "ghostty-configured-${pkgs.ghostty.version}";
      paths = [ pkgs.ghostty ];
      nativeBuildInputs = [ pkgs.makeWrapper ];
      postBuild = ''
        wrapProgram $out/bin/ghostty \
          --add-flags "--config-file=${ghosttyConfigFile}"
      '';
    };
  in
  {
    options.mySystem.programs.ghostty = {
      enable = mkEnableOption "Custom wrapped Ghostty terminal";

      enableFishIntegration = mkOption {
        type = types.bool;
        default = false;
        description = "Enable shell integration hooks for Fish.";
      };

      settings = mkOption {
        type = types.attrsOf (types.oneOf [
          types.str
          types.int
          types.float
          types.bool
          (types.listOf types.str)
        ]);
        default = {};
        description = "Raw Ghostty key-value configuration options.";
      };

      package = mkOption {
        type = types.package;
        default = wrappedGhostty;
        description = "The final wrapped Ghostty package output.";
      };
    };

    config = mkIf cfg.enable {
      environment.systemPackages = [ cfg.package ];

      programs.fish.vendor.configDirs = mkIf (cfg.enableFishIntegration && config ? programs.fish) [
        "${pkgs.ghostty}/share/fish/vendor_conf.d"
      ];
    };
  };
}

```
___
### Step 2: `modules/programs/my-ghostty.nix` (The Flake-Parts Integration)
>This turns Ghostty into a native dendritic module. It uses `lib.evalModules` internally inside `perSystem`, so `flake.nix` never has to think about it.
___
```nix
{
  self,
  inputs,
  ...
}:
{
  perSystem = { pkgs, ... }:
  let
    # Evaluates the engine and settings locally within perSystem
    eval = pkgs.lib.evalModules {
      modules = [
        ./ghostty.nix
        {
          mySystem.programs.ghostty = {
            enable = true;
            enableFishIntegration = true;
            settings = {
              theme = "Aurora";
              background-opacity = 1.0;
              adjust-cell-height = "-10%";
              adjust-cell-width = "-10%";
              cursor-style = "block";
              grapheme-width-method = "legacy";
              shell-integration-features = "no-cursor";
              scrollback-limit = 100000000;
              font-family = "JetBrainsMono NFM Bold";
              font-family-bold = "JetBrainsMono NFM ExtraBold";
              font-family-italic = "JetBrainsMono NFM Bold Italic";
              font-family-bold-italic = "JetBrainsMono NFM ExtraBold Italic";
              font-size = 22;
              font-feature = [
                "liga"
                "calt"
              ];
            };
          };
        }
      ];
      specialArgs = { inherit pkgs; };
    };
  in
  {
    # Exposes the evaluated package directly under `packages.myGhostty`
    packages.myGhostty = eval.config.mySystem.programs.ghostty.package;
  };

  # Exposes the system module export clean for downstream NixOS consumption
  flake.nixosModules.ghostty = { pkgs, ... }:
  {
    programs.ghostty = {
      enable = true;
      package = config.mySystem.programs.ghostty
      # Or
      package = pkgs.myGhostty
      # ?
      # or possibly if i change it above to fall under programs to reuse existing module logic for other modules and make config equal to config.programs.ghostty and or i could use wrapPackage to make myGhostty and set package = to myGhostty?
    };  
    ## Now Able to Remove This After Wrapping Part 1 in flake.nixosModules.ghosttyModule? # this method is definetly better.
    imports = [ ./ghostty.nix ]; # <== this defeats the purpose of import-tree.
    ##
    mySystem.programs.ghostty.enable = true; # Does this function the same as using wrapPackage and myGhostty and still able to expose stuff and lock in the settings and custom nixosmodule options in perSystem? to build built on multiple systems and via nix run @mygit/.#myGhostty(if wrapped and named myGhostty with wrapPackage)? or configuring something under flake.nixosModules.ghostty with programs.ghostty? or 
  };
}

```
___
### Step 3: `flake.nix` (Zero Logic / Fully Dendritic)
> [!NOTE] 
> Your `flake.nix` stays completely untouched regardless of how many packages or system modules you add.
> 
___
```nix
{
  description = "Nix Dendritic Flake";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-parts.url = "github:hercules-ci/flake-parts";
    import-tree.url = "github:vic/import-tree";
    wrappers.url = "github:BirdeeHub/nix-wrapper-modules";

    chaotic = {
      url = "github:chaotic-cx/nyx";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = inputs@{ flake-parts, import-tree, ... }:
    flake-parts.lib.mkFlake { inherit inputs; } {
      imports = import-tree ./modules;
    };
}
```

# Main Dendritic Guides
[[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]
[[Import&Filter Hosts NixOS&Dendritic]]
[[Migrating To Dendritic Nix Safely With My Main Configuration Staying Intact]]
[[Dendritic Nix Style Warpd]]
[[Dendritic Nix Style Wlr-Which-Key]]
[[Avoid Import-Tree]]
[[Dendritic Nix Working Flake-Parts Module]]
[[Putting Programs Config and Custom PKGS in perSystem (Runnable As Standalone Binary With Github Link + Path, Pkg Name]]
# Flake-Parts Advice
[[Flake-parts options parts.nix]]
[[Flake parts self' and inputs']]
[[Flake parts fixes -system-]]
[[Making Dendrite Modules Runnable As Standalone Packages Via Github Link + Path + PKG Name]]
# Major Weapons Ill Need To Complete This Task.
[[NixOS Module Mastery]]
[[REAL NixOS Modules From Scratch]]
[[Variants Of pkgs.write And pkgs.run]]
[[formatLine, mapAttrsToList And The Beauty Of builtins.toJSON, pkgs.formats.toml, And pkgs.formats.ini]]
[[override, overrideAttrs, and oldAttrs]]
# Extras and Advice On Dendritic Nix
[[Multi-Host Dendritic and publicly consumable nixosModules]]
[[File-system Organization and ASCII Priority]]
[[& vs && and ; operator]]
[[Sed-cli]]
[[vimiumC]]
[[Markdown syntax]]
[[JJ Workflow]]
# Get an Infinite Recursion Error?
[[Infinite Recursion Bug]]