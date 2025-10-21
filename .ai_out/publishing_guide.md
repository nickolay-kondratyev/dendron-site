# Dendron Publishing Guide

## Overview

Dendron allows you to publish your notes as a static website using Next.js. This guide covers everything you need to know about publishing your Dendron workspace.

## Quick Start

### Prerequisites
- Install [git](https://git-scm.com/download)
- Install the latest [Dendron CLI](https://wiki.dendron.so/notes/RjBkTbGuKCXJNuE4dyV6G/)
  ```bash
  npm install -g @dendronhq/dendron-cli@latest
  ```
- Have your notes in a Dendron workspace (folder with `dendron.yml`)

### Basic Publishing Steps

1. **Initialize publishing** (from workspace root):
   ```bash
   npx dendron publish init
   ```

2. **Preview locally**:
   ```bash
   npx dendron publish dev
   ```
   Visit `http://localhost:3000` to see your site

3. **Build for production**:
   ```bash
   npx dendron publish build
   ```

4. **Export static site**:
   ```bash
   npx dendron publish export
   ```
   Static files will be in `.next/out/`

## Configuration

### Essential Settings (dendron.yml)

```yaml
publishing:
  # Required: URL where your site will be hosted
  siteUrl: https://example.com
  
  # What notes to publish (default: all)
  siteHierarchies:
    - root  # publishes everything
    # OR specify specific hierarchies:
    # - dendron
    # - projects
    # - blog
  
  # Homepage note (defaults to first hierarchy)
  siteIndex: home
  
  # SEO settings
  seo:
    title: My Knowledge Base
    description: Everything I know
    author: Your Name
    twitter: https://twitter.com/yourhandle
```

### Publishing Specific Hierarchies

To publish only certain parts of your vault:

```yaml
publishing:
  siteHierarchies:
    - blog
    - projects
  siteIndex: blog.index  # Set homepage
```

### Per-Note Publishing Control

Control individual notes with frontmatter:

```yaml
---
published: false  # This note won't be published
---
```

Or set hierarchies to not publish by default:

```yaml
publishing:
  hierarchy:
    private:
      publishByDefault: false  # Only publish notes with 'published: true'
```

## Deployment Options

### GitHub Pages with GitHub Actions (Recommended)

1. **Create GitHub repository**
2. **Add workflow file** `.github/workflows/publish.yml`:

```yaml
name: Build Dendron Static Site

on:
  workflow_dispatch:
  push:
    branches:
    - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout source
      uses: actions/checkout@v2
      with:
        fetch-depth: 0

    - name: Restore Node modules cache
      uses: actions/cache@v2
      id: node-modules-cache
      with:
        path: |
          node_modules
          .next/*
          !.next/.next/cache
          !.next/.env.*
        key: ${{ runner.os }}-dendronv2-${{ hashFiles('**/yarn.lock', '**/package-lock.json') }}

    - name: Install dependencies
      run: yarn

    - name: Initialize or pull nextjs template
      run: "(test -d .next) && (echo 'updating dendron next...' && cd .next && git reset --hard && git pull && yarn && cd ..) || (echo 'init dendron next' && yarn dendron publish init)"

    - name: Restore Next cache
      uses: actions/cache@v2
      with:
        path: .next/.next/cache
        key: ${{ runner.os }}-nextjs-${{ hashFiles('.next/yarn.lock', '.next/package-lock.json') }}-${{ hashFiles('.next/**.[jt]s', '.next/**.[jt]sx') }}

    - name: Export notes
      run: yarn dendron publish export --target github --yes

    - name: Deploy site
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_branch: pages
        publish_dir: docs/
        force_orphan: true
        #cname: example.com  # Uncomment for custom domain
```

3. **Configure GitHub Pages**:
   - Go to Settings ’ Pages
   - Source: `pages` branch
   - Folder: `/ (root)`

4. **Update dendron.yml**:
```yaml
publishing:
  siteUrl: https://USERNAME.github.io
  assetsPrefix: /REPO_NAME  # Only if repo name ` username
```

### Netlify

1. **Create `dendron-publish-site.sh`**:
```bash
#!/usr/bin/env bash
rm -rf docs
yarn
(test -d .next) && (echo 'updating dendron next...' && cd .next && git reset --hard && git pull && yarn && cd ..) || (echo 'init dendron next' && yarn dendron publish init)
yarn dendron publish export
mv .next/out docs
```

2. **Create `netlify.toml`**:
```toml
[build]
  publish = "docs"
  command = "./dendron-publish-site.sh"

[build.environment]
  NETLIFY_NEXT_PLUGIN_SKIP = "true"

[[plugins]]
  package = "@netlify/plugin-lighthouse"

[[plugins]]
  package = "@netlify/plugin-nextjs"
```

3. **Connect repository in Netlify dashboard**

### Localhost

For local hosting (e.g., personal wiki):

```bash
# Build
npx dendron publish build
cd .next && npm run build

# Start server
npm run start  # Runs on localhost:3000
```

## Advanced Features

### Custom Sidebar

Create `sidebar.js`:

```javascript
module.exports = [
  {
    type: 'category',
    label: 'Getting Started',
    link: { type: 'note', id: 'intro' },
    items: [
      { type: 'note', id: 'quickstart' },
      { type: 'note', id: 'tutorial' }
    ]
  },
  {
    type: 'autogenerated',
    id: 'guides'  # Auto-generate from guides hierarchy
  }
];
```

Update `dendron.yml`:
```yaml
publishing:
  sidebarPath: './sidebar.js'
```

### Enable Comments (Giscus)

1. **Configure Giscus** at https://giscus.app
2. **Add to dendron.yml**:
```yaml
publishing:
  giscus:
    repo: "username/repo"
    repoId: "your-repo-id"
    category: "Announcements"
    categoryId: "your-category-id"
    mapping: "pathname"
    theme: "preferred_color_scheme"
    inputPosition: "top"
```

3. **Enable per note**:
```yaml
---
enableGiscus: true
---
```

### Themes and Styling

Add custom CSS/JS with `header.html`:

```html
<!-- Custom fonts, analytics, styles -->
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet">
<style>
  :root {
    --dendron-primary: #007ACC;
  }
</style>
```

```yaml
publishing:
  customHeaderPath: header.html
```

## Troubleshooting

### Common Issues

**Build fails with "Command not found"**
- Ensure npm/yarn is installed
- Check VS Code's default shell has npm access

**Pages not appearing**
- Verify hierarchies are listed in `siteHierarchies`
- Check notes have `published: true` if needed
- Ensure no duplicate notes across vaults

**Local preview is slow**
- This is normal - pages build on demand
- Production builds are pre-built and fast

### Upgrading

To update publishing:
```bash
cd .next
git reset --hard
git pull
npm install
```

## Best Practices

1. **Use GitHub Actions** for automated publishing
2. **Test locally** with `dendron publish dev` before deploying
3. **Configure SEO** settings for better discoverability
4. **Use hierarchies** to organize published content
5. **Set up redirects** when migrating from 11ty
6. **Enable caching** in CI/CD for faster builds

## Migration from 11ty

If upgrading from legacy 11ty publishing:
1. Pretty URLs no longer use `.html` suffix
2. Set up redirects to preserve old links
3. Update `dendron-cli` to latest version
4. Run configuration migration if prompted

## Additional Resources

- [Dendron Publishing Documentation](https://wiki.dendron.so/notes/4ushYTDoX0TYQ1FDtGQSg/)
- [GitHub Pages Template](https://github.com/dendronhq/template.publish.github-action)
- [Netlify Template](https://link.dendron.so/6WuI)
- [Discord #publishing Channel](https://discord.gg/dendron)