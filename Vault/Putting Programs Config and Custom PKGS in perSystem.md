You can achieve **exactly** that. You can expose a standalone `apps` or `packages` entry inside `perSystem` so you can fire off `nix run github:yourusername/yourrepo#wlr-which-key` from any bare-bones live ISO without needing NixOS installed.

In `perSystem`, you build the menu configs and package scripts directly using that system's `pkgs`, then wrap `wlr-which-key` into a runnable app/package.

Here is how you structure your module file so it gives you **both**:

1. A NixOS module (`flake.nixosModules.wlr-which-key`) for system configs.
2. A standalone runnable app (`perSystem.apps.wlr-which-key` / `perSystem.packages.wlr-which-key`) to execute anywhere.

```nix
{ self, inputs, lib, ... }:

{
  # 1. NixOS Module for your full system config
  flake.nixosModules.wlr-which-key = { config, pkgs, ... }: {
    # (Your existing option declarations and system module logic)
  };

  # 2. Standalone app usable on ANY machine via `nix run .#wlr-which-key` or `nix run github:user/repo#wlr-which-key`
  perSystem = { pkgs, system, ... }:
    let
      # Define settings & menus right here for the standalone package
      settings = {
        anchor = "center";
        font = "JetBrainsMono NFM 12";
        margin = 10;
        padding = 15;
        border_width = 2;
      };

      menus = {
        warp = [
          { key = "w"; desc = "Normal Mode"; cmd = "${pkgs.warpd}/bin/warpd --normal"; }
          { key = "h"; desc = "Hint Mode"; cmd = "${pkgs.warpd}/bin/warpd --hint"; }
          { key = "q"; desc = "Quadrant Mode"; cmd = "${pkgs.warpd}/bin/warpd --grid"; }
        ];
        apps = [
          { key = "l"; desc = "Launcher"; cmd = "${pkgs.noctalia-shell}/bin/noctalia-shell ipc call launcher toggle"; }
          { key = "h"; desc = "Herdr"; cmd = "${pkgs.ghostty}/bin/ghostty -e${pkgs.herdr}/bin/herdr"; }
          { key = "g"; desc = "Ghostty"; cmd = "${pkgs.ghostty}/bin/ghostty"; }
          { key = "y"; desc = "Yazi"; cmd = "${pkgs.ghostty}/bin/ghostty -e${pkgs.fish}/bin/fish -i -C 'y'"; }
          { key = "z"; desc = "Zen-Browser"; cmd = "${inputs.zen-browser.packages.${system}.default}/bin/zen-beta"; }
          { key = "o"; desc = "Obsidian"; cmd = "${pkgs.obsidian}/bin/obsidian /home/ty/Notes/Vault"; }
          { key = "n"; desc = "Obsidian (New Note)"; cmd = "xdg-open 'obsidian://new?vault=Vault&name=New%20Note'"; }
          { key = "v"; desc = "Vesktop"; cmd = "${pkgs.vesktop}/bin/vesktop"; }
          { key = "s"; desc = "OBS-Studio"; cmd = "${pkgs.obs-studio}/bin/obs"; }
          { key = "b"; desc = "Bitwarden"; cmd = "${pkgs.bitwarden-desktop}/bin/bitwarden"; }
          { key = "e"; desc = "EasyEffects"; cmd = "${pkgs.easyeffects}/bin/easyeffects"; }
        ];
      };

      # Helper to build YAML configs for each menu
      mkMenuPackage = name: menuEntries:
        let
          configSet = {
            inherit (settings) anchor padding border_width font;
            margin_top = settings.margin;
            margin_bottom = settings.margin;
            margin_left = settings.margin;
            margin_right = settings.margin;
            menu = menuEntries;
          };
          configFile = pkgs.writeText "${name}-config.yaml" (builtins.toJSON configSet);
        in
        pkgs.writeShellScriptBin "wlr-which-key-${name}" ''
          exec ${pkgs.wlr-which-key}/bin/wlr-which-key${configFile}
        '';

      # Generates binaries for each menu item (e.g., wlr-which-key-warp, wlr-which-key-apps)
      menuBins = lib.mapAttrsToList mkMenuPackage menus;

      # Main runner script that launches the apps menu by default
      mainRunner = pkgs.writeShellScriptBin "wlr-which-key" ''
        exec ${pkgs.wlr-which-key}/bin/wlr-which-key${
          pkgs.writeText "default-config.yaml" (builtins.toJSON {
            inherit (settings) anchor padding border_width font;
            margin_top = settings.margin;
            margin_bottom = settings.margin;
            margin_left = settings.margin;
            margin_right = settings.margin;
            menu = menus.apps;
          })
        }
      '';

      # Combine wlr-which-key, warpd, and all menu scripts into one closure
      combinedPackage = pkgs.symlinkJoin {
        name = "wlr-which-key-standalone";
        paths = [ mainRunner pkgs.wlr-which-key pkgs.warpd ] ++ menuBins;
      };
    in
    {
      # Expose as a package
      packages.wlr-which-key = combinedPackage;

      # Expose as a runnable app for `nix run`
      apps.wlr-which-key = {
        type = "app";
        program = "${combinedPackage}/bin/wlr-which-key";
      };
    };
}

```

---

### How to use it anywhere

On a fresh minimalist NixOS Live ISO (or any Wayland desktop with Nix installed):

```bash
# Run the default apps menu directly from your repo
nix run github:yourusername/yourrepo#wlr-which-key

# Or run specific menus directly if added as apps
nix run github:yourusername/yourrepo#wlr-which-key -- --menu apps

```

Nix will pull down `wlr-which-key`, build the temporary YAML configs, fetch the necessary dependencies into the store, and launch your menus on the spot.

[[Turning Main And Dendritic Host Into Modules While Maintaining Build Integrity And Function Of Main Config]]
[[Import&Filter Hosts NixOS&Dendritic]]