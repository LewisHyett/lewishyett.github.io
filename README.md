# lewishyett.github.io

A dummy GitHub Pages blog suite scaffolded with Jekyll + Minima.

## Local preview

```bash
bundle install
bundle exec jekyll serve
```

Open http://localhost:4000.

## Structure

| Path          | What it is                        |
|---------------|-----------------------------------|
| `_config.yml` | Site config, theme, plugins       |
| `index.md`    | Home page (post list)             |
| `about.md`    | About page                        |
| `_posts/`     | Blog posts (`YYYY-MM-DD-slug.md`) |

## Deploy

Deployment runs through GitHub Actions (`.github/workflows/jekyll.yml`), not the
legacy `github-pages` gem build.

One-time setup: on GitHub, **Settings → Pages → Build and deployment → Source →
GitHub Actions**.

After that, every push to `main` builds and publishes to
`https://lewishyett.github.io`.

All content is placeholder — replace before going live.
