# Programming Learning

Personal, systems-based programming mastery library, published as a Jekyll site on GitHub Pages. Each language is a hand-built curriculum: internals and execution model first, then types/data model, then idioms and tooling, then performance and production. Goal is deep, interview-ready fluency, not framework trivia. The author is an experienced JS/TS engineer (Python material compares against JS/TS).

## Layout

```
index.md            # hub: one .lang-card per track (the only place tracks are listed)
_config.yml         # root Jekyll config
assets/css/style.scss   # hub styling, incl. .lang-badge.<code> colors per track
<Track>/            # JavaScript, Python, TypeScript, Rust
  _config.yml       # per-track config (title, description, theme, markdown: kramdown)
  index.md          # track landing page (front matter: layout: default, title)
  00_roadmap.md     # progress/status; lessons 01..N follow
```

## Conventions per track

| Track | File naming | Notes |
|---|---|---|
| JavaScript | `NN-kebab-name.md` | 12 chapters + answer key + `14-mechanism-drill.html` |
| Python | `NN_snake_name.md` | 20 modules + answer key (`21_answer_key_00_18.md`) |
| TypeScript | `NN-kebab-name.md` | 16 chapters + exercises reference |
| Rust | `NN_snake_name.md` | 60 lessons, edition 2024, Solana/Substrate/EVM at the end |

- Every `.md` needs YAML front matter (`title:`, plus `layout: default` on index pages) or Jekyll will not render it to HTML.
- Keep the existing naming style within a track; do not mix `-` and `_`.
- Lessons are cumulative: roadmap's "Current position" / lesson counts must be updated whenever lessons are added.
- Rust lessons state their module, edition, and the exact `rustc` version used to verify code. Code samples must be compiled/run before being claimed as verified.

## Adding a new track

1. Create `<Track>/` with `_config.yml`, `index.md`, `00_roadmap.md`, and lessons.
2. Add a `.lang-card` to root `index.md` (badge code, name, meta line, status `complete` or `planned`).
3. Add a `.lang-badge.<code>` color rule in `assets/css/style.scss`.

## Editing guidelines

- Update the hub card meta line (chapter/lesson counts) when a track's size changes.
- Preserve the teaching tone: mechanism first, explain why, include interview-relevant framing.
- Don't rename or renumber lesson files without fixing every internal link (grep for the old name).
- `.DS_Store` is gitignored; don't commit it.
