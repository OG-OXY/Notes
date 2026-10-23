Ah, that exact annoyance with `x` is a universal Helix rite of passage!

You press `x` three times to grab three lines, accidentally hit it a fourth time, and suddenly you've captured half the function below what you wanted. Out of the box, uppercase `X` doesn't go backwards—it just extends the current selection to line bounds. So you're left awkwardly hitting `v` and scrolling back up, or undoing and restarting.

The good news? Because Helix has underlying commands for moving upwards (`extend_line_above`), **you can completely fix this in your Helix `config.toml**` so you have a true reverse-x!

### The Fix: Add Upwards Line Selection to `X`

If you want capital `X` to select *upwards* instead of its redundant default behavior, drop this into your Helix `config.toml` (which you can manage right inside your Nix configuration tree):

```toml
[keys.normal]
X = "extend_line_above"

[keys.select]
X = "extend_line_above"

```

Once you map that, `x` expands your selection downwards line-by-line, and capital `X` expands it upwards line-by-line. If you over-select going down, you can just tap `X` to claw the selection back upwards instantly.

*(Bonus tip for pasting blocks: if you paste something and it's sitting at the wrong indentation level, instead of manually selecting lines to fix it, you can just select the block and use `<` or `>` to shift the entire indented block left or right atomically without messing with manual spacing!)*

[Learn more about navigating the codebase with [Helix editor tutorial](https://www.youtube.com/watch?v=vvhIIMd_vW4) to speed up your workflow further.]

[[Oct 20 TODO]]