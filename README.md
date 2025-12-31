# CatKinKitKat.github.io

My personal site. Static, built with Zola, deployed to GitHub Pages. It's basically a digital notebook for backend war stories and infrastructure notes.

Yes, this is roughly the fourth iteration. Don't ask about the previous ones.

## Running Locally

**Prerequisites:** Zola v0.20+ and Git.

```sh
git clone --recursive https://github.com/CatKinKitKat/CatKinKitKat.github.io.git
cd CatKinKitKat.github.io
zola serve
```

If you forgot `--recursive`, fix the submodule: `git submodule update --init --recursive`

Site will be at `http://127.0.0.1:1111`. Build with `zola build` (output goes to `public/`).

## Structure

- `config.toml`: Site configuration (Title, theme, navigation).
- `content/`: Markdown files containing the actual content.
- `static/`: Custom CSS, JS, and the theme toggle click sound.
- `templates/`: Overrides for the anemone theme (mostly date handling).
- `writing_style_profile.md`: The writing blueprint (gitignored).

## Deployment

Pushing to `master` triggers the GitHub Actions workflow (`.github/workflows/deploy.yml`). It builds with Zola and deploys to Pages. There's also a manual trigger if you're impatient.

## Theme

[anemone](https://github.com/Speyll/anemone), managed as a submodule under `themes/anemone`. Minimal, dark mode support, doesn't ship a megabyte of JavaScript to render text.
