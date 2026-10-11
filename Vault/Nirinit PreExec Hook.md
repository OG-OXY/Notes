To fix the dynamic DRM connector mismatch in `nirinit`'s session file, we can write a quick pre-start script that inspects your active kernel outputs and patches `session.json` on the fly right before the `nirinit` service loads.

1. **Create a Sanitizer Script:** Fish script.
Create a script at `~/.local/bin/nirinit-sanitize.fish` that reads active connectors from `/sys/class/drm/` and updates the JSON file:

```shell
#!/usr/bin/env fish
set -l session_file "$HOME/.local/share/nirinit/session.json"
if not test -f $session_file; exit 0; end

# Find the first active connected DRM card output dynamically
for card in /sys/class/drm/card*-*
    if test -f "$card/status" -a (cat "$card/status") = "connected"
        set -l active_output (basename $card)
        # Use sed/string replacement to normalize old connectors in session.json to the live one
        # (or map them dynamically based on your layout needs)
        sed -i -E "s/\"output\": \"(DP-[0-9]+|HDMI-A-[0-9]+)\"/\"output\": \"$active_output\"/g" $session_file
        break
    end
end

```

*Verification:* Run `chmod +x ~/.local/bin/nirinit-sanitize.fish` and execute it manually once to verify it updates `session.json` without errors.


2. **Hook into Systemd Pre-Start:** Systemd service.
Add an override to your user service so this script runs automatically before `nirinit` starts up:

```nix
systemd.user.services.nirinit = {
  serviceConfig.ExecStartPre = "%h/.local/bin/nirinit-sanitize.fish";
};

```

*Verification:* Run `systemctl --user daemon-reload` and check that restarting `nirinit` cleanly executes the pre-start hook.

You are completely right—bash is more than enough here. Rust would be absolute overkill for a simple JSON patching utility, especially when standard text processing tools like `jq` or even a clean Python/Fish/Bash script can modify the text file instantly without compilation steps.

Since `nirinit` reads the JSON file straight off the disk, we can handle this entirely with a lightweight script that runs right before the service starts up.

1. **Create the Sanitizer Script:** Bash/Fish.
Create a script at `~/.local/bin/nirinit-sanitize.sh` that checks your active DRM connectors and replaces stale output names in your session file:

```bash
#!/usr/bin/env bash
SESSION_FILE="$HOME/.local/share/nirinit/session.json"
[[ -f "$SESSION_FILE" ]] || exit 0

# Grab the first active connected DRM connector
ACTIVE_OUTPUT=""
for card in /sys/class/drm/card*-*; do
    if [[ -f "$card/status" ]] && [[ "$(cat "$card/status")" == "connected" ]]; then
        ACTIVE_OUTPUT="$(basename "$card")"
        break
    fi
done

[[ -n "$ACTIVE_OUTPUT" ]] && sed -i -E "s/\"output\": \"(DP-[0-9]+|HDMI-A-[0-9]+)\"/\"output\": \"$ACTIVE_OUTPUT\"/g" "$SESSION_FILE"

```

*Verification:* Run `chmod +x ~/.local/bin/nirinit-sanitize.sh` and execute it manually once to verify it updates `session.json` without errors.


2. **Hook into Systemd Pre-Start:** NixOS Configuration.
Add an `ExecStartPre` directive to your user service so it patches the file automatically every time the service initializes:

```nix
systemd.user.services.nirinit = {
  serviceConfig.ExecStartPre = "%h/.local/bin/nirinit-sanitize.sh";
};

```

*Verification:* Run `systemctl --user daemon-reload` and check `systemctl --user status nirinit` to ensure the service runs the pre-start hook successfully.