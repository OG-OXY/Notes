# Note Strategies & Workflows Summary

## Here is the short, concise summary of everything covered, structured as a clean comparison of your note-taking strategy options and a clear breakdown of the workflow choices:

### Summary of Note Strategies & Workflows

* **Capture Rule:** Fast raw drafting *must* stay in Helix using Markdown to prevent breaking your development flow.
* **The Destination:** Org-mode is chosen for its unmatched depth in knowledge engineering, structural nesting, and Org-roam graph capabilities.
* **The Bridge:** An automated Fish shell inbox-and-flush script (`notes-sync`) handles converting raw Markdown to Org files, moving them to your vault, and cleaning the inbox automatically.

---

### Pros & Cons of the Opposing Choices

#### 1. Helix + Fish Inbox-to-Org Pipeline (Your Chosen Path)

* **Pros:**
* Zero-friction capture speed inside Helix (no context switching).
* Leverages Org-mode's maximum power for knowledge structure and graph linking.
* Fully automated clean-up via shell scripts avoids manual tracking overhead.


* **Cons:**
* Requires managing an intermediate conversion script (Pandoc).



#### 2. Pure Markdown via Denote (The Alternative Shortcut)

* **Pros:**
* Completely skips the Pandoc conversion step since notes stay as `.md` files forever.
* Native Emacs linking and IDs via standard filename conventions.


* **Cons:**
* Lacks advanced Org-mode features like block execution, complex table nesting, and deep property drawers.



#### 3. Pure Terminal TUI Ecosystem (`zk` + `fzf` + `glow`)

* **Pros:**
* Absolute zero bloat, lightning fast, and 100% terminal-native.


* **Cons:**
* Falls short on rich visual presentation and advanced knowledge graph canvas compared to Obsidian or Emacs.
