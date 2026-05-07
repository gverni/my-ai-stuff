---
name: copy-last
description: |
  Copies the last Claude response to the clipboard in raw markdown or HTML format.
  Use when the user wants to copy, export, or share the last response.
  Triggers on: /copy-last, /copy-last raw, /copy-last html
---

# /copy-last

Copies the last assistant response to the macOS clipboard.

## Usage

- `/copy-last raw` - Copy as raw markdown (keeps all formatting symbols)
- `/copy-last html` - Copy as rendered HTML (paste into rich text editors, Notion, email)
- `/copy-last` - Prompt user to choose format

## Instructions

When this skill is invoked:

1. Identify the last assistant message in this conversation (the message immediately before the user's /copy-last invocation). This is the content you will copy.

2. Determine the format from the argument:
   - If argument is `raw` or `markdown`: use raw mode
   - If argument is `html` or `formatted`: use html mode
   - If no argument: use AskUserQuestion to present format options to the user
     - Option 1: "Raw markdown" - Keep all formatting symbols, paste as text
     - Option 2: "Formatted HTML" - Renders as formatted text in Gmail, Notion, email, etc.
     - Read the user's choice and proceed with the selected format

3. For RAW mode:
   - Take the full text of the last assistant message exactly as written (with all markdown symbols)
   - Use Bash to pipe it to pbcopy using a heredoc:
     ```
     pbcopy <<'CLIPBOARD_EOF'
     <content here>
     CLIPBOARD_EOF
     ```
   - Confirm: "Copied to clipboard as raw markdown."

4. For HTML mode (pastes as formatted content in Gmail, Notion, email, etc.):
   - Take the full text of the last assistant message
   - Convert markdown to HTML using pandoc or Python markdown module
   - Convert the HTML to RTF format (Rich Text Format) so it pastes as formatted content
   - Copy RTF to clipboard
   
   **Workflow:**
   ```bash
   # Step 1: Convert markdown to HTML (try pandoc first, fall back to Python)
   # markdown content → HTML
   
   # Step 2: Save HTML to temp file, convert to RTF with textutil
   textutil -convert rtf /tmp/html_file.html -output /tmp/temp.rtf
   cat /tmp/temp.rtf | pbcopy
   
   # Clean up temp files
   rm /tmp/html_file.html /tmp/temp.rtf
   ```
   
   - If conversion fails, inform the user:
     "Cannot convert to HTML. Ensure you have Python markdown installed (`pip3 install markdown`)."
   - Confirm: "Copied to clipboard as formatted HTML (ready to paste into Gmail, Notion, email, etc.)."

## Important Notes

- The content to copy is ALWAYS the assistant message immediately before the /copy-last invocation
- Do NOT include the user's message, only the assistant response
- Preserve the full response including code blocks, tables, and lists
- For the heredoc approach, use a delimiter that won't appear in the content (e.g. CLIPBOARD_EOF)
- If the content contains single quotes or special characters, the heredoc approach handles this safely
