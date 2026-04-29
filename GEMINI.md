# Blog Translation Workflow (On-Demand)

I will only translate files that are explicitly passed as arguments. I will not scan the directory or automatically sync files in the background.

### Translation Command: "Translate [file_path]"

When a file path is provided for translation:

1.  **Language Detection:**
    *   If the file ends in `.ko.md`, the source is **Korean** and the target is **English** (`.md`).
    *   If the file ends in `.md` (and not `.ko.md`), the source is **English** and the target is **Korean** (`.ko.md`).

2.  **Execution:**
    *   Read the source file.
    *   Translate to the target language.
    *   **STRICT RULES:**
        *   Preserve Hugo front matter (`+++` or `---`).
        *   Do not translate technical terms (Linux, Kernel, LKM, Socket, system call, etc.).
        *   Maintain all Markdown formatting and links.
    *   Write the result to the target file path.

3.  **Safety:**
    *   If the target file already exists, **always ask for confirmation** before overwriting it to protect manual refinements.

### Cleanup Command: "Check for orphaned translations"
*   List files that have a translation but no source (or vice versa) and ask if they should be deleted.
