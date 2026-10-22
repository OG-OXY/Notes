- Get Doom Emacs with Org Mode [[Doom Emacs Module]]
- Learn To Org [[Emacs Org Mode]]
- Get Kakoune Editor?
- Write Ghostty Session Revival Script(Auto WorkSpace Setup)
- Can NoteTaking Be Done In Helix?
  [[Editor, Notes, Ghostty + Herdr Workflow plan]]

- Grab Steel Helix [[Helix Master and Steel]]  DONE
- Add Upward Line Selection (X) And Integrate Yazi To Helix [[Helix X]]
- Configure Doom Emacs Under System Or Stick WIth Home Module? [[Doom Emacs Configured With System]] 

# Summary
Your Immediate Action Items

    1. Fix Helix X Selection
    Drop X = "extend_line_above" into your Helix keybindings (config.toml) so reverse line selection works without manual v scrolling.

    2. Enable Yazi Picker in Helix
    Bind C-y / space + e to launch yazi with --chooser-file to eliminate manual pathing when creating files/directories.

    3. Lock Down Niri Focus Binds
    Add Mod+Escape allow-inhibiting=false { toggle-keyboard-shortcuts-inhibit; } and prefer-no-csd to config.kdl to break app input locks.

    4. Point Helix Input to Master
    Update your dendritic flake.nix input to github:helix-editor/helix/master to build bleeding-edge Helix directly from source.

    5. Wire Up nix-doom-emacs-unstraightened
    Import nix-doom-emacs-unstraightened as a flake input and enable programs.doom-emacs in your NixOS module to kill off Obsidian forever.