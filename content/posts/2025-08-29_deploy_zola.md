+++
title = "How to deploy zola on github pages"
date = "2025-08-29"

[taxonomies]
tags = ["tutorial"]
+++


### Intro

This blog is built with Zola (written in Rust), and since it just builds a static page without any database or server-side components, it's a prime candidate for hosting with GitHub Pages.

### The official way

The current documentation has a page on how to deploy to GitHub Pages, which works and was the first approach on how this blog is deployed, but is considered "legacy"1. It uses the shalzz/zola-deploy-action action to build the zola site and commit the static files to the gh-pages branch. Once pushed, a built-in GitHub action pages-build-deployment takes the files from that branch and actually deploys the pages. So in the end, the workflow is as follows:

- A push to main triggers the shalzz/zola-deploy-action action
- The shalzz/zola-deploy-action action pushes the static files to the gh-pages branch
- A push to the gh-pages branch triggers pages-build-deployment, which does the actual deployment


### The better way

The preffered and recommended way is to deploy directly within the action, using upload-pages-artifact and deploy-pages. We can combine these inside a single workflow, which means we no longer require a push to the gh-pages branch and don't rely on the second action to do the actual deployment from that branch.

To make this change, you need to update your github action file (see below), and in your repository on GitHub, go to Settings -> Pages -> Build and deployment -> Change Source to GitHub Actions.

Here is an adjusted workflow:

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
        run: sudo snap install --edge zola

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
      url: ${{ steps.deployment.outputs.page_url }}

    # Specify runner + deployment step
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```



This results in a much quicker deployment, and only requires one action to run. If you had used the old approach before, you can now delete the gh-pages branch as well.
