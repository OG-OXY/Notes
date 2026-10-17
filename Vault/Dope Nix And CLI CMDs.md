check references of where a package belongs or where its coming from
nix-store -q --references $(which wpctl) | head
