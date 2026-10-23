To delete a line using `sed`, ==use the `d` command combined with the specific line number or text pattern you want to target==.

## 1. Delete by Line Number

To delete an exact line number, use the number followed immediately by `d`.

- Delete a single line (e.g., line 5):
    
    ```bash
    sed -i '5d' filename.nix
    ```
    
- Delete a range of lines (e.g., lines 104 to 114) [Query-relevant Context]:
    
    ```bash
    sed -i '104,114d' filename.nix
    ```
    
- Delete the very last line of a file:
    
    ```bash
    sed -i '$d' filename.nix
    ```
    

## 2. Delete by Matching Text

If you do not know the line number, you can delete any line that contains a specific phrase by wrapping the text in forward slashes `/text/`.

- Delete lines containing a word (e.g., delete any line containing `openssh`):
    
    ```bash
    sed -i '/openssh/d' filename.nix
    ```
    
- Delete commented lines (lines starting with `#`):
    
    ```bash
    sed -i '/^#/d' filename.nix
    ```
    

## What the Flags Mean

- `-i`: Tells `sed` to edit the file in-place (saving the changes directly to the file instead of just printing the output to your screen).
[[Markdown syntax]]