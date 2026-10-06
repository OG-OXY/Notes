**Yes, absolutely.** You should prefix almost every application launcher command with `uwsm app --` when running Niri under UWSM.

When you launch a program directly from Niri (e.g., `spawn "ghostty"` or `${pkgs.ghostty}/bin/ghostty`), it runs as a **direct child process of Niri**. It inherits Niri's single main cgroup, its environment variables from the exact moment Niri booted, and its D-Bus execution context.

When you use `uwsm app --`, UWSM hands the process off to Systemd, spawning it in its own isolated `app-*.scope` unit.

---

### Why You Should Use `uwsm app --` Everywhere

#### 1. Isolated Systemd Scopes & D-Bus Cleanliness

Without `uwsm app --`, every app you open shares Niri's main process scope. This is what caused your earlier OBS/Portal D-Bus collision error (`Could not register app ID`):

* **Without UWSM:** OBS ran in the same D-Bus scope as your terminal or window manager, causing D-Bus to reject duplicate App ID bindings.
* **With `uwsm app -- obs`:** Systemd creates `app-obs@....scope`. OBS gets a completely unique D-Bus connection ID, fresh cgroup, and clean session attributes.

#### 2. Resource Limits & Clean Process Termination

If an app leaks memory, locks up, or crashes, running it under `uwsm app --` prevents it from taking down your Wayland compositor. Additionally, closing the app cleans up all child processes within that scope completely.

#### 3. Fresh Environment Variables

`uwsm app` ensures the launched app gets the latest environment variables exported to the Systemd user session (like updated `WAYLAND_DISPLAY` or `XDG_CURRENT_DESKTOP`).

---

### How to Format Your Binds (Niri & `wlr-which-key`)

#### 1. Standard GUI Applications

Prefix the binary or store path directly with `uwsm app --`:

* **In Niri (`config.kdl`):**
```kdl
binds {
    Mod+Return { spawn "uwsm" "app" "--" "ghostty"; }
    Mod+B      { spawn "uwsm" "app" "--" "firefox"; }
    Mod+O      { spawn "uwsm" "app" "--" "obs"; }
}

```


* **In `wlr-which-key` (Nix expression):**
```nix
# Direct store path call
exec = "uwsm app -- ${pkgs.obs-studio}/bin/obs";

```



---

#### 2. Terminal Applications

When launching CLI tools inside a terminal emulator like Ghostty, put `uwsm app --` in front of the terminal launcher itself:

* **In Niri (`config.kdl`):**
```kdl
binds {
    Mod+Shift+T { spawn "uwsm" "app" "--" "ghostty" "-e" "btop"; }
}

```


* **In `wlr-which-key` (Nix expression):**
```nix
exec = "uwsm app -- ${pkgs.ghostty}/bin/ghostty -e ${pkgs.yazi}/bin/yazi";

```



---

#### 3. Chaining / Piping Multiple Commands (`&&`, `||`, `|`)

`uwsm app --` expects a single command executable and its direct arguments. It **does not parse shell operators** like `&&` or `|` directly.

If you need to chain commands, pipe outputs, or run inline shell scripts, wrap the chain inside a shell invocation:

* **In Niri / `wlr-which-key`:**
```nix
# Wrap chained commands inside bash/sh
exec = "uwsm app -- ${pkgs.bash}/bin/bash -c '${pkgs.maim}/bin/maim | ${pkgs.wl-clipboard}/bin/wl-copy'";

```



---

### Summary Rules

| Command Type | How to Format |
| --- | --- |
| **Standard GUI App** | `uwsm app -- ${pkgs.app}/bin/app` |
| **Terminal + CLI Tool** | `uwsm app -- ${pkgs.ghostty}/bin/ghostty -e ${pkgs.tool}/bin/tool` |
| **Chained / Piped Script** | `uwsm app -- ${pkgs.bash}/bin/bash -c 'cmd1 && cmd2'` |