# binki-vault

The content behind Vault, binki's knowledge app. Vault reads this repo, so whatever is here shows up there.

## How it's laid out

- Every top-level folder with a `space.yml` is a space: a game dashboard, a project, a folder of notes, or a list.
- The folder name is the space's address in Vault (`/vault/terraria`). Use lowercase letters, digits and dashes.
- Folders starting with `_` or `.` are never spaces. `_assets/` is for images shared across spaces.
- Markdown files inside a space are its notes. A title in the frontmatter wins over the first `# heading`.

## `space.yml`

```yaml
title: Terraria
template: game # game | project | notes | collection | commands
color: "#6aa84f"
sections: [bosses, crafting]
```

If a `space.yml` is invalid, Vault still lists the space and says what's wrong with it.

## Writing notes

Notes are plain markdown with GitHub extras (tables, task lists, ~~strikethrough~~). Frontmatter is optional:

```md
---
title: Boss progression
tags: [bosses, progression]
---
```

- Link other notes with relative paths: `[Drops](../notes/drops.md#rare)`. A path starting with `/` starts at the repo root.
- Small images that belong to a note go in `assets/` next to it: `![Map](assets/map.png)`.
- Uploaded media is linked by its permanent ID: `![Screenshot](media:<id>)`. Vault's editor inserts these when you paste.

Blocks use directive syntax. A broken block shows what's wrong with it instead of disappearing.

| Block       | Write                                                                                  |
| ----------- | -------------------------------------------------------------------------------------- |
| Callout     | `:::tip` … `:::` (also `note`, `warning`); title: `:::tip[Before you start]`           |
| Collapsible | `:::details{summary="Drops"}` … `:::`                                                  |
| Tabs        | `::::tabs`, with `:::tab{title="Melee"}` … `:::` inside, then `::::`                   |
| Gallery     | `::gallery{images="assets/a.png, media:<id>"}`                                         |
| Sticky note | `:::sticky[Ominous Altar]` … `:::`; in the editor, Ctrl+Shift+S wraps the current line |

## Guides

Guides start as drafts in Vault: from **Add guide from URL** on a space, the binki extension's **Guide** button (use that one for sites with bot protection), or **Keep in Vault** in Shelf. Review the draft and click **Promote**: it lands in `<space>/guides/` with `source` and `fetched` in the frontmatter. Images stay in Vault as `media:<id>`. **Re-fetch** on a promoted guide opens a pull request from a `guides/…-refetch-…` branch, so your edits are never overwritten.

## Projects

A `project` space has three tabs: **Docs** (its notes here), **Code** (the repo in `repo:` of its `space.yml`, read-only: browse the files, docs render, code is highlighted) and **References** (links and images). The repo's markdown files are searchable with Ctrl+K. Vault never writes to a code repo.

## Lists

A `collection` space (`wishlist/`) and a `commands` space (`server-commands/`) only have their `space.yml` here, plus any notes you add; the items themselves live in Vault's database. Pasting a shop link in the wishlist reads its title, price and image. Commands can hold `{{name}}` parts that you fill in on the page before copying.

Edits made on the site are committed to `main` by **Vault**, with you as the author and an `Edited-via: vault-site` trailer. `git log --grep "Edited-via"` lists them.

Images and videos uploaded in Vault live in RustFS, not here. Favorites, the wishlist and the command list live in Vault's database.
