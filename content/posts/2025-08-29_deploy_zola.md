+++
title = "How to deploy zola on github pages"
date = "2025-08-29"
description = "How this site is built with Zola 0.23.4 and deployed to GitHub Pages from one Actions workflow."

[taxonomies]
tags = ["DevOps", "tutorial"]
+++

### Intro

This blog is built with Zola (written in Rust), and since it just builds a static page without any database or server-side components, it's a prime candidate for hosting with GitHub Pages. The custom domain is the file `static/CNAME`, which contains `sysrex.com`.

<!-- more -->

### The branch approach

[Zola's GitHub Pages documentation](https://www.getzola.org/documentation/deployment/github-pages/) describes two options. The one this site used first builds the site and commits the static files to a `gh-pages` branch, via [shalzz/zola-deploy-action](https://github.com/shalzz/zola-deploy-action). A push to that branch is what the built-in `pages-build-deployment` workflow then publishes. The sequence is:

- A push to main triggers shalzz/zola-deploy-action
- That action pushes the static files to the gh-pages branch
- A push to the gh-pages branch triggers pages-build-deployment, which does the actual deployment

### Publishing the artifact

The workflow in this repository deploys from the action itself, with `upload-pages-artifact` and `deploy-pages`. One workflow builds the site and publishes it. Nothing has to land on a `gh-pages` branch first.

The workflow file is [`.github/workflows/gh-pages.yml`](https://github.com/sysrex/sysrex-website/blob/main/.github/workflows/gh-pages.yml). In the repository settings, go to Settings -> Pages -> Build and deployment and set Source to GitHub Actions.

Nivis requires Zola 0.23.4 or newer (`min_version` in the theme). The workflow pins that release tarball. `snap install --edge zola` follows the edge channel, which is not what this site builds with.

```yaml

on: push
name: Build and deploy GH Pages
jobs:
  # Build the site
  build:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: 'recursive'

      - name: Install zola
        run: |
          curl -sL https://github.com/getzola/zola/releases/download/v0.23.4/zola-v0.23.4-x86_64-unknown-linux-gnu.tar.gz | tar xz
          sudo mv zola /usr/local/bin/

      - name: Build site
        run: zola build

      - name: Upload static files as artifact
        id: deployment
        uses: actions/upload-pages-artifact@v3
        with:
          path: public/

  # Deploy to GitHub Pages
  deploy:
    # Add a dependency to the build job
    needs: build

    # Grant GITHUB_TOKEN the permissions required to make a Pages deployment
    permissions:
      pages: write      # to deploy to Pages
      id-token: write   # to verify the deployment originates from an appropriate source

    # Deploy to the github-pages environment
    environment:
      name: github-pages
      url: {{ "${{ steps.deployment.outputs.page_url }}" }}

    # Specify runner + deployment step
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

`static/CNAME` is copied into the built site, which is how GitHub Pages picks up the custom domain. If the old `gh-pages` branch is still around from the first setup, it can be deleted once this workflow is the Pages source.
