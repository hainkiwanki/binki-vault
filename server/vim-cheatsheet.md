# Vim cheatsheet

Quick reference for editing files (like `.env`) on the server with Vim.

Vim is a *modal* editor: keys do different things depending on the mode you're in. Most confusion comes from typing while in the wrong mode.

- **Normal mode** (default): keys are commands (move, delete, save, quit).
- **Insert mode**: keys type text. The bottom line shows `-- INSERT --`.

> If you pressed **Ctrl+X** expecting nano's "exit", Vim enters a completion submode and shows `-- ^X mode (^]^D^E^F^I^K^L^N^O^Ps^U^V^Y)`. Just press **Esc**.

## Getting out

1. Press **Esc** once or twice to return to Normal mode.
2. Type one of these and press **Enter**:
   - `:wq` saves and quits.
   - `:q!` quits **without** saving.

Rule of thumb: **Esc first, then a `:` command.**

## Basics

| Key | What it does |
|---|---|
| `i` | Start typing (Insert mode) at the cursor |
| `Esc` | Stop typing and go back to Normal mode |
| Arrow keys, or `h` `j` `k` `l` | Move the cursor (left, down, up, right) |
| `o` | Add a new line below and start typing |
| `x` | Delete the character under the cursor |
| `dd` | Delete the whole line |
| `u` | Undo |
| `Ctrl+R` | Redo |
| `/text` then `Enter` | Search for "text"; press `n` for the next match |
| `G` / `gg` | Jump to the end / start of the file |
| `:w` | Save |
| `:q` | Quit |

## Using nano instead

To skip Vim, open the file with nano, where **Ctrl+O** saves and **Ctrl+X** exits:

```bash
nano .env
```

You can also make nano the default editor for tools that open one for you, like `git commit` or `crontab -e`:

```bash
export EDITOR=nano
```

To keep that setting, add the line to your `~/.bashrc`.
