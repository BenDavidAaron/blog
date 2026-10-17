# Hugo blog

This directory contains a small Hugo blog. It uses Markdown content and GitHub Pages.

## Local use

Install Hugo, then run:

```sh
hugo server -D
```

Open `http://localhost:1313/` in a browser.

Create a post with:

```sh
hugo new content posts/my-post.md
```

Set `draft: false` before publication.

## GitHub Pages

Push the repository to GitHub and enable Pages in repository settings.

Select **GitHub Actions** as the build source. The workflow in `.github/workflows/pages.yml` builds and deploys the site after each push to `main`.

Update `title`, `description`, and `baseURL` in `hugo.yaml` for local use and metadata. The workflow supplies the GitHub Pages URL during its build.

