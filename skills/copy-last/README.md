# copy-last

Copies the last Claude response to your clipboard in either raw markdown or formatted HTML.

## Usage

```bash
/copy-last              # Choose format interactively
/copy-last raw          # Copy as raw markdown
/copy-last html         # Copy as formatted HTML (pastes into Gmail, Notion, email, etc.)
```

## Installation

Requires Python `markdown` package:

```bash
pip3 install markdown
```

## macOS dependencies

- `textutil` - Built-in, converts HTML to RTF for rich text pasting
- `pbcopy` - Built-in, copies to clipboard
