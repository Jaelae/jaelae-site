# jaelae.com

Personal IT blog and portfolio — built with [Hugo](https://gohugo.io/), hosted on [Cloudflare Pages](https://pages.cloudflare.com/).

## Quick Start

### Prerequisites
- [Hugo Extended](https://gohugo.io/installation/) v0.128.0 or later
- Git

### Local Development

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/jaelae-site.git
cd jaelae-site

# Run dev server
hugo server -D

# Visit http://localhost:1313
```

### Writing a New Blog Post

```bash
# Create a new post
hugo new blog/my-new-post.md

# Or just create the file manually in content/blog/
```

Post front matter:
```yaml
---
title: "Your Post Title"
date: 2026-03-10
description: "A brief description for the listing and SEO."
tags: ["VMware", "Cloud", "Automation"]
draft: false
---

Your content here in Markdown...
```

### Building for Production

```bash
hugo --minify
# Output is in /public
```

## Deploy to Cloudflare Pages

### First-time Setup

1. Push this repo to GitHub
2. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/) → Pages → Create a project
3. Connect your GitHub repo
4. Build settings:
   - **Framework preset:** Hugo
   - **Build command:** `hugo --minify`
   - **Build output directory:** `public`
   - **Environment variable:** `HUGO_VERSION` = `0.139.0`
5. Deploy

### Connect Your Domain

Since jaelae.com already uses Cloudflare DNS:

1. In Cloudflare Pages → your project → Custom domains
2. Add `jaelae.com`
3. Cloudflare will automatically update your DNS records
4. Remove the old A record pointing to InMotion Hosting (198.46.89.184)

### Auto-Deploy

Every push to the `main` branch automatically rebuilds and deploys the site.

## Project Structure

```
jaelae-site/
├── hugo.toml              # Site configuration
├── assets/css/main.css    # Stylesheet (processed by Hugo Pipes)
├── content/
│   ├── about.md           # About page
│   ├── contact.md         # Contact page
│   └── blog/              # Blog posts (Markdown)
├── layouts/
│   ├── _default/
│   │   ├── baseof.html    # Base template
│   │   ├── single.html    # Single post layout
│   │   ├── list.html      # Blog listing layout
│   │   ├── about.html     # About page layout
│   │   └── contact.html   # Contact page layout
│   ├── index.html         # Home page
│   └── partials/          # Reusable template fragments
└── static/
    ├── favicon.ico
    └── images/
        ├── headshot.jpg
        └── logos/          # Full logo system
```

## License

Content © Jason Leydon. All rights reserved.
