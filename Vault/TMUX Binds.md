# Tmux Complete Key-bind Reference Guide

Prefix key: `Ctrl + b` (Press `Ctrl + b`, release, then press the command key).

  

## 1. Session & Window Management

| **Action**              | **Keybinding**        | **Description**                         |
| ----------------------- | --------------------- | --------------------------------------- |
| **New Session**         | `tmux`                | Start a new session from the terminal   |
| **New Session (Named)** | `tmux new -s name`    | Start a named session                   |
| **Detach Session**      | `Ctrl + b`, `d`       | Leave session running in the background |
| **List Sessions**       | `tmux ls`             | Show all active sessions from terminal  |
| **Attach Session**      | `tmux attach -t name` | Reattach to a specific session          |
| **Kill Session**        | `Ctrl + b`, `$`       | Terminate the current session           |
| **New Window**          | `Ctrl + b`, `c`       | Create a new window                     |
| **Next Window**         | `Ctrl + b`, `n`       | Switch to the next window               |
| **Previous Window**     | `Ctrl + b`, `p`       | Switch to the previous window           |
| **Choose Window**       | `Ctrl + b`, `w`       | Interactive window picker menu          |
| **Rename Window**       | `Ctrl + b`, `,`       | Rename the current window               |
| **Kill Window**         | `Ctrl + b`, `&`       | Close the current window                |

## 2. Pane Splitting & Navigation

| **Action**                 | **Keybinding**           | **Description**                            |
| -------------------------- | ------------------------ | ------------------------------------------ |
| **Split Horizontally**     | `Ctrl + b`, `"`          | Split pane top/bottom                      |
| **Split Vertically**       | `Ctrl + b`, `%`          | Split pane left/right                      |
| **Navigate Panes**         | `Ctrl + b`, `Arrow Keys` | Move focus between panes                   |
| **Toggle Layouts**         | `Ctrl + b`, `Space`      | Cycle through default pane arrangements    |
| **Kill Pane**              | `Ctrl + b`, `x`          | Close the active pane                      |
| **Zoom Pane**              | `Ctrl + b`, `z`          | Maximize/restore the active pane           |
| **Show Pane Numbers**      | `Ctrl + b`, `q`          | Display index numbers on panes to jump     |
| **Convert Pane to Window** | `Ctrl + b`, `!`          | Break current pane out into its own window |

## 3. Copy Mode & Scrolling

| **Action**          | **Keybinding**  | **Description**                               |
| ------------------- | --------------- | --------------------------------------------- |
| **Enter Copy Mode** | `Ctrl + b`, `[` | Scroll back through history and copy text     |
| **Exit Copy Mode**  | `q`             | Exit scrolling/copy mode                      |
| **Search Up**       | `?`             | Search backward in history while in copy mode |
| **Paste Buffer**    | `Ctrl + b`, `]` | Paste copied text                             |

## 4. Miscellaneous & Control

| **Action**         | **Keybinding**             | **Description**                            |
| ------------------ | -------------------------- | ------------------------------------------ |
| **Command Prompt** | `Ctrl + b`, `:`            | Open the tmux command line                 |
| **Display Clock**  | `Ctrl + b`, `t`            | Show a large digital clock in the pane     |
| **Reload Config**  | `tmux source ~/.tmux.conf` | Reload configuration file (run from shell) |
