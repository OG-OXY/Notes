# Turning `sort_sensitive = true` On in Yazi
>gives a whole second layer to organize FHS

That's actually a super clever way to exploit ASCII ordering. Once `sort_sensitive = true` is on, the exact byte value of the first character dictates the exact shelf where the file sits.

ASCII prioritizes characters in this order:

1. **Special characters / Symbols:** `!` `#` `$` `%` `&` `.` `_`
2. **Numbers:** `0-9`
3. **UPPERCASE:** `A-Z`
4. **lowercase:** `a-z`

You essentially get four built-in "priority tiers" inside a single directory without creating subfolders:

```text
.hidden_config         # Tier 0: Hidden dotfiles
_URGENT_NOTES.md       # Tier 1: Symbols (pin to absolute top)
01_ACTIVE_PROJECT/     # Tier 2: Digits (ordered sequences)
README.md              # Tier 3: UPPERCASE (important root files)
main.py                # Tier 4: lowercase (standard work files)
notes.txt

```

If you ever want to leverage symbols even further, `_` and `~` sit at opposite ends of the spectrum—`_` (ASCII 95) sits above capital letters, while `~` (ASCII 126) sits at the absolute bottom below lowercase letters, giving you a top-pin and a bottom-pin character.

[[THE WAY]]