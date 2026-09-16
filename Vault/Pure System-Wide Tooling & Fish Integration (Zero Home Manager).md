# Comprehensive Guide: Pure System-Wide NixOS Tooling & Fish Integration (Zero Home Manager)

This guide outlines how to configure a fully deterministic, dotfileless NixOS system where tools, shell completions, and environment wrappers (like the Yazi `cd` hook and CLI initialization scripts) are handled entirely system-wide via core NixOS modules and `/etc`, eliminating Home Manager completely.

---

## 1. Core NixOS Fish Enablement & Global Hooks

Enable Fish natively as the system shell and populate `/etc/fish/conf.d/` with initialization scripts. This replaces all Home Manager `enableFishIntegration` flags.

```nix
{ pkgs, ... }:

{
  # Enable Fish globally on the system
  programs.fish.enable = true;

  # Set fish as default shell for users if desired
  users.defaultUserShell = pkgs.fish;

  # Install CLI tools globally so binaries and vendor completions are exposed
  environment.systemPackages = with pkgs; [
    fish
    yazi
    zoxide
    atuin
    direnv
    bat
    glow
  ];

  # Centralized system-wide fish configuration and tool hooks
  environment.etc = {
    # Zoxide global integration hook
    "fish/conf.d/zoxide.fish".text = ''
      if type -q zoxide
          zoxide init fish | source
      end
    '';

    # Atuin global integration hook
    "fish/conf.d/atuin.fish".text = ''
      if type -q atuin
          atuin init fish --disable-up-arrow | source
      end
    '';

    # Direnv global hook
    "fish/conf.d/direnv.fish".text = ''
      if type -q direnv
          direnv hook fish | source
      end
    '';

    # Yazi automatic directory changing wrapper function
    "fish/conf.d/yazi.fish".text = ''
      function y
          set tmp (mktemp -t "yazi-cwd.XXXXXX")
          yazi $argv --cwd-file="$tmp"
          if set cwd (cat -- "$tmp"); and [ -n "$cwd" ]; and [ "$cwd" != "$PWD" ]
              builtin cd -- "$cwd"
          end
          rm -f -- "$tmp"
      end
    '';
  };
}

```

---

## 2. Managing Global Tool Configuration Files via `/etc`

Instead of letting applications drop configuration files into `~/.config/`, use `environment.etc` to enforce global defaults for all users.

```nix
{
  # Global Yazi configurations
  environment.etc."yazi/yazi.toml".text = ''
    [manager]
    ratio = [1, 4, 3]
    show_hidden = true
    sort_by = "modified"
    sort_dir_first = true

    [preview]
    tab_size = 2
    max_width = 600
    max_height = 900
  '';

  environment.etc."yazi/keymap.toml".text = ''
    [[manager.prepend_keymap]]
    on = [ "T" ]
    run = "plugin toggle-pane"
    desc = "Hide or show the preview pane"
  '';
}

```

---

## 3. Ensuring Global Fish Completions Path

NixOS automatically exposes package completions via `/run/current-system/sw/share/fish/vendor_completions.d/`. To ensure Fish indexes them properly across all user sessions without local configuration:

```nix
{
  environment.variables = {
    FISH_VENDOR_COMPLETIONS_PATH = "/run/current-system/sw/share/fish/vendor_completions.d";
  };
}

```

---

## Verification Checklist

After applying your configuration changes via `sudo nixos-rebuild switch`:

1. **Verify Shell Integration:** Open a new terminal instance (`fish`) and check if `zoxide`, `atuin`, and `direnv` commands or hooks are immediately responsive.
2. **Verify Yazi Wrapper:** Type `y` to launch Yazi, navigate to a nested directory, and exit. Confirm that your terminal shell automatically changes working directory to match.
3. **Verify Completions:** Type a tool name (e.g., `git` or `atuin`) followed by `Tab` to ensure autocompletions load natively from the system path.