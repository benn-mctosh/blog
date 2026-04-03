# POSSE Blog Setup: GitHub Pages + Jekyll + GitHub Actions
### Publish on Own Site, Syndicate Everywhere

**What this guide produces:** A Jekyll blog at `blog.[mysite].com`, served free via GitHub Pages, that auto-posts to Bluesky and Mastodon the moment you `git push`, with a streamlined semi-manual workflow for Substack.

---

## Table of Contents

1. [Export your Bear Blog posts](#1-export-your-bear-blog-posts)
2. [Set up your local environment](#2-set-up-your-local-environment)
3. [Create the Jekyll blog repository](#3-create-the-jekyll-blog-repository)
4. [Move your domain: update the CNAME](#4-move-your-domain-update-the-cname)
5. [Import your old Bear Blog posts](#5-import-your-old-bear-blog-posts)
6. [Get your API credentials](#6-get-your-api-credentials)
7. [Set up the POSSE syndication script](#7-set-up-the-posse-syndication-script)
8. [Set up GitHub Actions for automatic syndication](#8-set-up-github-actions-for-automatic-syndication)
9. [Your publication workflow](#9-your-publication-workflow)
10. [CLI git reference](#10-cli-git-reference)
11. [Resources and further reading](#11-resources-and-further-reading)

---

## 1. Export your Bear Blog posts

Bear Blog has a built-in CSV exporter that captures all your posts and metadata.

1. Log in to Bear Blog → go to **Settings**
2. Scroll to the bottom and click **Export all blog data** — this downloads a `.csv` file with all your posts, slugs, dates, and content
3. Save this file; you'll convert it to Jekyll-compatible markdown files in Step 5

> **Tip:** The CSV export includes the raw markdown content of each post. If you have images, you'll need to re-host them somewhere (GitHub repo, Cloudinary free tier, or Imgur) since Bear Blog hosts them on its own CDN. Download any images you care about now.

---

## 2. Set up your local environment

### Install Homebrew (if not already installed)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install Ruby and Jekyll

GitHub Pages uses Ruby. The cleanest approach on macOS is to use `rbenv` to manage Ruby versions rather than the system Ruby.

```bash
brew install rbenv ruby-build
rbenv init
# Add the following line to your ~/.zshrc or ~/.bash_profile:
eval "$(rbenv init - zsh)"
# Then restart your terminal, or:
source ~/.zshrc

rbenv install 3.2.2
rbenv global 3.2.2

gem install jekyll bundler
```

Verify:
```bash
jekyll -v   # should print something like "jekyll 4.x.x"
```

### Install Python 3 and pip

You likely already have these. Check:
```bash
python3 --version
pip3 --version
```

If not: `brew install python`

### Install Git CLI

```bash
brew install git
git --version
```

Configure your identity (required for commits):
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## 3. Create the Jekyll blog repository

### On GitHub

1. Go to [github.com/new](https://github.com/new)
2. Name the repository `blog` (this will live at `github.com/[yourusername]/blog`)
3. Set it to **Public** (required for GitHub Pages on a free account)
4. Check **Add a README file**
5. Click **Create repository**

### Clone it locally

```bash
cd ~/Documents   # or wherever you keep projects
git clone https://github.com/[yourusername]/blog.git
cd blog
```

### Scaffold a new Jekyll site

```bash
jekyll new . --force
```

This creates the standard Jekyll structure:
```
blog/
├── _config.yml        # Site settings
├── _posts/            # Your blog posts go here
├── _layouts/          # (created after you add a theme)
├── index.md           # Homepage
├── about.md
├── Gemfile            # Ruby dependencies
└── .gitignore
```

### Configure `_config.yml`

Open `_config.yml` and update these fields:

```yaml
title: "Your Blog Title"
description: "A description for search engines and RSS readers"
url: "https://blog.yoursite.com"       # Your custom domain
baseurl: ""                            # Leave empty for subdomain setup

# Author info (used by some themes and RSS)
author:
  name: Your Name
  email: you@example.com

# Enables RSS feed (used by Substack import)
plugins:
  - jekyll-feed
  - jekyll-seo-tag
```

### Update the Gemfile

Open `Gemfile` and replace its contents with:

```ruby
source "https://rubygems.org"

gem "github-pages", group: :jekyll_plugins
gem "jekyll-feed"
gem "jekyll-seo-tag"
gem "webrick"   # needed for Ruby 3.x
```

Then install dependencies:
```bash
bundle install
```

### Test locally

```bash
bundle exec jekyll serve
```

Open [http://localhost:4000](http://localhost:4000) — you should see a default Jekyll site.

### Add a CNAME file

Create a file named `CNAME` (no extension) in the root of the repo:

```
blog.yoursite.com
```

This tells GitHub Pages which domain this site lives at. **Do not add `https://`** — just the bare domain.

### Push everything to GitHub

```bash
git add .
git commit -m "Initial Jekyll setup"
git push origin main
```

### Enable GitHub Pages

1. Go to your repo on GitHub → **Settings** → **Pages**
2. Under **Source**, select **Deploy from a branch**
3. Choose branch `main`, folder `/ (root)`
4. Click **Save**

GitHub will now build and deploy the site automatically on every push.

---

## 4. Move your domain: update the CNAME

You already have `blog.[mysite].com` pointing somewhere (Bear Blog). You need to redirect it to GitHub Pages instead.

### At your DNS provider (Cloudflare, Namecheap, etc.)

Find your existing `blog` CNAME record and update its **value/target** from whatever it currently is to:

```
[yourusername].github.io
```

Leave the **Name** as `blog`. The record should look like:

| Type  | Name | Value                      | TTL  |
|-------|------|----------------------------|------|
| CNAME | blog | yourusername.github.io     | Auto |

> **Note:** DNS changes can take up to 24 hours to propagate, though Cloudflare is usually near-instant. During propagation, the site may briefly be inaccessible or show the old Bear Blog.

### Back on GitHub

1. Go to your `blog` repo → **Settings** → **Pages**
2. Under **Custom domain**, type `blog.yoursite.com` and click **Save**
3. GitHub will verify DNS ownership (may take a few minutes)
4. Once verified, check **Enforce HTTPS** (may take up to an hour to become available)

> **What happened:** You didn't need to change the `[mysite].com` main site at all — only the `blog` subdomain's CNAME target changed. Your main site is untouched.

---

## 5. Import your old Bear Blog posts

### Convert the CSV to Jekyll markdown files

Bear Blog exports a CSV with columns including `title`, `slug`, `content`, `published_date`, and `tags`. Each Jekyll post needs to be a `.md` file in `_posts/` named `YYYY-MM-DD-slug.md` with YAML front matter.

Save this script as `convert_bearblog.py` in your blog directory:

```python
#!/usr/bin/env python3
"""
Convert a Bear Blog CSV export to Jekyll _posts/ markdown files.
Usage: python3 convert_bearblog.py bearblog-export.csv
"""

import csv
import sys
import os
import re
from datetime import datetime

def slugify(text):
    """Create a URL-safe slug from text."""
    text = text.lower().strip()
    text = re.sub(r'[^\w\s-]', '', text)
    text = re.sub(r'[\s_-]+', '-', text)
    return text

def convert(csv_path):
    posts_dir = "_posts"
    os.makedirs(posts_dir, exist_ok=True)

    with open(csv_path, newline='', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        for row in reader:
            # Skip pages and unpublished posts
            if row.get('is_page', '').strip().lower() == 'true':
                continue
            if row.get('publish', '').strip().lower() != 'true':
                continue

            # Parse the date
            date_str = row.get('published_date', '').strip()
            try:
                dt = datetime.fromisoformat(date_str.replace('Z', '+00:00'))
                date_formatted = dt.strftime('%Y-%m-%d')
                date_iso = dt.strftime('%Y-%m-%dT%H:%M:%S+00:00')
            except Exception:
                date_formatted = '2024-01-01'
                date_iso = '2024-01-01T00:00:00+00:00'

            title = row.get('title', 'Untitled').strip()
            slug = row.get('slug', '').strip() or slugify(title)
            tags_raw = row.get('all_tags', '').strip()
            tags = [t.strip() for t in tags_raw.split(',') if t.strip()] if tags_raw else []
            content = row.get('content', '').strip()
            description = row.get('meta_description', '').strip()

            # Build front matter
            tags_yaml = '\n'.join(f'  - {t}' for t in tags)
            tags_section = f"tags:\n{tags_yaml}" if tags else "tags: []"

            front_matter = f"""---
layout: post
title: "{title.replace('"', '\\"')}"
date: {date_iso}
{tags_section}
description: "{description.replace('"', '\\"')}"
# POSSE fields (fill these in when re-publishing or for new posts)
# syndicate_to:
#   - bluesky
#   - mastodon
# microblog_content: "Brief summary for social post. Max ~280 chars. #tag1 #tag2"
# syndicated: false
---

{content}
"""
            filename = f"{posts_dir}/{date_formatted}-{slug}.md"
            with open(filename, 'w', encoding='utf-8') as out:
                out.write(front_matter)
            print(f"  ✓ {filename}")

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python3 convert_bearblog.py bearblog-export.csv")
        sys.exit(1)
    convert(sys.argv[1])
```

Run it:
```bash
python3 convert_bearblog.py bearblog-export.csv
```

Your old posts will appear in `_posts/`. Review a few to make sure formatting looks right, then push:

```bash
git add _posts/
git commit -m "Import Bear Blog posts"
git push origin main
```

---

## 6. Get your API credentials

You need credentials for two services: Bluesky and Mastodon. Store them securely as described below — **never commit them to git**.

### Bluesky App Password

1. Go to [bsky.app](https://bsky.app) → **Settings** → **App Passwords**
2. Click **Add App Password**
3. Name it something descriptive like `jekyll-posse`
4. Copy the generated password immediately — you won't see it again
5. Note your handle (e.g., `yourname.bsky.social`)

> **Why an App Password instead of your main password?** App Passwords are scoped credentials that can be revoked individually without changing your main account password. Always use them for scripts.

### Mastodon Access Token

1. Log in to your Mastodon instance (e.g., `mastodon.social`)
2. Go to **Preferences** → **Development** → **Your Applications** → **New Application**
3. Fill in:
   - **Application name:** `jekyll-posse`
   - **Scopes:** check only `write:statuses` (that's all you need)
4. Click **Submit**
5. Click on the application you just created
6. Copy the **Access Token** value

Note your Mastodon instance URL (e.g., `https://mastodon.social`).

### Store credentials in a local `.env` file

Create a file named `.env` in the root of your blog repo:

```
BLUESKY_HANDLE=yourname.bsky.social
BLUESKY_APP_PASSWORD=xxxx-xxxx-xxxx-xxxx
MASTODON_INSTANCE=https://mastodon.social
MASTODON_ACCESS_TOKEN=your_token_here
```

**Immediately add `.env` to `.gitignore`** so it is never committed:

```bash
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Ignore .env"
git push origin main
```

---

## 7. Set up the POSSE syndication script

The script is based on Kevin Kuhl's [hugo-posse](https://github.com/kevinrkuhl/hugo-posse/) (MIT license), adapted for Jekyll's directory structure and front matter conventions.

### Install Python dependencies

Create `requirements.txt` in your blog root:

```
atproto
Mastodon.py
python-dotenv
PyYAML
requests
Markdown
```

Install:
```bash
pip3 install -r requirements.txt
```

### The script: `posse.py`

### Important!

```bash
source .venv/bin/activate # run this each session before using the script
```


Create `posse.py` in your blog root:

```python
#!/usr/bin/env python3
"""
POSSE syndication script for Jekyll + GitHub Pages.
Adapted from Kevin Kuhl's hugo-posse (MIT License):
https://github.com/kevinrkuhl/hugo-posse/
Modifications from Kevin's script are by Claude Code, so no copyright is asserted

Usage:
  source .venv/bin/activate # run this each session before using the script
  
  python3 posse.py                  # Syndicate all pending posts + rebuild Substack feed
  python3 posse.py --dry-run        # Preview without posting or writing files
  python3 posse.py --feed-only      # Rebuild substack-feed.xml without syndicating
"""

import os
import sys
import re
import glob
import yaml
import logging
import warnings
import argparse
import requests
import html
import markdown
from datetime import datetime, timezone
from email.utils import format_datetime
from pathlib import Path
from dotenv import load_dotenv

# Suppress atproto/pydantic compatibility warnings
warnings.filterwarnings("ignore", category=UserWarning)

from atproto import Client as BskyClient
from mastodon import Mastodon

# --- Configuration ---

load_dotenv()
POSTS_DIR = "_posts"

# Feed for importing substack posts with just a preview
SUBSTACK_FEED_PATH = "substack-feed.xml"   # committed to repo root; served by GH Pages

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%H:%M:%S"
)
log = logging.getLogger(__name__)


# --- Helpers ---

def load_config():
    """Read _config.yml and return the dict."""
    try:
        with open("_config.yml", "r") as f:
            return yaml.safe_load(f)
    except Exception as e:
        log.error(f"Could not read _config.yml: {e}")
        sys.exit(1)


def parse_post(filepath):
    """
    Parse a Jekyll post. Returns (front_matter_dict, content_str) or None on failure.
    Jekyll posts start with --- front matter ---.
    """
    with open(filepath, "r", encoding="utf-8") as f:
        raw = f.read()

    if not raw.startswith("---"):
        return None

    parts = raw.split("---", 2)
    if len(parts) < 3:
        return None

    try:
        fm = yaml.safe_load(parts[1])
        return fm, parts[2].strip()
    except yaml.YAMLError as e:
        log.warning(f"YAML parse error in {filepath}: {e}")
        return None


def post_url(filepath, site_url):
    """
    Reconstruct the public URL of a post from its filename.
    Jekyll convention: _posts/YYYY-MM-DD-slug.md → /YYYY/MM/DD/slug/
    """
    name = Path(filepath).stem
    match = re.match(r"(\d{4})-(\d{2})-(\d{2})-(.*)", name)
    if not match:
        return None
    year, month, day, slug = match.groups()
    return f"{site_url}/{year}/{month}/{day}/{slug}/"


def heading_to_anchor(heading_text):
    """
    Replicate kramdown's default heading ID generation:
    lowercase, strip punctuation (except hyphens), collapse spaces to hyphens.
    e.g. "The Rest of the Argument" → "the-rest-of-the-argument"
         "What's Next?" → "whats-next"
    """
    slug = heading_text.lower()
    slug = re.sub(r"[^\w\s-]", "", slug)   # strip punctuation
    slug = re.sub(r"[\s_]+", "-", slug)    # spaces/underscores → hyphens
    slug = slug.strip("-")
    return slug


def split_at_more(content, cutoff_heading=None):
    """
    Split post content at <!--more-->.

    Returns (preview_md, anchor_id) where:
      - preview_md  is the markdown above <!--more-->
      - anchor_id   is the #fragment for the "Continue reading" link

    anchor_id resolution priority:
      1. If cutoff_heading is set in front matter, slugify that heading text.
      2. Otherwise, find the first ## heading *after* <!--more--> and slugify it.
      3. If no heading follows <!--more-->, return no anchor (link goes to post root).

    Returns (None, None) if no <!--more--> marker is found.
    """
    if "<!--more-->" not in content:
        return None, None

    before, after = content.split("<!--more-->", 1)

    if cutoff_heading:
        anchor = heading_to_anchor(cutoff_heading)
        return before.strip(), anchor

    # Find first ATX heading (##, ###, etc.) in the after-section
    heading_match = re.search(r"^#{1,6}\s+(.+)$", after.strip(), re.MULTILINE)
    if heading_match:
        anchor = heading_to_anchor(heading_match.group(1).strip())
        return before.strip(), anchor

    return before.strip(), None


def markdown_to_html(md_text):
    """Convert markdown to HTML for RSS feed content."""
    return markdown.markdown(
        md_text,
        extensions=["fenced_code", "tables", "nl2br"]
    )


def build_substack_feed(all_posts, site_url, config, dry_run=False):
    """
    Write substack-feed.xml — an RSS 2.0 feed containing only the preview
    portion of each post (content above <!--more-->), with a 'Continue reading'
    link anchored to the correct section heading on the canonical post.

    Posts without <!--more--> are included in full (they opted out of truncation).
    Posts with published: false in front matter are skipped.

    The file is written to SUBSTACK_FEED_PATH in the repo root, where GitHub
    Pages will serve it at https://blog.yoursite.com/substack-feed.xml.
    Point Substack's RSS importer at that URL instead of /feed.xml.
    """
    site_title = config.get("title", "My Blog")
    site_desc  = config.get("description", "")

    items = []
    for filepath in sorted(all_posts, reverse=True):
        result = parse_post(filepath)
        if result is None:
            continue
        fm, content = result

        # Skip drafts
        if fm.get("published") is False:
            continue

        title     = fm.get("title", "Untitled")
        date_val  = fm.get("date")
        desc      = fm.get("description", "")
        url       = post_url(filepath, site_url)
        if not url:
            continue

        # Convert date to RFC 2822 for RSS
        if isinstance(date_val, datetime):
            pub_date = format_datetime(date_val)
        elif date_val:
            try:
                dt = datetime.fromisoformat(str(date_val).replace("Z", "+00:00"))
                pub_date = format_datetime(dt)
            except Exception:
                pub_date = format_datetime(datetime.now(timezone.utc))
        else:
            pub_date = format_datetime(datetime.now(timezone.utc))

        # Determine cutoff_heading override from front matter (optional)
        cutoff_heading = fm.get("substack_cutoff_heading", None)

        preview_md, anchor = split_at_more(content, cutoff_heading)

        if preview_md is not None:
            # Build the continue-reading URL
            read_more_url = f"{url}#{anchor}" if anchor else url

            preview_html = markdown_to_html(preview_md)
            read_more_html = (
                f'<p><a href="{html.escape(read_more_url)}">'
                f'Continue reading →</a></p>'
            )
            item_content = preview_html + "\n" + read_more_html
            log.info(f"  Feed: truncated at <!--more--> → #{anchor or '(root)'}")
        else:
            # No <!--more-->: include full post
            item_content = markdown_to_html(content)
            log.info(f"  Feed: no <!--more--> found, including full content")

        # Escape for XML
        def xe(s):
            return html.escape(str(s)) if s else ""

        items.append(f"""    <item>
      <title>{xe(title)}</title>
      <link>{xe(url)}</link>
      <guid isPermaLink="true">{xe(url)}</guid>
      <pubDate>{pub_date}</pubDate>
      <description>{xe(desc)}</description>
      <content:encoded><![CDATA[{item_content}]]></content:encoded>
    </item>""")

    feed_xml = f"""<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0"
     xmlns:content="http://purl.org/rss/1.0/modules/content/"
     xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{html.escape(site_title)}</title>
    <link>{html.escape(site_url)}</link>
    <description>{html.escape(site_desc)}</description>
    <atom:link href="{html.escape(site_url)}/substack-feed.xml"
               rel="self" type="application/rss+xml"/>
    <language>en-us</language>
    <lastBuildDate>{format_datetime(datetime.now(timezone.utc))}</lastBuildDate>
{chr(10).join(items)}
  </channel>
</rss>
"""
    if dry_run:
        log.info(f"[DRY RUN] Would write {SUBSTACK_FEED_PATH} ({len(items)} item(s))")
        return

    with open(SUBSTACK_FEED_PATH, "w", encoding="utf-8") as f:
        f.write(feed_xml)
    log.info(f"✓ Wrote {SUBSTACK_FEED_PATH} ({len(items)} item(s))")


def check_url_live(url, max_retries=6, delay=10):
    """Poll a URL until it returns 200 or retries are exhausted."""
    import time
    for attempt in range(max_retries):
        try:
            r = requests.get(url, timeout=5)
            if r.status_code == 200:
                return True
            log.info(f"URL not live yet ({r.status_code}). Retry {attempt+1}/{max_retries} in {delay}s...")
        except requests.RequestException:
            log.info(f"Connection error. Retry {attempt+1}/{max_retries} in {delay}s...")
        time.sleep(delay)
    return False


def mark_syndicated(filepath):
    """Write syndicated: true into the post's front matter."""
    with open(filepath, "r", encoding="utf-8") as f:
        raw = f.read()
    updated = raw.replace("syndicated: false", "syndicated: true", 1)
    if "syndicated: true" not in updated:
        updated = raw.replace("---\n", "---\nsyndicated: true\n", 1)
    with open(filepath, "w", encoding="utf-8") as f:
        f.write(updated)


# --- Syndication functions ---

def post_to_bluesky(text, dry_run=False):
    handle = os.environ.get("BLUESKY_HANDLE")
    password = os.environ.get("BLUESKY_APP_PASSWORD")
    if not handle or not password:
        log.error("Bluesky credentials missing from .env")
        return False

    if dry_run:
        log.info(f"  [DRY RUN] Would post to Bluesky:\n  {text}")
        return True

    try:
        client = BskyClient()
        client.login(handle, password)
        client.send_post(text)
        log.info("  ✓ Posted to Bluesky")
        return True
    except Exception as e:
        log.error(f"  ✗ Bluesky error: {e}")
        return False


def post_to_mastodon(text, dry_run=False):
    instance = os.environ.get("MASTODON_INSTANCE")
    token = os.environ.get("MASTODON_ACCESS_TOKEN")
    if not instance or not token:
        log.error("Mastodon credentials missing from .env")
        return False

    if dry_run:
        log.info(f"  [DRY RUN] Would post to Mastodon:\n  {text}")
        return True

    try:
        mastodon = Mastodon(access_token=token, api_base_url=instance)
        mastodon.status_post(text)
        log.info("  ✓ Posted to Mastodon")
        return True
    except Exception as e:
        log.error(f"  ✗ Mastodon error: {e}")
        return False


# --- Main ---

def main():
    parser = argparse.ArgumentParser(description="POSSE syndication for Jekyll")
    parser.add_argument("--dry-run", action="store_true",
                        help="Preview without posting or writing files")
    parser.add_argument("--no-url-check", action="store_true",
                        help="Skip checking that post URL is live")
    parser.add_argument("--feed-only", action="store_true",
                        help="Rebuild substack-feed.xml only, skip social syndication")
    args = parser.parse_args()

    config   = load_config()
    site_url = config.get("url", "").rstrip("/")
    all_posts = sorted(glob.glob(f"{POSTS_DIR}/*.md"), reverse=True)

    # Always rebuild the Substack feed on every push
    log.info("Building Substack preview feed...")
    build_substack_feed(all_posts, site_url, config, dry_run=args.dry_run)

    if args.feed_only:
        log.info("--feed-only set; skipping social syndication.")
        return

    # Collect posts pending social syndication
    pending = []
    for filepath in all_posts:
        result = parse_post(filepath)
        if result is None:
            continue
        fm, _ = result

        if fm.get("syndicated") is True:
            continue
        targets = fm.get("syndicate_to")
        if not targets:
            continue

        content = fm.get("microblog_content", "").strip()
        if not content:
            log.warning(f"  ⚠ No microblog_content in {filepath}, skipping social")
            continue

        url = post_url(filepath, site_url)
        if not url:
            log.warning(f"  ⚠ Could not determine URL for {filepath}, skipping")
            continue

        pending.append({
            "filepath": filepath,
            "title":    fm.get("title", "Untitled"),
            "targets":  targets,
            "content":  content,
            "url":      url,
        })

    if not pending:
        log.info("No posts pending social syndication.")
        return

    log.info(f"\nFound {len(pending)} post(s) for social syndication:")
    for p in pending:
        log.info(f"  • {p['title']} → {', '.join(p['targets'])}")

    if args.dry_run:
        log.info("\n--- DRY RUN (no posts will be made) ---")

    for post in pending:
        title    = post["title"]
        targets  = post["targets"]
        content  = post["content"]
        url      = post["url"]
        filepath = post["filepath"]

        log.info(f"\nSyndicating: {title}")
        log.info(f"  URL: {url}")

        if not args.dry_run and not args.no_url_check:
            log.info("  Checking that post is live...")
            if not check_url_live(url):
                log.error("  ✗ Post URL not reachable after retries. Skipping.")
                continue

        social_text = f"{content}\n\n{url}"

        if len(social_text) > 300:
            log.warning(
                f"  ⚠ Social text is {len(social_text)} chars "
                f"(Bluesky limit: 300). Consider shortening microblog_content."
            )

        success = True
        if "bluesky" in targets:
            ok = post_to_bluesky(social_text, dry_run=args.dry_run)
            success = success and ok

        if "mastodon" in targets:
            ok = post_to_mastodon(social_text, dry_run=args.dry_run)
            success = success and ok

        if success and not args.dry_run:
            mark_syndicated(filepath)
            log.info(f"  ✓ Marked as syndicated in {filepath}")

    log.info("\nDone.")


if __name__ == "__main__":
    main()
```

### Test the script

First, rebuild just the Substack feed to check that parsing works without touching any social APIs:

```bash
python3 posse.py --feed-only --dry-run
```

Then do a full dry run (feed + social preview):

```bash
python3 posse.py --dry-run
```

You should see feed-building output for all your posts, and a list of posts pending social syndication (initially none, until you add `syndicate_to` front matter to a post). Once you have a new post ready, run for real:

```bash
python3 posse.py
```

> **The URL-check feature:** By default, the script polls your post's live URL before syndicating to social platforms. This prevents posting a link that 404s because GitHub Pages hasn't finished building yet. It retries 6 times, 10 seconds apart (~1 minute total). The Substack feed is written immediately without this check, since Substack pulls it on your schedule rather than the moment you push.

> **`substack-feed.xml` is a committed file.** The script writes it locally, and the GitHub Actions workflow commits it back to the repo (alongside the `syndicated: true` flags). GitHub Pages then serves it as a static file. You never need to manage it manually.

---

## 8. Set up GitHub Actions for automatic syndication

This step makes syndication completely automatic: `git push` triggers a build, GitHub Pages builds your site, then the Actions workflow runs `posse.py`.

### Store credentials as GitHub Secrets

1. Go to your repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
2. Add these four secrets one at a time:
   - `BLUESKY_HANDLE` → your Bluesky handle
   - `BLUESKY_APP_PASSWORD` → your app password
   - `MASTODON_INSTANCE` → your instance URL
   - `MASTODON_ACCESS_TOKEN` → your access token

### Create the workflow file

Create `.github/workflows/posse.yml`:

```yaml
name: POSSE Syndication

on:
  push:
    branches: [main]

jobs:
  syndicate:
    runs-on: ubuntu-latest
    # Only run if markdown files in _posts changed
    if: contains(github.event.commits[0].message, '_posts') || true

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Wait for GitHub Pages to build
        run: |
          echo "Waiting 90 seconds for GitHub Pages build to complete..."
          sleep 90

      - name: Run POSSE syndication
        env:
          BLUESKY_HANDLE: ${{ secrets.BLUESKY_HANDLE }}
          BLUESKY_APP_PASSWORD: ${{ secrets.BLUESKY_APP_PASSWORD }}
          MASTODON_INSTANCE: ${{ secrets.MASTODON_INSTANCE }}
          MASTODON_ACCESS_TOKEN: ${{ secrets.MASTODON_ACCESS_TOKEN }}
        run: python3 posse.py

      - name: Commit syndicated flags and Substack feed back to repo
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add _posts/ substack-feed.xml
          git diff --cached --quiet || git commit -m "chore: syndicated flags + substack feed [skip ci]"
          git push
```

> **The `[skip ci]` tag** in the commit message prevents the workflow from running again when the bot pushes the `syndicated: true` update back to the repo — otherwise you'd get an infinite loop.

Push the workflow file:

```bash
git add .github/
git commit -m "Add POSSE GitHub Actions workflow"
git push origin main
```

Watch it run: go to your repo → **Actions** tab. You'll see the workflow trigger on every push.

---

## 9. Your publication workflow

Here's the complete end-to-end process for publishing a new post:

### Step 1: Write your post

Create a new file in `_posts/` following the naming convention:

```
_posts/YYYY-MM-DD-your-post-slug.md
```

Use the template file (provided separately) to start with the right front matter.

Key things to write *while you're still in writing mode*:
- **`microblog_content`**: Your ~250-character summary for Bluesky and Mastodon. Write this now — it should feel like a "here's the argument" tweet, not a marketing blurb. Add 1–3 hashtags if appropriate.
- **`description`**: 1–2 sentences for SEO/RSS preview. Substack will use this as the preview text.

### Step 2: Commit and push

```bash
git add _posts/YYYY-MM-DD-your-post-slug.md
git commit -m "Publish: Your Post Title"
git push origin main
```

**What happens automatically in the next ~2 minutes:**
- GitHub Pages builds and deploys your site
- The GitHub Actions workflow waits 90 seconds, then runs `posse.py`
- `posse.py` checks that your post URL is live, then posts to Bluesky and Mastodon
- The bot commits `syndicated: true` back to the post file

### Step 3: Import to Substack (~2 minutes)

The script generates `substack-feed.xml` — a truncated feed containing only the content above each post's `<!--more-->` marker, with a "Continue reading →" link anchored to the right section heading. Point Substack at this URL instead of your main feed.

**First-time setup (once only):**
1. Go to your Substack dashboard → **Posts** → **Import**
2. Click **Import from RSS feed**
3. Paste: `https://blog.yoursite.com/substack-feed.xml`
4. Substack remembers this URL — future imports are one click

**Per-post workflow:**
1. Select the new post from the import list
2. Click **Import** — it arrives as a draft showing only your preview content and a "Continue reading →" button linking to the anchored section on your canonical post
3. Add a header image if you want, check the preview text (pulled from `description`), then send

> **Posts without `<!--more-->`** are included in the Substack feed in full — the script treats the marker as opt-in. If you want a post to appear in Substack without truncation, simply omit the marker.

> **The anchor link** in "Continue reading →" points to `https://blog.yoursite.com/YYYY/MM/DD/slug/#the-heading-slug`. If you used `substack_cutoff_heading` in the front matter, the script uses that heading's slug; otherwise it auto-detects the first heading after `<!--more-->`.

### Step 4: Substack Notes (~30 seconds)

Paste your `microblog_content` (you already wrote it) into a Substack Note, add the link to your post, and publish. Done.

### Step 5: LinkedIn (optional, manual)

LinkedIn is best handled manually due to its restrictive API. A good LinkedIn post about a long-form piece is different from a skeet anyway — consider writing a brief context paragraph and pasting the link. This doesn't need to be automated.

---

### Publication checklist

```
[ ] Post written and proofread
[ ] microblog_content filled in (≤ ~250 chars leaving room for URL)
[ ] description filled in (1-2 sentences)
[ ] syndicate_to: includes bluesky and/or mastodon
[ ] git add, commit, push
[ ] Check Actions tab to confirm workflow ran successfully
[ ] Substack: import from RSS, review draft, send
[ ] Substack Notes: paste microblog_content + link
[ ] LinkedIn: (optional) write context paragraph + link
```

---

## 10. CLI git reference

You've been using GitHub Desktop — here's the equivalent CLI git workflow. The commands you'll use 95% of the time are very few.

### The core workflow

```bash
# 1. See what's changed
git status

# 2. Stage your changes (the "." stages everything; or name specific files)
git add .
git add _posts/2025-03-29-my-post.md   # stage a specific file

# 3. Commit with a message
git commit -m "Publish: My Post Title"

# 4. Push to GitHub
git push origin main
```

### Other commands you'll occasionally need

```bash
# Pull latest changes from GitHub (do this if you edit on another machine)
git pull origin main

# See your recent commits
git log --oneline -10

# Undo staged changes (un-stage a file without deleting it)
git restore --staged _posts/draft.md

# Discard uncommitted changes to a file (careful — this is permanent)
git restore _posts/draft.md

# See what changed in a file
git diff _posts/2025-03-29-my-post.md

# See the full history of a specific file
git log --follow _posts/2025-03-29-my-post.md
```

### Branching (useful when drafting)

```bash
# Create a branch for a draft
git checkout -b draft/my-new-post

# Push the branch (without triggering syndication, since main won't change)
git push origin draft/my-new-post

# When ready to publish, merge to main
git checkout main
git merge draft/my-new-post
git push origin main
```

> **Pro tip for drafts:** The syndication script only runs when you push to `main`. Working in a branch lets you back up your draft to GitHub without triggering a publish.

### Learning resources

- **[The Official Git Book (Pro Git)](https://git-scm.com/book/en/v2)** — free online, chapters 1–3 cover everything you need
- **[Oh My Git!](https://ohmygit.org/)** — a game-based visual learner for git concepts
- **[GitHub's Git Cheat Sheet](https://education.github.com/git-cheat-sheet-education.pdf)** — printable one-pager
- **[Dangit, Git!](https://dangitgit.com/)** — plain-language fixes for common mistakes ("I messed up and need to undo something")

---

## 11. Resources and further reading

### Jekyll

- **[Jekyll docs](https://jekyllrb.com/docs/)** — official documentation, especially [Posts](https://jekyllrb.com/docs/posts/) and [Front Matter](https://jekyllrb.com/docs/front-matter/)
- **[GitHub Pages dependency versions](https://pages.github.com/versions/)** — shows exactly which Jekyll/plugin versions GitHub Pages supports
- **[Minimal Mistakes theme](https://mmistakes.github.io/minimal-mistakes/)** — a popular, well-documented Jekyll theme with good typography if you want to upgrade from the default

### POSSE & the script

- **[Kevin Kuhl's hugo-posse (MIT License)](https://github.com/kevinrkuhl/hugo-posse/)** — the upstream script this is adapted from; see also his [Part 1 writeup](https://www.kevinrkuhl.com/blog/2025/12/posse-for-hugo-pt1/) and [Part 2 writeup](https://www.kevinrkuhl.com/blog/2025/12/posse-for-hugo-pt2/)
- **[Josh Beckman's Jekyll + Bluesky crosspost (Ruby)](https://www.joshbeckman.org/blog/crossposting-to-bluesky-from-jekyll)** — an alternative Ruby approach if you prefer staying in the Jekyll/Ruby ecosystem
- **[IndieWeb POSSE wiki](https://indieweb.org/POSSE)** — the conceptual background and a huge list of implementations

### APIs

- **[Bluesky API: Creating a post](https://docs.bsky.app/blog/create-post)** — official guide with Python examples
- **[atproto Python SDK](https://atproto.blue/en/latest/)** — documentation for the `atproto` library
- **[Mastodon.py documentation](https://mastodonpy.readthedocs.io/)** — documentation for the `Mastodon.py` library

### GitHub Actions

- **[GitHub Actions quickstart](https://docs.github.com/en/actions/quickstart)** — 10-minute intro
- **[GitHub Secrets documentation](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)** — how to store and use credentials securely in workflows

---

*Last updated: March 2026*
