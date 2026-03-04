# Dev Blog

Hugo blog using the [Stack theme](https://github.com/CaiJimmy/hugo-theme-stack) (v3.34.2).

## Project Structure

- `content/post/<slug>/index.md` — blog posts (Hugo page bundles)
- `content/post/<slug>/images/` — post images (referenced as `images/filename.png`)
- `assets/scss/custom.scss` — all style overrides (theme loads this last, so it wins)
- `layouts/shortcodes/` — custom shortcodes: `alert`, `code-hint`, `code`, `prompt`
- `config/_default/` — Hugo configuration (params.toml, config.toml)

## Conventions

### Blog Posts
- Frontmatter: title, description, slug, date, image, categories, tags, links
- Code blocks: use `{linenos=false}` for blocks where line numbers don't add value
- File paths above code blocks: `{{< code-hint "filename" "path/to/file" >}}`
- Prompt blocks (for quoting user prompts to Claude): `{{< prompt >}}...{{< /prompt >}}`

### Style Overrides
- All CSS customisation goes in `assets/scss/custom.scss`
- Dark mode: use `&[data-scheme="dark"]` inside `:root` for variables, or `[data-scheme="dark"] .class` for rules
- Theme variables are in the cached module at `assets/scss/variables.scss` — override, don't edit the theme
