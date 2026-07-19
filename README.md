# happywalkers.github.io

Yunpeng's content-first Jekyll site, hosted by GitHub Pages. The design is a customized, single-column interpretation of [`no-style-please`](https://github.com/riggraz/no-style-please): a profile introduction followed by compact image-and-title post rows.

## Adding a post on GitHub

Create a Markdown file under `_posts` using the name `YYYY-MM-DD-title.md`:

```yaml
---
title: My new post
tags:
  - Robotics
cover: /assets/images/my-cover.png
---

Post content goes here.
```

The `thumbnail` field is optional. When it is omitted, the homepage uses `cover` as the thumbnail. Commit to the configured Pages branch to trigger the managed Jekyll build.

## Local preview

The `Gemfile` matches GitHub Pages' pinned dependency bundle:

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://127.0.0.1:4000/>.

For the closest GitHub Pages build parity, use GitHub's build container:

```bash
docker run --rm \
  -e GITHUB_WORKSPACE=/github/workspace \
  -e INPUT_SOURCE=. \
  -e INPUT_DESTINATION=./_site \
  -e INPUT_FUTURE=false \
  -e GITHUB_REPOSITORY=HappyWalkers/happywalkers.github.io \
  -v "$PWD:/github/workspace" \
  ghcr.io/actions/jekyll-build-pages:v1.0.13

python3 -m http.server 4000 --directory _site
```

## Site structure

- `_config.yml`: profile, URL, plugin, and publishing settings
- `_posts/`: Markdown posts
- `_layouts/`: homepage, post, page, and error templates
- `_includes/post-list.html`: thumbnail/title feed
- `assets/css/main.scss`: responsive single-column design
- `assets/images/covers/`: local full-size cover images
- `assets/images/thumbnails/`: optimized feed thumbnails

Existing dated permalinks are intentionally preserved.
