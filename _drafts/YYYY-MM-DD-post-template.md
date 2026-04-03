---
layout: post
title: Your Post Title Here
date: 2026-03-29T12:00:00+00:00
description: ""
tags:
syndicate_to:
  - bluesky
  - mastodon
microblog_content: ""
syndicated: false
---

Write your introductory content here. Everything above the `<!--more-->` marker
will appear in the Substack preview feed as the "teaser" that Substack subscribers
see, ending with a "Continue reading →" button. 

This section should work as a standalone hook — readers who only see the preview
on Substack should understand what the post is about and want to click through.

<!--more-->

## The Section Readers Land On

This heading is what the "Continue reading →" link anchors to. The script will
auto-detect it as the first heading after `<!--more-->` and construct the URL:

    https://blog.yoursite.com/YYYY/MM/DD/slug/#the-section-readers-land-on

Everything below <!--more--> is full content on your canonical blog only.
Substack subscribers who click through land here.

---

Continue your post normally below. You can use all standard Markdown:

**Bold**, *italic*, `inline code`, and [links](https://example.com).

- Bullet list
- Another item

```python
# Code blocks
print("Hello, world!")
```

> Blockquotes look like this.


Footnotes work like this.[^1]

[^1]: Make sure the footnote is on a new line
## Another Heading Further Down

If you wanted readers to land *here* instead of at "The Section Readers Land On"
above, you would add this to the front matter:

    substack_cutoff_heading: "Another Heading Further Down"

The <!--more--> marker can stay wherever it is in the post body — the
`substack_cutoff_heading` field overrides only the anchor target, not the
cutoff point for the preview content.

 ```html
<span class="pic-caption"> Pic captions look like this, they need html tags within the span </span> 
 ```


For link previews, use this element: 

```html

<div class="link-preview">
  <a href="https://www.theatlantic.com/science/archive/2024/01/bigheaded-ant-lion-ecosystem-cascade/677241/"><img src="https://bear-images.sfo2.cdn.digitaloceanspaces.com/bennettmcintosh-1706476670-0.jpg" alt="Thumbnail of lions"></a>
  <div class="link-preview-body">
    <a href="https://www.sciencenews.org/article/invasive-ant-lion-dinner-trees-ecosystem" class="link-preview-hed">How an invasive ant changed a lion's dinner menu</a>
    <span class="link-preview-dek">The impact rippled up from ants to trees to elephants to lions and their prey</span>
    <em class="link-preview-outlet">The Atlantic</em>
  </div>
</div>
```


---

*This is Some Preliminary Thoughts, [Bennett McIntosh’s](tab:https://bennettmcintosh.com) blog. You can sign up for updates via [email](/subscribe/) or [rss](/feed/), or unsubscribe [here](tab:https://forms.gle/1k1VB3DBuHjfpYcj7).*
