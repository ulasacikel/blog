# Security Research Blog

This site is built with Hugo and the Devise theme. It has been tested with Hugo 0.167.0.

## Run locally

On macOS, install Hugo with `brew install hugo`. From the repository root:

```sh
git submodule update --init --recursive
hugo server --buildDrafts --renderToMemory
```

Open the local URL printed by Hugo (normally <http://localhost:1313/>). The server reloads when you edit content or configuration. Stop it with Ctrl+C.

## Add a post

```sh
hugo new content post/my-finding.md
```

Edit the new Markdown file in `content/post/`. New posts are drafts by default, so `--buildDrafts` includes them in the local preview. Put images under `static/images/` and link to them as `/images/filename.png`. When the post is ready to publish, set `draft = false` in its front matter.

Run `hugo` to generate the publishable site in `public/`.
