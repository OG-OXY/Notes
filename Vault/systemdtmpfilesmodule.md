---
id: systemdtmpfilesmodule
aliases: []
tags: []
---
Building a Home Manager-free system module pattern for your core tools is straightforward once you generalize the `systemd.tmpfiles` approach. By writing clean, modular options that output Nix text files and drop them straight into `~/.config`, you keep your entire user environment fully declarative from `nixos-rebuild`.

Here is the blueprint for how you can structure a reusable module pattern for any tool (like your terminal, shell initialization, or window manager config) without a single line of Home Manager:

### 1. The Reusable Module Pattern

Create a module file (e.g., `modules/xdg-app.nix`) that accepts an attribute set of settings or raw text, writes it to the Nix store, and drops a symlink via `systemd.tmpfiles`:

```nix
{ config, lib, pkgs, ... }:

let
  cfg = config.programs.myTool; # Replace with your tool name
in {
  options.programs.myTool = {
    enable = lib.mkEnableOption "myTool configuration";
    
    # Optional: support structured settings or raw text blocks
    configFile = lib.mkOption {
      type = lib.types.path;
      description = "Path to the generated config file.";
    };
  };

  config = lib.mkIf cfg.enable {
    # System-wide tmpfiles rule forcing the symlink into ~/.config/mytool/config
    systemd.tmpfiles.rules = [
      "d /home/ty/.config/mytool 0755 ty users -"
      "L+ /home/ty/.config/mytool/config - - - - ${cfg.configFile}"
    ];
  };
}

```

### 2. Scaling to Multi-User / Multi-Path Configs

If you want to manage multiple config files or ensure user directories scale cleanly without hardcoding `/home/ty`, you can map over `config.users.users` or build a generic helper loop inside your dendritic flake architecture.

**Dynamic User Mapping via `config.users.users**`

To avoid hardcoding `/home/ty`, iterate over `config.users.users` using `lib.mapAttrsToList`. This extracts the actual home directory path (`user.home`) and username for any user configured on the system, filtering out system users by checking if their shell or home directory matches expected patterns (or explicitly targeting human users).

**Multi-File Attribute Set Pattern**

To handle multiple files across different tool configs, define an option that takes an attribute set where the key is the relative path inside `~/.config/` and the value is the text block or Nix store path.

**Combined Module Implementation**

The following reusable NixOS module pattern implements both approaches. It loops through target users and generates `systemd.tmpfiles.rules` for multiple configuration files automatically:

```nix
{ config, lib, pkgs, ... }:

let
  cfg = config.programs.systemDotfiles;

  # Filter for actual human users (e.g. users with a home directory under /home)
  normalUsers = lib.filterAttrs 
    (name: user: user.home != null && lib.hasPrefix "/home/" user.home) 
    config.users.users;

  # Helper to generate tmpfiles lines for a given user and config map
  mkUserRules = username: user: files:
    let
      home = user.home;
    in
      lib.flatten (lib.mapAttrsToList (subPath: fileContent: [
        # Ensure parent directory exists
        "d ${home}/.config/${lib.dirOf subPath} 0755 ${username} ${user.group} -"
        # Create symlink or file
        "L+ ${home}/.config/${subPath} - - - - ${fileContent}"
      ]) files);

  # Flatten rules across all target users and all files
  allTmpfilesRules = lib.flatten (
    lib.mapAttrsToList (username: user: mkUserRules username user cfg.files) normalUsers
  );
in
{
  options.programs.systemDotfiles = {
    enable = lib.mkEnableOption "system-level config deployment via tmpfiles";

    files = lib.mkOption {
      type = lib.types.attrsOf lib.types.path;
      default = {};
      description = "Attribute set of ~/.config/ relative paths mapped to store paths/files.";
      example = lib.literalExpression ''
        {
          "warpd/config" = pkgs.writeText "warpd-config" "left: space";
          "ghostty/config" = pkgs.writeText "ghostty-config" "theme = catppuccin-mocha";
        }
      '';
    };
  };

  config = lib.mkIf cfg.enable {
    systemd.tmpfiles.rules = allTmpfilesRules;
  };
}

```

**How to Consume It in Your Flake**

With the generic module in place, you can feed multiple tool configurations into it from anywhere in your dendritic flake without binding to Home Manager:

```nix
programs.systemDotfiles = {
  enable = true;
  files = {
    "warpd/config" = pkgs.writeText "warpd-config" ''
      left: space
      right: r
      hint_activation_key: A-M-h
    '';
    "ghostty/config" = pkgs.writeText "ghostty-config" ''
      font-size = 14
      cursor-style = block
    '';
  };
};

```
