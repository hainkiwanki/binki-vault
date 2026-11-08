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
color: '#6aa84f'
sections: [bosses, crafting]
```

If a `space.yml` is invalid, Vault still lists the space and says what's wrong with it.

Images and videos uploaded in Vault live in RustFS, not here. Favorites, the wishlist and the command list live in Vault's database.
